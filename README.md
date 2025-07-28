# 🚀 React App CI/CD Pipeline with AWS EC2, GitHub Actions & NGINX

A complete DevOps solution for automatically deploying React.js applications to AWS EC2 instances using GitHub Actions for continuous integration and delivery, with NGINX as the production web server.

## Overview

This project demonstrates a modern, automated deployment pipeline that:
- Builds React applications on every push to the main branch
- Automatically deploys to AWS EC2 instances via SSH
- Configures NGINX for optimal production serving
- Provides zero-downtime deployments with proper error handling

**🖼️ Live Demo Screenshot**
![App Architecture](Docs/browsing test.png)

---

##  Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Developer     │    │   GitHub        │    │   AWS EC2       │
│   Push Code     │───▶│   Actions       │───▶│   + NGINX       │
│                 │    │   CI/CD         │    │   Production    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**🖼️ Architecture Diagram**
*[Diagram placeholder: Visual representation of the CI/CD pipeline]*

---

## Prerequisites

Before setting up this project, ensure you have:

- **AWS Account** with EC2 instance running Ubuntu 20.04+
- **GitHub Repository** with admin access
- **Domain/IP** for your EC2 instance
- **SSH Key Pair** for EC2 access
- **Node.js 16+** for local development

---

## Project Structure

```
react-cicd-project/
├── .github/
│   └── workflows/
│       └── deploy.yml          # GitHub Actions workflow
├── public/
│   ├── index.html
│   └── manifest.json
├── src/
│   ├── components/
│   ├── App.js
│   └── index.js
├── nginx/
│   └── default.conf            # NGINX configuration
├── scripts/
│   └── deploy.sh               # Deployment script
├── package.json
├── package-lock.json
└── README.md
```

**🖼️ Project Structure**
![App Architecture](Docs/structure.png)


---

## Setup Guide

### 1. AWS EC2 Configuration

**Launch EC2 Instance:**
```bash
# Instance Type: t2.micro (Free Tier)
# OS: Ubuntu 20.04 LTS
# Security Group: Allow HTTP (80), HTTPS (443), SSH (22)
```

**🖼️ EC2 Dashboard**
![App Architecture](Docs/aws-ec2.png)


### 2. GitHub Secrets Configuration

Navigate to your repository: `Settings → Secrets and variables → Actions`

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `AWS_EC2_HOST` | EC2 Public IP Address | `54.123.45.67` |
| `AWS_EC2_USER` | SSH Username | `ubuntu` |
| `AWS_SSH_PRIVATE_KEY` | Private Key Content | `-----BEGIN OPENSSH PRIVATE KEY-----...` |

**🖼️ GitHub Secrets**
![App Architecture](Docs/github secrets.png)

### 3. SSH Key Setup

```bash
# Generate SSH key pair (if you don't have one)
ssh-keygen -t rsa -b 4096 -C "your-email@example.com"

# Copy public key to EC2
ssh-copy-id -i ~/.ssh/id_rsa.pub ubuntu@YOUR_EC2_IP

# Add private key to GitHub Secrets
cat ~/.ssh/id_rsa
```

---

## Technologies Used

| Technology | Purpose | Version |
|------------|---------|---------|
| **React.js** | Frontend Framework | 18.x |
| **GitHub Actions** | CI/CD Pipeline | Latest |
| **AWS EC2** | Cloud Infrastructure | Ubuntu 20.04 |
| **NGINX** | Web Server | 1.18+ |
| **Node.js** | JavaScript Runtime | 16.x+ |

---

## GitHub Actions Workflow

### Workflow File: `.github/workflows/deploy.yml`

```yaml
name: Deploy React App to EC2

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout Code
      uses: actions/checkout@v3
      
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
        
    - name: Install Dependencies
      run: npm ci
      
    - name: Build React App
      run: npm run build
      
    - name: Deploy to EC2
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.AWS_EC2_HOST }}
        username: ${{ secrets.AWS_EC2_USER }}
        key: ${{ secrets.AWS_SSH_PRIVATE_KEY }}
        script: |
          # Install NGINX if not present
          sudo apt update
          sudo apt install -y nginx
          
          # Create application directory
          sudo mkdir -p /var/www/react
          sudo chown -R $USER:$USER /var/www/react
          
    - name: Upload Build Files
      uses: appleboy/scp-action@v0.1.4
      with:
        host: ${{ secrets.AWS_EC2_HOST }}
        username: ${{ secrets.AWS_EC2_USER }}
        key: ${{ secrets.AWS_SSH_PRIVATE_KEY }}
        source: "build/*"
        target: "/var/www/react"
        strip_components: 1
        
    - name: Configure NGINX
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.AWS_EC2_HOST }}
        username: ${{ secrets.AWS_EC2_USER }}
        key: ${{ secrets.AWS_SSH_PRIVATE_KEY }}
        script: |
          # Configure NGINX
          sudo tee /etc/nginx/sites-available/react > /dev/null <<EOF
          server {
              listen 80;
              listen [::]:80;
              server_name _;
              
              root /var/www/react;
              index index.html index.htm;
              
              # Handle React Router
              location / {
                  try_files \$uri \$uri/ /index.html;
              }
              
              # Enable Gzip Compression
              gzip on;
              gzip_vary on;
              gzip_min_length 1024;
              gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;
              
              # Cache static assets
              location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
                  expires 1y;
                  add_header Cache-Control "public, immutable";
              }
          }
          EOF
          
          # Enable site
          sudo ln -sf /etc/nginx/sites-available/react /etc/nginx/sites-enabled/
          sudo rm -f /etc/nginx/sites-enabled/default
          
          # Test and reload NGINX
          sudo nginx -t
          sudo systemctl reload nginx
          sudo systemctl enable nginx
```

**🖼️ GitHub Actions Running**
![App Architecture](Docs/deploy Summary.png)

---

## NGINX Configuration

### Production-Ready NGINX Setup

**File: `/etc/nginx/sites-available/react`**

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com www.your-domain.com;
    
    root /var/www/react;
    index index.html index.htm;

    
    location / {
        try_files $uri $uri/ /index.html;
    }
    
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}
```

---

## Deployment Process

### Build Process

```bash
# Local development
npm install
npm start

# Production build
npm run build
# Creates optimized build/ directory
```

**🖼️ Build Output**
![App Architecture](Docs/build details.png)

### Deployment Steps

1. **Code Push**: Developer pushes to main branch
2. **Trigger**: GitHub Actions workflow automatically starts
3. **Build**: React app is built in GitHub's runners
4. **Connect**: SSH connection established to EC2
5. **Deploy**: Files transferred and NGINX configured
6. **Verify**: Application becomes live

**🖼️ Deployment Success**
![App Architecture](Docs/deploy Summary.png)

---

## Testing & Verification

### 1. Check Application Status

```bash
# SSH into your EC2 instance
ssh ubuntu@YOUR_EC2_IP

# Verify NGINX is running
sudo systemctl status nginx

# Check application files
ls -la /var/www/react/

# View NGINX logs
sudo tail -f /var/log/nginx/react_access.log
```

### 2. Browser Testing

Visit your application:
```
http://YOUR_EC2_PUBLIC_IP
```

**🖼️ Live Application**
![App Architecture](Docs/browsing test.png)



### Debug Commands

```bash
# Check NGINX configuration
sudo nginx -t

# View detailed logs
sudo journalctl -u nginx -f

# Restart services
sudo systemctl restart nginx

# Check disk space
df -h
```
---

### Basic Monitoring Setup

```bash
# Install htop for system monitoring
sudo apt install htop

# Monitor NGINX access logs
sudo tail -f /var/log/nginx/react_access.log | grep -E "(GET|POST)"

# Check system resources
free -m
df -h
```

### GitHub Actions Monitoring

- Monitor workflow runs in the Actions tab
- Set up notifications for failed deployments
- Use deployment badges in README

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature-name`
3. Commit changes: `git commit -m "Add feature"`
4. Push to branch: `git push origin feature-name`
5. Submit a pull request

---
**⭐ If this project helped you, please give it a star!**
## 👨‍💻 Author

**Kerolos Mamdouh**
**- 📍 Location:** Cairo, Egypt
**- 📧 Email:** kerolosmamdouh20@gmail.com
**- 💼 LinkedIn:** [Connect with me](https://linkedin.com/in/kerolosmamdouh)
**- 🐙 GitHub:** [@kerolosmamdouh](https://github.com/kerolosmamdouh)
