# Combined Deployment Configuration Files

This document combines all deployment configuration files for a full-stack React application with CI/CD pipeline and AWS deployment.

---

## 1. Dockerfile

**Purpose:** Multi-stage Docker build for React application with Nginx server

```dockerfile
# ============================
# Dockerfile (Multi-Stage Build)
# ============================

# ---------- Stage 1: Build the React App ----------
FROM node:18 AS build

# Set working directory
WORKDIR /app

# Copy package files and install dependencies
COPY package*.json ./
RUN npm install

# Copy all source files and build the app
COPY . .
RUN npm run build

# ---------- Stage 2: Serve with Nginx ----------
FROM nginx:alpine

# Copy build output from previous stage to Nginx web root
COPY --from=build /app/build /usr/share/nginx/html

# Expose port 80
EXPOSE 80

# Start Nginx server
CMD ["nginx", "-g", "daemon off;"]
```

---

## 2. .dockerignore

**Purpose:** Exclude unnecessary files from Docker build context

```
node_modules
build
.git
.gitignore
Dockerfile
.dockerignore
README.md
```

---

## 3. GitHub Actions Workflow (main.yml)

**Purpose:** CI/CD Pipeline for React App deployment

**File Location:** `.github/workflows/main.yml`

```yaml
# ============================================
# File: .github/workflows/main.yml
# ============================================
# GitHub Actions CI/CD Pipeline for React App
# ============================================

name: CI/CD Pipeline for React App

on:
  push:
    branches:
      - main  # Trigger workflow on push to the main branch

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      # Step 1: Checkout the repository
      - name: Checkout repository
        uses: actions/checkout@v4

      # Step 2: Set up Node.js
      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'

      # Step 3: Install dependencies
      - name: Install dependencies
        run: npm install

      # Step 4: Run tests
      - name: Run tests
        run: npm test --if-present

      # Step 5: Build the React app
      - name: Build the project
        run: npm run build

      # Step 6 (Optional): Deploy to GitHub Pages
      - name: Deploy to GitHub Pages
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./build
```

---

## 4. AWS Full Stack Deployment Script (deploy_fullstack_aws.sh)

**Purpose:** Automated deployment script for full-stack application on AWS with Load Balancer

```bash
#!/bin/bash

# ==============================================
# Full Stack Deployment on AWS with Load Balancer
# ==============================================

# ---------- 1. Backend: Node.js / Express App ----------
# File: server.js
cat > server.js <<'EOF'
const express = require('express');
const cors = require('cors');
const app = express();

app.use(cors());

app.get('/', (req, res) => {
  res.send('Hello from Node.js backend - served via AWS Load Balancer!');
});

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`Backend running on port ${PORT}`));
EOF

# Initialize and install dependencies
npm init -y
npm install express cors

# ---------- 2. Frontend: React App ----------
# Create React app (simplified example)
npx create-react-app frontend
cd frontend

cat > src/App.js <<'EOF'
import React from "react";

function App() {
  return (
    <div style={{ textAlign: "center", marginTop: "50px" }}>
      <h1>Full Stack App on AWS</h1>
      <p>Accessing backend API: <a href="http://your-load-balancer-dns:5000">Click Here</a></p>
    </div>
  );
}

export default App;
EOF

cd ..

# ---------- 3. Dockerfiles ----------
# Backend Dockerfile
cat > Dockerfile.backend <<'EOF'
FROM node:18
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["node", "server.js"]
EOF

# Frontend Dockerfile
cat > frontend/Dockerfile <<'EOF'
FROM node:18 AS build
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF

# ---------- 4. Deploy to AWS ----------
echo "Building and pushing Docker images to AWS ECR..."

# Authenticate Docker to AWS ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin YOUR_AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build and push backend image
docker build -t backend -f Dockerfile.backend .
docker tag backend:latest YOUR_AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/backend:latest
docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/backend:latest

# Build and push frontend image
cd frontend
docker build -t frontend .
docker tag frontend:latest YOUR_AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/frontend:latest
docker push YOUR_AWS_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/frontend:latest

echo "Deployment script completed. Configure ECS tasks and Load Balancer in AWS Console."
```

---

## File Structure Overview

```
project-root/
├── Dockerfile                          # Multi-stage build for React + Nginx
├── .dockerignore                       # Docker build exclusions
├── .github/
│   └── workflows/
│       └── main.yml                    # GitHub Actions CI/CD pipeline
├── deploy_fullstack_aws.sh             # AWS deployment automation script
├── server.js                           # Backend Express server (generated by script)
└── frontend/                           # React frontend (generated by script)
    ├── Dockerfile                      # Frontend Docker configuration
    └── src/
        └── App.js                      # Main React component
```

---

## Usage Instructions

### Using Docker
```bash
# Build the Docker image
docker build -t react-app .

# Run the container
docker run -p 80:80 react-app
```

### Using GitHub Actions
1. Push code to the `main` branch
2. GitHub Actions will automatically:
   - Install dependencies
   - Run tests
   - Build the project
   - Deploy to GitHub Pages

### Using AWS Deployment Script
```bash
# Make the script executable
chmod +x deploy_fullstack_aws.sh

# Run the deployment script
./deploy_fullstack_aws.sh
```

**Note:** Replace `YOUR_AWS_ACCOUNT_ID` in the script with your actual AWS account ID before running.

---

## Configuration Requirements

- **Node.js**: Version 18 or higher
- **Docker**: Latest stable version
- **AWS CLI**: Configured with appropriate credentials
- **GitHub**: Repository with Actions enabled
- **AWS Services**: ECR, ECS, and Load Balancer configured

---

## Summary

This combined source file includes:
1. **Dockerfile**: Multi-stage build configuration for React app with Nginx
2. **.dockerignore**: Build optimization by excluding unnecessary files
3. **GitHub Actions Workflow**: Automated CI/CD pipeline for testing and deployment
4. **AWS Deployment Script**: Full-stack deployment automation with backend, frontend, and load balancer setup
