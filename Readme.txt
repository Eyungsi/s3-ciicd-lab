Resume Template

yum install unzip wget httpd -y
git clone https://github.com/utrains/static-resume.git
rm -rf /var/www/html/*
cp -r static-resume/* /var/www/html/
systemctl start httpd
systemctl enable httpd
<<<<<<< HEAD

=======
.......................................................................................................................................................
GitHub Actions Workflow: Deploy Static Resume to AWS EC2

# Static Resume Deployment to AWS EC2 using GitHub Actions

This README describes the GitHub Actions workflow used to build and deploy a static resume website to AWS EC2 instances for both development and production environments. The CI/CD pipeline automates the build and deployment steps, ensuring consistent delivery across environments.

## Workflow Overview
This GitHub Actions workflow is triggered:
- On every push to the `main` branch
- Or manually via the GitHub Actions UI (`workflow_dispatch`)

It consists of three jobs:
- `build`: Prepares the static files
- `deploy-dev`: Deploys to the development EC2 instance
- `deploy-prod`: Deploys to the production EC2 instance, only after successful dev deployment

## Environments and Secrets
The workflow requires the following secrets to be configured in your GitHub repository:
- `AWS_REGION`: AWS region (e.g., us-east-1)
- `DEV_SERVER_IP`: Public IP address of the development EC2 instance
- `PROD_SERVER_IP`: Public IP address of the production EC2 instance
- `SSH_PRIVATE_KEY`: SSH private key for accessing EC2 instances

## Complete GitHub Actions Workflow with Inline Comments
```yaml
name: Deploy Static Resume to AWS EC2  # Name of the workflow

on:
  push:
    branches: ["main"]               # Trigger workflow on push to main branch
  workflow_dispatch:                  # Allow manual trigger via GitHub UI

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}     # AWS region (e.g., us-east-1)
  APP_NAME: "static-resume"                 # App name (used for readability)
  DEV_SERVER: ${{ secrets.DEV_SERVER_IP }}  # IP address of the Dev EC2 server
  PROD_SERVER: ${{ secrets.PROD_SERVER_IP }}# IP address of the Prod EC2 server

jobs:
  build:
    name: Prepare Static Site
    runs-on: ubuntu-latest                 # Use latest Ubuntu runner
    steps:
      - name: Checkout code
        uses: actions/checkout@v4         # Pulls the latest code from the repo

      - name: Upload HTML files
        uses: actions/upload-artifact@v4  # Uploads build files for use in next jobs
        with:
          name: static-resume-build
          path: ./                        # Set to ./build if using a build subfolder

  deploy-dev:
    name: Deploy to Development EC2
    needs: build                          # Wait for build job to finish
    runs-on: ubuntu-latest
    environment: development              # Targets the development environment
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: static-resume-build
          path: build/                    # Save downloaded files into build/ folder

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.7.0 # Setup SSH to connect to EC2
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Deploy to Dev EC2
        run: |
          rsync -avz -e "ssh -o StrictHostKeyChecking=no" build/ ec2-user@${{ env.DEV_SERVER }}:/tmp/static-resume/  # Sync files to /tmp on EC2

          ssh -o StrictHostKeyChecking=no ec2-user@${{ env.DEV_SERVER }} << 'EOF'  # SSH into EC2
            set -ex                                       # Enable debug and exit on error
            DEPLOY_DIR="/var/www/html/dev"               # Set deploy directory

            if ! command -v httpd &>/dev/null; then       # Install Apache if not installed
              sudo dnf install -y httpd
              sudo systemctl enable httpd
            fi

            sudo systemctl stop httpd || true             # Stop Apache (ignore errors)
            sudo fuser -k 80/tcp || true                  # Kill any process on port 80
            sudo rm -rf $DEPLOY_DIR/*                    # Remove previous site files
            sudo mkdir -p $DEPLOY_DIR                    # Ensure deploy directory exists
            sudo cp -r /tmp/static-resume/* $DEPLOY_DIR/ # Copy files to deploy directory
            sudo chown -R apache:apache $DEPLOY_DIR      # Change ownership to Apache
            sudo chmod -R 755 $DEPLOY_DIR                # Set permissions

            if command -v sestatus &>/dev/null && sudo sestatus | grep -q enabled; then  # Restore SELinux context if enabled
              sudo restorecon -Rv $DEPLOY_DIR
            fi

            sudo apachectl configtest                    # Test Apache configuration
            sudo systemctl start httpd                   # Start Apache
            echo "Dev deployment complete"
          EOF

  deploy-prod:
    name: Deploy to Production EC2
    needs: deploy-dev                   # Wait for dev deployment to complete
    if: github.ref == 'refs/heads/main' # Only run on main branch
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download artifacts
        uses: actions/download-artifact@v4
        with:
          name: static-resume-build
          path: build/

      - name: Setup SSH
        uses: webfactory/ssh-agent@v0.7.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: Deploy to Prod EC2
        run: |
          rsync -avz -e "ssh -o StrictHostKeyChecking=no" build/ ec2-user@${{ env.PROD_SERVER }}:/tmp/static-resume/  # Sync files to production EC2

          ssh -o StrictHostKeyChecking=no ec2-user@${{ env.PROD_SERVER }} << 'EOF'
            set -ex                                       # Enable debug and exit on error
            DEPLOY_DIR="/var/www/html/prod"              # Set deploy directory

            if ! command -v httpd &>/dev/null; then       # Install Apache if not installed
              sudo dnf install -y httpd
              sudo systemctl enable httpd
            fi

            sudo systemctl stop httpd || true             # Stop Apache if running
            sudo fuser -k 80/tcp || true                  # Kill any process using port 80
            sudo rm -rf $DEPLOY_DIR/*                    # Clear old files
            sudo mkdir -p $DEPLOY_DIR                    # Create the directory if needed
            sudo cp -r /tmp/static-resume/* $DEPLOY_DIR/ # Copy new site files
            sudo chown -R apache:apache $DEPLOY_DIR      # Set correct ownership
            sudo chmod -R 755 $DEPLOY_DIR                # Set correct permissions

            if command -v sestatus &>/dev/null && sudo sestatus | grep -q enabled; then  # Restore SELinux context if applicable
              sudo restorecon -Rv $DEPLOY_DIR
            fi

            sudo apachectl configtest                    # Test Apache configuration
            sudo systemctl start httpd                   # Start Apache web server
            curl -I http://localhost/prod || true        # Optional: verify response from local prod path
            echo "Production deployment complete"
          EOF
```

---
This complete document includes workflow setup, explanations for each line, and inline comments to guide new users through the deployment process from source code to EC2 production server.
>>>>>>> 10f47c2 (new changes)

