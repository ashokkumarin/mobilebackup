# AWS Mobile Backup - Complete Implementation Plan

## 🎯 Phase 1: Foundation Setup (Week 1-2)

### 1.1 AWS Account & Initial Setup
```bash
# Install AWS CLI (if not already done)
aws --version

# Create S3 bucket
aws s3 mb s3://your-mobile-backup-bucket --region us-east-1

# Enable versioning (optional but recommended)
aws s3api put-bucket-versioning \
  --bucket your-mobile-backup-bucket \
  --versioning-configuration Status=Enabled
```

### 1.2 Create DynamoDB Table
```bash
# Create metadata table
aws dynamodb create-table \
  --table-name MobileBackupMetadata \
  --attribute-definitions \
    AttributeName=userId,AttributeType=S \
    AttributeName=fileId,AttributeType=S \
  --key-schema \
    AttributeName=userId,KeyType=HASH \
    AttributeName=fileId,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

### 1.3 Create SQS Queue
```bash
# Create queue for download notifications
aws sqs create-queue \
  --queue-name mobile-backup-download-queue \
  --attributes VisibilityTimeoutSeconds=60,MessageRetentionPeriod=1209600
```

### 1.4 IAM Roles Setup
```json
// Lambda execution role policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::your-mobile-backup-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:*:table/MobileBackupMetadata"
    },
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage"
      ],
      "Resource": "arn:aws:sqs:us-east-1:*:mobile-backup-download-queue"
    }
  ]
}
```

## 🔧 Phase 2: Backend Services (Week 3-4)

### 2.1 Lambda Function: Generate Presigned URLs
```javascript
// generatePresignedUrl.js
const AWS = require('aws-sdk');
const s3 = new AWS.S3();
const dynamodb = new AWS.DynamoDB.DocumentClient();

exports.handler = async (event) => {
    try {
        const { fileName, fileType, userId } = JSON.parse(event.body);
        const fileId = `${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
        const key = `${userId}/${fileId}-${fileName}`;
        
        // Generate presigned URL
        const presignedUrl = s3.getSignedUrl('putObject', {
            Bucket: process.env.BUCKET_NAME,
            Key: key,
            ContentType: fileType,
            Expires: 3600, // 1 hour
        });
        
        // Store metadata in DynamoDB
        await dynamodb.put({
            TableName: process.env.DYNAMODB_TABLE,
            Item: {
                userId: userId,
                fileId: fileId,
                fileName: fileName,
                fileType: fileType,
                s3Key: key,
                status: 'pending',
                createdAt: new Date().toISOString(),
                downloadedAt: null
            }
        }).promise();
        
        return {
            statusCode: 200,
            headers: {
                'Access-Control-Allow-Origin': '*',
                'Access-Control-Allow-Headers': 'Content-Type,Authorization'
            },
            body: JSON.stringify({
                uploadUrl: presignedUrl,
                fileId: fileId,
                key: key
            })
        };
    } catch (error) {
        return {
            statusCode: 500,
            body: JSON.stringify({ error: error.message })
        };
    }
};
```

### 2.2 Lambda Function: S3 Event Handler
```javascript
// s3EventHandler.js
const AWS = require('aws-sdk');
const dynamodb = new AWS.DynamoDB.DocumentClient();
const sqs = new AWS.SQS();

exports.handler = async (event) => {
    for (const record of event.Records) {
        if (record.eventName.startsWith('ObjectCreated')) {
            const bucket = record.s3.bucket.name;
            const key = record.s3.object.key;
            const size = record.s3.object.size;
            
            // Extract userId and fileId from key
            const [userId, fileInfo] = key.split('/');
            const fileId = fileInfo.split('-')[0];
            
            // Update DynamoDB
            await dynamodb.update({
                TableName: process.env.DYNAMODB_TABLE,
                Key: {
                    userId: userId,
                    fileId: fileId
                },
                UpdateExpression: 'SET #status = :status, uploadedAt = :uploadedAt, fileSize = :fileSize',
                ExpressionAttributeNames: {
                    '#status': 'status'
                },
                ExpressionAttributeValues: {
                    ':status': 'uploaded',
                    ':uploadedAt': new Date().toISOString(),
                    ':fileSize': size
                }
            }).promise();
            
            // Send message to SQS
            await sqs.sendMessage({
                QueueUrl: process.env.SQS_QUEUE_URL,
                MessageBody: JSON.stringify({
                    userId: userId,
                    fileId: fileId,
                    s3Key: key,
                    bucket: bucket,
                    size: size,
                    timestamp: new Date().toISOString()
                })
            }).promise();
        }
    }
    
    return { statusCode: 200 };
};
```

### 2.3 API Gateway Setup
```yaml
# serverless.yml (using Serverless Framework)
service: mobile-backup-api

provider:
  name: aws
  runtime: nodejs18.x
  region: us-east-1
  environment:
    BUCKET_NAME: your-mobile-backup-bucket
    DYNAMODB_TABLE: MobileBackupMetadata
    SQS_QUEUE_URL: !Ref DownloadQueue

functions:
  generatePresignedUrl:
    handler: generatePresignedUrl.handler
    events:
      - http:
          path: /upload-url
          method: post
          cors: true
  
  s3EventHandler:
    handler: s3EventHandler.handler
    events:
      - s3:
          bucket: your-mobile-backup-bucket
          event: s3:ObjectCreated:*

resources:
  Resources:
    DownloadQueue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: mobile-backup-download-queue
        VisibilityTimeoutSeconds: 60
```

## 📱 Phase 3: Mobile App (Week 5-6)

### 3.1 React Native Setup
```bash
# Initialize React Native project
npx react-native init MobileBackupApp
cd MobileBackupApp

# Install required packages
npm install @react-native-async-storage/async-storage
npm install react-native-image-picker
npm install react-native-fs
npm install axios
```

### 3.2 Core Mobile App Components
```javascript
// UploadService.js
import axios from 'axios';
import RNFS from 'react-native-fs';

class UploadService {
    constructor() {
        this.apiBaseUrl = 'https://your-api-gateway-url.amazonaws.com/dev';
    }
    
    async uploadFile(filePath, fileName, fileType, userId) {
        try {
            // Get presigned URL
            const response = await axios.post(`${this.apiBaseUrl}/upload-url`, {
                fileName: fileName,
                fileType: fileType,
                userId: userId
            });
            
            const { uploadUrl, fileId, key } = response.data;
            
            // Read file as base64
            const fileData = await RNFS.readFile(filePath, 'base64');
            const buffer = Buffer.from(fileData, 'base64');
            
            // Upload to S3
            await axios.put(uploadUrl, buffer, {
                headers: {
                    'Content-Type': fileType
                },
                onUploadProgress: (progressEvent) => {
                    const progress = (progressEvent.loaded / progressEvent.total) * 100;
                    console.log(`Upload Progress: ${progress}%`);
                }
            });
            
            return { success: true, fileId, key };
        } catch (error) {
            console.error('Upload failed:', error);
            return { success: false, error: error.message };
        }
    }
}

export default new UploadService();
```

### 3.3 Main App Component
```javascript
// App.js
import React, { useState } from 'react';
import { View, Text, TouchableOpacity, FlatList, Alert } from 'react-native';
import { launchImageLibrary } from 'react-native-image-picker';
import UploadService from './UploadService';

const App = () => {
    const [uploads, setUploads] = useState([]);
    const userId = 'user123'; // In real app, get from auth

    const selectAndUploadMedia = () => {
        const options = {
            mediaType: 'mixed',
            quality: 0.8,
            multiple: true
        };

        launchImageLibrary(options, async (response) => {
            if (response.assets) {
                for (const asset of response.assets) {
                    const uploadData = {
                        id: Date.now() + Math.random(),
                        fileName: asset.fileName,
                        status: 'uploading',
                        progress: 0
                    };
                    
                    setUploads(prev => [...prev, uploadData]);
                    
                    const result = await UploadService.uploadFile(
                        asset.uri,
                        asset.fileName,
                        asset.type,
                        userId
                    );
                    
                    setUploads(prev => prev.map(item => 
                        item.id === uploadData.id 
                            ? { ...item, status: result.success ? 'completed' : 'failed' }
                            : item
                    ));
                }
            }
        });
    };

    return (
        <View style={{ flex: 1, padding: 20 }}>
            <TouchableOpacity
                style={{ backgroundColor: '#007AFF', padding: 15, borderRadius: 8 }}
                onPress={selectAndUploadMedia}
            >
                <Text style={{ color: 'white', textAlign: 'center' }}>
                    Select & Upload Media
                </Text>
            </TouchableOpacity>
            
            <FlatList
                data={uploads}
                keyExtractor={item => item.id.toString()}
                renderItem={({ item }) => (
                    <View style={{ padding: 10, borderBottomWidth: 1 }}>
                        <Text>{item.fileName}</Text>
                        <Text>Status: {item.status}</Text>
                    </View>
                )}
            />
        </View>
    );
};

export default App;
```

## 💻 Phase 4: Windows Service (Week 7-8)

### 4.1 Node.js Windows Service
```javascript
// windowsService.js
const AWS = require('aws-sdk');
const fs = require('fs');
const path = require('path');

class BackupDownloadService {
    constructor() {
        this.sqs = new AWS.SQS({ region: 'us-east-1' });
        this.s3 = new AWS.S3({ region: 'us-east-1' });
        this.dynamodb = new AWS.DynamoDB.DocumentClient({ region: 'us-east-1' });
        
        this.queueUrl = 'https://sqs.us-east-1.amazonaws.com/YOUR-ACCOUNT/mobile-backup-download-queue';
        this.downloadPath = 'C:\\MobileBackup';
        this.isRunning = false;
        
        // Ensure download directory exists
        if (!fs.existsSync(this.downloadPath)) {
            fs.mkdirSync(this.downloadPath, { recursive: true });
        }
    }
    
    async start() {
        this.isRunning = true;
        console.log('Backup Download Service started');
        this.pollQueue();
    }
    
    async stop() {
        this.isRunning = false;
        console.log('Backup Download Service stopped');
    }
    
    async pollQueue() {
        while (this.isRunning) {
            try {
                const messages = await this.sqs.receiveMessage({
                    QueueUrl: this.queueUrl,
                    MaxNumberOfMessages: 10,
                    WaitTimeSeconds: 20
                }).promise();
                
                if (messages.Messages) {
                    for (const message of messages.Messages) {
                        await this.processMessage(message);
                    }
                }
            } catch (error) {
                console.error('Error polling queue:', error);
                await this.sleep(5000); // Wait 5 seconds before retry
            }
        }
    }
    
    async processMessage(message) {
        try {
            const data = JSON.parse(message.Body);
            const { userId, fileId, s3Key, bucket } = data;
            
            console.log(`Processing download for file: ${s3Key}`);
            
            // Create user directory
            const userDir = path.join(this.downloadPath, userId);
            if (!fs.existsSync(userDir)) {
                fs.mkdirSync(userDir, { recursive: true });
            }
            
            // Download from S3
            const fileName = path.basename(s3Key);
            const localFilePath = path.join(userDir, fileName);
            
            const s3Object = await this.s3.getObject({
                Bucket: bucket,
                Key: s3Key
            }).promise();
            
            fs.writeFileSync(localFilePath, s3Object.Body);
            
            // Update DynamoDB
            await this.dynamodb.update({
                TableName: 'MobileBackupMetadata',
                Key: {
                    userId: userId,
                    fileId: fileId
                },
                UpdateExpression: 'SET #status = :status, downloadedAt = :downloadedAt, localPath = :localPath',
                ExpressionAttributeNames: {
                    '#status': 'status'
                },
                ExpressionAttributeValues: {
                    ':status': 'downloaded',
                    ':downloadedAt': new Date().toISOString(),
                    ':localPath': localFilePath
                }
            }).promise();
            
            // Delete from S3 (optional - or use lifecycle policy)
            // await this.s3.deleteObject({
            //     Bucket: bucket,
            //     Key: s3Key
            // }).promise();
            
            // Delete message from queue
            await this.sqs.deleteMessage({
                QueueUrl: this.queueUrl,
                ReceiptHandle: message.ReceiptHandle
            }).promise();
            
            console.log(`Successfully downloaded: ${fileName}`);
            
        } catch (error) {
            console.error('Error processing message:', error);
        }
    }
    
    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}

// Start the service
const service = new BackupDownloadService();
service.start();

// Handle graceful shutdown
process.on('SIGINT', () => {
    console.log('Received SIGINT, shutting down gracefully');
    service.stop();
    process.exit(0);
});
```

### 4.2 Windows Service Installation
```bash
# Install as Windows Service using node-windows
npm install -g node-windows

# Create service installer
node installService.js
```

```javascript
// installService.js
const Service = require('node-windows').Service;

const svc = new Service({
    name: 'Mobile Backup Download Service',
    description: 'Downloads mobile backup files from AWS S3',
    script: 'C:\\path\\to\\your\\windowsService.js'
});

svc.on('install', function() {
    console.log('Service installed successfully');
    svc.start();
});

svc.install();
```

## 🛡️ Phase 5: Security & Authentication (Week 9)

### 5.1 AWS Cognito Setup
```bash
# Create Cognito User Pool
aws cognito-idp create-user-pool \
  --pool-name MobileBackupUsers \
  --policies '{"PasswordPolicy":{"MinimumLength":8,"RequireUppercase":true,"RequireLowercase":true,"RequireNumbers":true}}'

# Create Cognito User Pool Client
aws cognito-idp create-user-pool-client \
  --user-pool-id us-east-1_XXXXXXXXX \
  --client-name MobileBackupApp \
  --no-generate-secret
```

### 5.2 Update Lambda with Authentication
```javascript
// Add to generatePresignedUrl.js
const jwt = require('jsonwebtoken');
const jwksClient = require('jwks-rsa');

const client = jwksClient({
    jwksUri: 'https://cognito-idp.us-east-1.amazonaws.com/us-east-1_XXXXXXXXX/.well-known/jwks.json'
});

const verifyToken = async (token) => {
    const decodedToken = jwt.decode(token, { complete: true });
    const kid = decodedToken.header.kid;
    
    const key = await client.getSigningKey(kid);
    const signingKey = key.getPublicKey();
    
    return jwt.verify(token, signingKey);
};

// Add authentication check at the beginning of handler
exports.handler = async (event) => {
    try {
        const authHeader = event.headers.Authorization || event.headers.authorization;
        if (!authHeader) {
            return {
                statusCode: 401,
                body: JSON.stringify({ error: 'No authorization header' })
            };
        }
        
        const token = authHeader.replace('Bearer ', '');
        const decoded = await verifyToken(token);
        const userId = decoded.sub;
        
        // Rest of your existing code...
    } catch (error) {
        return {
            statusCode: 401,
            body: JSON.stringify({ error: 'Invalid token' })
        };
    }
};
```

## 📊 Phase 6: Monitoring & Optimization (Week 10)

### 6.1 CloudWatch Dashboard
```json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/S3", "BucketSizeBytes", "BucketName", "your-mobile-backup-bucket"],
          ["AWS/Lambda", "Invocations", "FunctionName", "generatePresignedUrl"],
          ["AWS/SQS", "NumberOfMessagesSent", "QueueName", "mobile-backup-download-queue"]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Mobile Backup Metrics"
      }
    }
  ]
}
```

### 6.2 S3 Lifecycle Policy
```json
{
  "Rules": [
    {
      "ID": "DeleteAfterDownload",
      "Status": "Enabled",
      "Expiration": {
        "Days": 7
      },
      "Filter": {
        "Prefix": ""
      }
    }
  ]
}
```

## 🚀 Deployment Checklist

### Phase 1 Deployment:
- [ ] Create AWS account and configure CLI
- [ ] Create S3 bucket with proper CORS settings
- [ ] Create DynamoDB table
- [ ] Create SQS queue
- [ ] Set up IAM roles and policies

### Phase 2 Deployment:
- [ ] Deploy Lambda functions
- [ ] Configure API Gateway
- [ ] Set up S3 event notifications
- [ ] Test API endpoints with Postman

### Phase 3 Deployment:
- [ ] Build and test mobile app locally
- [ ] Configure API endpoints
- [ ] Test file upload functionality
- [ ] Deploy to physical device for testing

### Phase 4 Deployment:
- [ ] Set up Windows service
- [ ] Configure AWS credentials on local machine
- [ ] Test SQS polling and S3 download
- [ ] Install as Windows service

### Phase 5 Deployment:
- [ ] Set up Cognito User Pool
- [ ] Update Lambda functions with authentication
- [ ] Update mobile app with login functionality
- [ ] Test end-to-end authentication

### Phase 6 Deployment:
- [ ] Create CloudWatch dashboard
- [ ] Set up S3 lifecycle policies
- [ ] Configure CloudWatch alarms
- [ ] Set up cost monitoring

## 💡 Testing Strategy

1. **Unit Testing**: Test each Lambda function individually
2. **Integration Testing**: Test API Gateway + Lambda combinations
3. **End-to-End Testing**: Full flow from mobile app to local download
4. **Load Testing**: Test with multiple file uploads simultaneously
5. **Error Testing**: Test failure scenarios (network issues, invalid files, etc.)

## 📈 Monitoring & Maintenance

- Set up CloudWatch alarms for failed Lambda executions
- Monitor S3 storage costs
- Regular cleanup of old DynamoDB records
- Monitor SQS queue depth
- Set up SNS notifications for critical errors

This implementation plan gives you a production-ready system while learning key AWS services!
