# Flask Application with Docker and GitHub Actions Deployment

This project is a simple Flask web application that serves a "Hello, Docker!" message. It is containerized using Docker and deployed to an AWS EC2 Ubuntu server using a GitHub Actions pipeline. The pipeline builds the Docker image, pushes it to Docker Hub, and deploys it to the EC2 instance via SSH.

## Project Structure

- app.py: The Flask application code.
- Dockerfile: Defines the Docker image for the Flask app.
- requirements.txt: Lists Python dependencies (Flask).
- .github/workflows/workflow.yml: GitHub Actions pipeline for building, pushing, and deploying the Docker image.

## Prerequisites

- **Docker**: Installed on your local machine and the EC2 instance.
- **Docker Hub Account**: For storing the Docker image.
- **GitHub Repository**: To store the code and run the pipeline.
- **AWS EC2 Instance**: Running Ubuntu with SSH access and Docker installed.
- **Git**: For version control and pushing code to GitHub.

## Setup Instructions

### 1. Clone the Repository

Clone this repository to your local machine:

```bash
git clone https://github.com/yourusername/your-repo.git
cd your-repo
```

Replace yourusername and your-repo with your GitHub username and repository name.

### 2. Create and Test the Flask Application

The Flask application (app.py) is a simple web server that responds with "Hello, Docker!" at the root endpoint (/).

**app.py**:

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return 'Hello, Docker!'

if __name__ == '__main__':
    app.run(debug=True, host='0.0.0.0')
```

**requirements.txt**:

```
Flask==2.3.2
```

To test locally:

1. Create a virtual environment:
    
    ```bash
    python3 -m venv venv
    source venv/bin/activate
    ```
    
2. Install dependencies:
    
    ```bash
    pip install -r requirements.txt
    ```
    
3. Run the Flask app:
    
    ```bash
    python app.py
    ```
    
4. Open http://localhost:5000 in a browser. You should see "Hello, Docker!".

### 3. Dockerize the Application

The Dockerfile builds a Docker image for the Flask app.

**Dockerfile**:

```
FROM python:3.9-slim
WORKDIR /app/
COPY requirements.txt .
RUN pip install --upgrade pip
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

**Build and test the Docker image locally**:

1. Build the image:
    
    ```bash
    docker build -t yourusername/hello-flask:latest .
    ```
    
    Replace yourusername with your Docker Hub username.
    
2. Run the container:
    
    ```bash
    docker run -d -p 80:5000 --name hello-flask yourusername/hello-flask:latest
    ```
    
3. Open http://localhost in a browser to verify the app is running.
4. Stop and remove the container:
    
    ```bash
    docker stop hello-flask
    docker rm hello-flask
    ```
    

### 4. Configure Docker Hub

1. **Create a Docker Hub account**: Sign up at hub.docker.com.
2. **Create a repository**: Create a repository named hello-flask (public or private).
3. **Add Docker Hub credentials to GitHub Secrets**:
    - Go to your GitHub repository → Settings → Secrets and variables → Actions → Secrets.
    - Add two secrets:
        - Name: DOCKER_USERNAMEValue: Your Docker Hub username (e.g., yourusername).
        - Name: DOCKER_PASSWORDValue: Your Docker Hub password or an access token (recommended).
            - To create an access token: In Docker Hub, go to Account Settings → Security → New Access Token.

### 5. Set Up the EC2 Ubuntu Server

1. **Launch an EC2 instance**:
    - In AWS Management Console, launch an Ubuntu-based EC2 instance (e.g., Ubuntu 20.04 or 22.04).
    - Ensure the security group allows:
        - Inbound SSH (port 22) from your IP or 0.0.0.0/0 (restrict in production).
        - Inbound HTTP (port 80) for the Flask app.
    - Note the instance’s public IP or DNS (e.g., ec2-XX-XXX-XXX-XXX.compute-1.amazonaws.com).
2. **Install Docker on the EC2 instance**: SSH into the EC2 instance:
    
    ```bash
    ssh -i your-key.pem ubuntu@<EC2_PUBLIC_IP>
    ```
    
    Install Docker:
    
    ```bash
    sudo apt update
    sudo apt install -y docker.io
    sudo systemctl start docker
    sudo systemctl enable docker
    ```
    
    Add the ubuntu user to the Docker group:
    
    ```bash
    sudo usermod -aG docker ubuntu
    ```
    
    Log out and back in to apply the group change.
    
3. **Generate a new SSH key pair**: On the EC2 instance, create a new SSH key for GitHub Actions:
    
    ```bash
    ssh-keygen -t ed25519 -C "github-actions-key" -f ~/.ssh/github_actions_key -N ""
    ```
    
    This creates ~/.ssh/github_actions_key (private key) and ~/.ssh/github_actions_key.pub (public key) with no passphrase.
    
4. **Add the public key to** authorized_keys:
    
    ```bash
    cat ~/.ssh/github_actions_key.pub >> ~/.ssh/authorized_keys
    chmod 700 ~/.ssh
    chmod 600 ~/.ssh/authorized_keys ~/.ssh/github_actions_key ~/.ssh/github_actions_key.pub
    chown ubuntu:ubuntu ~/.ssh ~/.ssh/authorized_keys ~/.ssh/github_actions_key ~/.ssh/github_actions_key.pub
    ```
    
5. **Copy the private key**: Display the private key:
    
    ```bash
    cat ~/.ssh/github_actions_key
    ```
    
    Copy the entire output (including -----BEGIN OPENSSH PRIVATE KEY----- and -----END OPENSSH PRIVATE KEY-----).
    
6. **Add SSH key to GitHub Secrets**:
    - In your GitHub repository → Settings → Secrets and variables → Actions → Secrets.
    - Add two secrets:
        - Name: EC2_SSH_KEYValue: Paste the private key from the previous step.
        - Name: EC2_SSH_PASSPHRASE (optional) Value: The passphrase if you set one (skip if no passphrase).
    - Add EC2 credentials:
        - Name: EC2_HOSTValue: The EC2 instance’s public IP or DNS.
        - Name: EC2_USERValue: ubuntu (or the appropriate username for your AMI).
7. **(Optional) Configure Docker Hub credentials on EC2**: If the hello-flask image is private, log in to Docker Hub on the EC2 instance:
    
    ```bash
    docker login -u yourusername -p yourpassword
    ```
    
    Alternatively, the workflow logs in using DOCKER_USERNAME and DOCKER_PASSWORD (as shown below).
    

### 6. Configure the GitHub Actions Pipeline

The pipeline builds the Docker image, pushes it to Docker Hub, and deploys it to the EC2 instance.

.github/workflows/workflow.yml:

```yaml
name: Docker Image Push

on:
  push:
    branches:
      - master

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v3
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKER_USERNAME }}/hello-flask:latest

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          passphrase: ${{ secrets.EC2_SSH_PASSPHRASE }} # Remove if no passphrase
          port: 22
          debug: true
          script: |
            whoami
            docker --version
            echo "${{ secrets.DOCKER_PASSWORD }}" | docker login -u "${{ secrets.DOCKER_USERNAME }}" --password-stdin
            docker pull ${{ secrets.DOCKER_USERNAME }}/hello-flask:latest
            docker stop hello-flask || true
            docker rm hello-flask || true
            docker run -d --name hello-flask -p 80:5000 ${{ secrets.DOCKER_USERNAME }}/hello-flask:latest
```

**Notes**:

- The debug: true option provides verbose logs for troubleshooting.
- The docker login command in the script ensures the EC2 instance can pull private images.
- Update appleboy/ssh-action to the latest version if needed (e.g., v1.0.3 as of June 2025).

### 7. Deploy and Test

1. **Push code to GitHub**:
    
    ```bash
    git add .
    git commit -m "Add Flask app and pipeline"
    git push origin master
    ```
    
    This triggers the GitHub Actions pipeline.
    
2. **Monitor the pipeline**:
    - Go to your repository → Actions → Select the workflow run.
    - Check the logs for each step, especially “Deploy to EC2” for SSH or Docker errors.
3. **Verify the deployment**:
    - Open http://<EC2_PUBLIC_IP> in a browser. You should see "Hello, Docker!".
    - SSH into the EC2 instance and check the container:
        
        ```bash
        docker ps
        ```
