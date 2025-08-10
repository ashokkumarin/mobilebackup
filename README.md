# 📱 Mobile Backup System

> Create private cloud, which will be able to take backup of images, videos and documents from mobile and store it in personal computer
> Automatically backup your mobile photos and videos to your local computer using AWS cloud services

[![AWS](https://img.shields.io/badge/AWS-Cloud-orange?style=flat&logo=amazon-aws)](https://aws.amazon.com)
[![React Native](https://img.shields.io/badge/React%20Native-Mobile-blue?style=flat&logo=react)](https://reactnative.dev)
[![Node.js](https://img.shields.io/badge/Node.js-Backend-green?style=flat&logo=node.js)](https://nodejs.org)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

## 🎯 Problem & Solution

**Problem**: Manually backing up photos and videos from mobile devices to computers is time-consuming and often forgotten.

**Solution**: An automated cloud-based backup system that uploads media from your mobile device to AWS, then automatically downloads to your local computer, providing a seamless backup experience.

## 🏗️ Architecture Overview

```
Mobile App → AWS S3 → Event Trigger → SQS Queue → Local Windows Service
```

The system uses a serverless, event-driven architecture for efficient and cost-effective media backup:

- **Mobile App**: React Native app for selecting and uploading media
- **AWS Backend**: Lambda functions, S3 storage, DynamoDB metadata, SQS messaging
- **Local Service**: Windows service that automatically downloads files to your computer

## 🚀 Features

- ✅ **Automatic Backup**: Set it and forget it - files backup automatically
- ✅ **Cost Optimized**: Only pay for what you use (~$3-12/month for personal use)
- ✅ **Secure**: AWS Cognito authentication with encrypted storage
- ✅ **Resumable Uploads**: Large videos can resume if interrupted
- ✅ **Progress Tracking**: Real-time upload progress in mobile app
- ✅ **Auto Cleanup**: Files automatically deleted from cloud after local download
- ✅ **Organized Storage**: Files organized by user and date on local system
- ✅ **Monitoring**: CloudWatch dashboards for system health

## 📂 Repository Structure

```
mobilebackup/
├── docs/                          # Documentation
│   ├── architecture.md            # Detailed architecture documentation
│   └── implementation-plan.md     # Step-by-step implementation guide
├── mobile-app/                    # React Native mobile application
│   ├── src/
│   │   ├── components/
│   │   ├── services/
│   │   └── screens/
│   ├── android/
│   ├── ios/
│   └── package.json
├── aws-backend/                   # AWS Lambda functions and infrastructure
│   ├── functions/
│   │   ├── generatePresignedUrl.js
│   │   ├── s3EventHandler.js
│   │   └── authHandler.js
│   ├── infrastructure/
│   │   ├── serverless.yml
│   │   ├── cloudformation.yml
│   │   └── cognito-setup.json
│   └── package.json
├── windows-service/               # Local download service
│   ├── src/
│   │   ├── downloadService.js
│   │   ├── config.js
│   │   └── utils.js
│   ├── install/
│   │   └── installService.js
│   └── package.json
├── scripts/                       # Setup and deployment scripts
│   ├── setup-aws-resources.sh
│   ├── deploy-backend.sh
│   └── build-mobile.sh
├── .env.example                   # Environment variables template
├── .gitignore
└── README.md
```

## 🛠️ Technology Stack

### Mobile App
- **Framework**: React Native
- **State Management**: React Hooks
- **File Operations**: react-native-fs
- **Image Picker**: react-native-image-picker
- **HTTP Client**: Axios

### AWS Backend
- **Compute**: AWS Lambda (Node.js 18.x)
- **Storage**: Amazon S3
- **Database**: Amazon DynamoDB
- **Messaging**: Amazon SQS
- **API**: Amazon API Gateway
- **Authentication**: Amazon Cognito
- **Monitoring**: Amazon CloudWatch

### Local Service
- **Runtime**: Node.js
- **Service Management**: node-windows
- **AWS SDK**: aws-sdk-js

## ⚡ Quick Start

### Prerequisites
- AWS Account with CLI configured
- Node.js 18+ installed
- React Native development environment
- Windows machine for local service

### 1. Clone Repository
```bash
git clone https://github.com/yourusername/mobilebackup.git
cd mobilebackup
```

### 2. Set Up AWS Resources
```bash
# Copy environment template
cp .env.example .env

# Edit .env with your AWS details
# Run setup script
chmod +x scripts/setup-aws-resources.sh
./scripts/setup-aws-resources.sh
```

### 3. Deploy Backend
```bash
cd aws-backend
npm install
npm run deploy
```

### 4. Configure Mobile App
```bash
cd mobile-app
npm install

# Update config with your API Gateway URL
# Build and run
npx react-native run-android
# or
npx react-native run-ios
```

### 5. Install Windows Service
```bash
cd windows-service
npm install

# Update config with your AWS credentials
# Install as Windows service
node install/installService.js
```

## 📋 Detailed Setup Guide

For complete step-by-step setup instructions, see:
- 📖 [Implementation Plan](docs/implementation-plan.md) - Detailed implementation phases
- 🏗️ [Architecture Guide](docs/architecture.md) - System architecture overview

## 🔧 Configuration

### Environment Variables
Create a `.env` file in the root directory:

```env
# AWS Configuration
AWS_REGION=us-east-1
AWS_ACCOUNT_ID=123456789012

# S3 Configuration
S3_BUCKET_NAME=your-mobile-backup-bucket

# DynamoDB Configuration
DYNAMODB_TABLE_NAME=MobileBackupMetadata

# SQS Configuration
SQS_QUEUE_URL=https://sqs.us-east-1.amazonaws.com/123456789012/mobile-backup-download-queue

# Cognito Configuration
COGNITO_USER_POOL_ID=us-east-1_XXXXXXXXX
COGNITO_CLIENT_ID=your-client-id

# API Gateway Configuration
API_GATEWAY_URL=https://your-api-id.execute-api.us-east-1.amazonaws.com/dev

# Local Configuration
LOCAL_DOWNLOAD_PATH=C:\MobileBackup
```

### Mobile App Configuration
Update `mobile-app/src/config/app-config.js`:

```javascript
export const APP_CONFIG = {
  API_BASE_URL: 'https://your-api-gateway-url.amazonaws.com/dev',
  COGNITO_USER_POOL_ID: 'us-east-1_XXXXXXXXX',
  COGNITO_CLIENT_ID: 'your-cognito-client-id',
  AWS_REGION: 'us-east-1'
};
```

## 💰 Cost Estimation

### Monthly Costs (Personal Use - ~1GB uploads)
| Service | Estimated Cost |
|---------|----------------|
| S3 Storage | $1-3 |
| Lambda Executions | $0.20-0.50 |
| DynamoDB | $0.25-1.00 |
| SQS Messages | $0.40-0.80 |
| API Gateway | $1-2 |
| **Total** | **$3-7/month** |

### Cost Optimization Tips
- Enable S3 lifecycle policies for automatic cleanup
- Use S3 Intelligent Tiering for long-term storage
- Monitor usage with AWS Cost Explorer
- Set up billing alerts

## 🔒 Security

### Authentication
- AWS Cognito for user management
- JWT tokens for API authentication
- Secure credential storage

### Data Protection
- Encryption in transit (HTTPS/TLS)
- Encryption at rest (S3, DynamoDB)
- Presigned URLs with expiration

### Access Control
- IAM roles with least privilege
- Resource-based policies
- API Gateway authorization

## 📊 Monitoring

### CloudWatch Metrics
- Upload success rates
- Processing latency
- Error rates
- Storage usage
- Cost metrics

### Alarms & Notifications
- Failed uploads/downloads
- High error rates
- Cost threshold alerts
- Queue depth monitoring

## 🧪 Testing

### Unit Tests
```bash
# Backend tests
cd aws-backend && npm test

# Mobile app tests
cd mobile-app && npm test

# Windows service tests
cd windows-service && npm test
```

### Integration Tests
```bash
# End-to-end testing
npm run test:e2e
```

### Load Testing
```bash
# Test with multiple concurrent uploads
npm run test:load
```

## 📱 Usage

1. **Install mobile app** on your device
2. **Sign up/Login** using the app
3. **Select photos/videos** to backup
4. **Upload automatically** happens in background
5. **Files download** automatically to your computer
6. **Monitor progress** through mobile app and CloudWatch

## 🤝 Contributing

We welcome contributions! Please see our [Contributing Guidelines](CONTRIBUTING.md).

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📝 Development Roadmap

### Phase 1 (Current) - Core Functionality
- [x] Basic mobile app with upload
- [x] AWS backend services
- [x] Windows download service
- [x] Basic authentication

### Phase 2 - Enhanced Features
- [ ] iOS app support
- [ ] Selective sync (choose folders)
- [ ] Duplicate detection
- [ ] Compression options

### Phase 3 - Advanced Features
- [ ] Multi-user support
- [ ] Web dashboard
- [ ] Cross-platform desktop apps (macOS, Linux)
- [ ] Advanced sharing features

### Phase 4 - Enterprise Features
- [ ] Team/family sharing
- [ ] Advanced analytics
- [ ] Custom retention policies
- [ ] API for third-party integrations

## 🐛 Known Issues

- Large video uploads (>100MB) may timeout on slow connections
- Windows service requires manual restart after system reboot
- iOS background upload limitations (iOS restrictions)

See [Issues](https://github.com/yourusername/mobilebackup/issues) for full list.

## 📞 Support

- 📖 **Documentation**: Check `/docs` folder
- 🐛 **Bug Reports**: [GitHub Issues](https://github.com/yourusername/mobilebackup/issues)
- 💬 **Discussions**: [GitHub Discussions](https://github.com/yourusername/mobilebackup/discussions)
- 📧 **Email**: support@mobilebackup.dev

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- AWS documentation and examples
- React Native community
- Node.js ecosystem contributors
- Open source libraries used in this project

## 🔗 Related Projects

- [AWS Mobile SDK](https://aws.amazon.com/mobile/sdk/) - Official AWS mobile SDKs
- [Serverless Framework](https://serverless.com/) - Serverless application framework
- [React Native](https://reactnative.dev/) - Mobile app development framework

---

⭐ **Star this repository** if you find it helpful!

📝 **Questions?** Open an issue or start a discussion!

🚀 **Ready to backup?** Follow the [Quick Start Guide](#-quick-start) above!

---
<sub>Built with ❤️ for the AWS learning community</sub>