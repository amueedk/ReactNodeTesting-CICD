# ReactNodeTesting - DevOps CI/CD Pipeline

## Overview
This repository contains a ReactJS and NodeJS application with a complete DevOps CI/CD implementation. The project was cloned from [eljamaki01/ReactNodeTesting](https://github.com/eljamaki01/ReactNodeTesting) and configured with automated testing, deployment pipelines, and AWS infrastructure.

## AWS Infrastructure

### EC2 Instances
Two AWS EC2 instances are configured for different deployment environments:

**Testing Environment**
- **Purpose**: QA validation and testing of pull request changes
- **AMI**: Ubuntu Server 24.* LTS (HVM)
- **Dependencies**: Node.js 18, npm, PM2, all required React and NodeJS packages
- **Access**: http://<Testing_Instance_IP>:3000
- **Deployment Trigger**: Pull requests to main branch
- **Process Management**: PM2 (`testing-app`)

**Staging Environment**
- **Purpose**: Production-like environment for client and stakeholder access
- **AMI**: Ubuntu Server 24.* LTS (HVM)
- **Dependencies**: Node.js 18, npm, PM2, all required React and NodeJS packages
- **Access**: http://<Staging_Instance_IP>:3000
- **Deployment Trigger**: Push to main branch
- **Process Management**: PM2 (`staging-app`)

### Security Configuration
- **Shared Security Group**: Both instances use a common security group for centralized rule management
- **Inbound Rules**: Configured to allow HTTP traffic (port 80/3000) and SSH access (port 22)
- **Outbound Rules**: Configured for necessary package installations and updates
- Any security rule changes automatically apply to all instances

## GitHub Actions Workflows

### Pull Request Workflow (Testing Deployment)
This workflow automatically triggers when a pull request is created from any feature or fix branch. It can also be manually triggered using `workflow_dispatch`.

**Pipeline Stages:**
1. **Build Stage**: 
   - Checks out the code from the PR branch using `actions/checkout@v4`
   - Sets up Node.js 18 environment
   - Installs all npm dependencies
   - Runs `npm run build-react` to compile React application
   - Validates successful build completion

2. **Unit Testing**: 
   - Runs `npm test -- --watchAll=false` for React component tests
   - Tests include: todoTitle mock tests, App components, cart operations, zipcode entry
   - Uses Jest and React Testing Library
   - Fails the pipeline if any test fails

3. **Code Quality/Linting**: 
   - Runs `npx eslint src/` for code quality checks
   - Uses react-app ESLint configuration
   - Identifies potential bugs and code smells
   - Warnings are logged but don't fail the build

4. **AWS Testing Deployment**: 
   - Connects to Testing EC2 instance via SSH using `appleboy/ssh-action`
   - Clones repository if not exists, otherwise fetches updates
   - Checks out and pulls the PR branch (`github.head_ref`)
   - Runs `npm install` and `npm run build-react`
   - Restarts application with PM2: `pm2 restart testing-app` or starts new process
   - Application runs on port 3000

5. **Email Notifications**: 
   - Uses `dawidd6/action-send-mail` with Gmail SMTP
   - **On Failure**: Sends email with subject "❌ Testing Deployment Failed" and GitHub Actions log reference
   - **On Success**: Sends email with subject "✅ Testing Deployment Successful", PR branch name, and access link (http://<Testing_Instance_IP>:3000)

### Main Branch Workflow (Staging Deployment)
This workflow automatically triggers when changes are merged into the main branch (after QA approval). It can also be manually triggered using `workflow_dispatch`.

**Pipeline Stages:**
1. **Build Stage**: 
   - Checks out code from main branch using `actions/checkout@v4`
   - Sets up Node.js 18 environment
   - Installs dependencies with `npm install`
   - Runs `npm run build-react` for production-ready bundle

2. **Unit Testing**: 
   - Runs complete test suite
   - Ensures all tests pass before deployment
   - Maintains code quality standards

3. **Code Quality/Linting**: 
   - Performs final code quality checks
   - Validates adherence to coding standards

4. **AWS Staging Deployment**: 
   - Connects to Staging EC2 instance via SSH using `appleboy/ssh-action`
   - Navigates to `~/ReactNodeTesting-CICD` or clones repository if needed
   - Pulls latest changes from main branch
   - Runs `npm install` and `npm run build-react`
   - Restarts all PM2 processes: `pm2 restart all` or starts new `staging-app`
   - Application runs on port 3000

5. **Email Notifications**: 
   - Uses `dawidd6/action-send-mail` with Gmail SMTP
   - **On Failure**: Sends email with subject "❌ Staging Deployment Failed" and GitHub Actions log reference
   - **On Success**: Sends email with subject "✅ Staging Deployment Successful" and access link (http://<Staging_Instance_IP>:3000)

### Workflow Features
- **Automated Triggers**: PR creation and main branch pushes
- **Manual Dispatch**: Both workflows can be manually triggered via GitHub Actions UI
- **Environment Secrets**: AWS credentials, SSH keys, and email configurations stored securely
- **Deployment Logs**: Complete logs available in GitHub Actions for debugging

### Required GitHub Secrets
The following secrets are configured in the repository settings for the workflows to function:

- **`DEV_EMAIL`**: Gmail address used for sending deployment notifications
- **`DEV_EMAIL_PASSWORD`**: App password for Gmail SMTP authentication
- **`EC2_USER`**: SSH username for EC2 instances (typically `ubuntu`)
- **`TESTING_EC2_IP`**: Public IP address of the Testing EC2 instance
- **`TESTING_SSH_KEY`**: Private SSH key for accessing Testing EC2 instance
- **`STAGING_EC2_IP`**: Public IP address of the Staging EC2 instance
- **`STAGING_SSH_KEY`**: Private SSH key for accessing Staging EC2 instance
