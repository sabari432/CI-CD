# Edu-Site React App Deployment

This repository contains a ReactJS education website deployed using GitLab CI/CD pipeline to an AWS EC2 instance.

---

## Overview

- Build React app using Node 18 Alpine image.
- Deploy the built React app to an EC2 instance running nginx.
- Deployment uses `rsync` over SSH with private key authentication.
- Nginx is restarted automatically after deployment to serve the latest build.

---

## SSH Key Setup for Deployment

To enable passwordless SSH access for GitLab CI/CD to your EC2 server:

1. **Generate a new SSH key pair** (on your local machine or CI runner):

    ```bash
    ssh-keygen -t rsa -b 4096 -C "gitlab-deploy-key"
    ```

2. **Copy the public key** to add it to your EC2 server:

    ```bash
    cat ~/.ssh/id_rsa.pub
    ```

3. **Login to your EC2 instance** (adjust user and key file as needed):

    ```bash
    ssh -i static.pem ec2-user@3.109.213.216
    ```

4. **Add the public key to the authorized keys on the EC2 server**:

    ```bash
    echo "paste-your-public-key-content-here" >> ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    ```

5. **Test SSH access using the private key**:

    ```bash
    ssh -i ~/.ssh/id_rsa ubuntu@3.109.213.216
    ```
6. **Get Private Key**:

    ```bash
    cat ~/.ssh/id_rsa
    ```
---

## GitLab CI/CD Pipeline Configuration

```yaml
stages:
  - build
  - deploy

# Step 1: Build React app
build_react:
  stage: build
  image: node:18-alpine
  script:
    - npm install --legacy-peer-deps
    - CI=false npm run build
  artifacts:
    paths:
      - build/
  only:
    - main

# Step 2: Deploy to EC2
deploy_to_ec2:
  stage: deploy
  image: alpine:latest
  dependencies:
    - build_react
  before_script:
    - apk add --no-cache openssh-client rsync
    - mkdir -p ~/.ssh
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' > ~/.ssh/id_rsa
    - chmod 600 ~/.ssh/id_rsa
    - ssh-keyscan -H 3.109.213.216 >> ~/.ssh/known_hosts
  script:
    # Use sudo to write to /var/www/html (if needed)
    - rsync -avz --delete --rsync-path="sudo rsync" build/ ubuntu@3.109.213.216:/var/www/html/
    # Restart nginx to serve the updated site
    - ssh ubuntu@3.109.213.216 "sudo systemctl restart nginx"
  only:
    - main
