# 🚀 CI/CD Pipeline with GitHub Actions and AWS EC2 (Self-Hosted Runner)

## 🎯 Goal
Build a **fully automated CI/CD pipeline** using **GitHub Actions** and a **self-hosted runner** on an AWS EC2 instance.  
The pipeline automatically:
- builds a Docker image  
- runs tests with pytest  
- deploys the FastAPI app to the EC2 server when tests pass  

---

## 🧩 Steps to Reproduce

### 🪜 Step 1 — Create a Free AWS Account
👉 [https://aws.amazon.com/free/](https://aws.amazon.com/free/)  
AWS offers a 12-month free tier that’s enough for this demo.

---

### 🪜 Step 2 — Create an IAM Admin User
- Go to **IAM → Users → Create user**  
- Create an **admin user** (instead of using the root account) for security reasons.  
- Attach the **AdministratorAccess** policy.

---

### 🪜 Step 3 — Create a Key Pair
- Go to **EC2 → Key pairs → Create key pair**  
- Choose type **RSA** and format **.pem**  
- Download the `.pem` file — you’ll need it for SSH access later.

---

### 🪜 Step 4 — Launch an EC2 Instance
- Go to **EC2 → Instances → Launch instance**
- Name: `ci-cd-runner`
- AMI: **Ubuntu 22.04**
- Instance type: **t3.micro**
- Key pair: select your downloaded `.pem`
- Security group:  
  - allow **SSH (22)** from *My IP*  
  - allow **HTTP (80)** from *0.0.0.0/0*

---

### 🪜 Step 5 — Connect & Install Git and Docker
SSH into the server:
```bash
ssh -i ~/.ssh/your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

#### Install Docker & Git:

```bash
sudo apt update && sudo apt install -y git
sudo apt install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker ubuntu
newgrp docker
```
### 🪜 Step 6 — Create the App Files

- **main.py**

- **requirements.txt**

- **Dockerfile**

- **docker-compose.yml**

- **test_api.py**

### 🪜 Step 7 — Set Up GitHub Actions (Basic Test Workflow)

Create file: `.github/workflows/test.yml`

Example:

```yaml
name: test-pipeline

on:
  push:
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: checkout repository
        uses: actions/checkout@v4

      - name: set up python
        uses: actions/setup-python@v5
        with:
          python-version: "3.11"

      - name: install dependencies
        run: |
          python -m pip install --upgrade pip
          if [ -f requirements.txt ]; then
            pip install -r requirements.txt
          else
            pip install fastapi uvicorn pytest httpx
          fi

      - name: run tests
        run: pytest -s -v
```

Push this to GitHub and confirm it runs successfully.

### 🪜 Step 8 — Install Self-Hosted Runner on EC2

On your EC2 terminal:

```bash
cd ~
mkdir actions-runner && cd actions-runner

curl -o actions-runner-linux-x64-2.328.0.tar.gz -L \
https://github.com/actions/runner/releases/download/v2.328.0/actions-runner-linux-x64-2.328.0.tar.gz

tar xzf actions-runner-linux-x64-2.328.0.tar.gz
```

Then from GitHub → Settings → Actions → Runners → New self-hosted runner → Linux x64
Copy the URL + Token and run (just change your username, repo and token):

```bash
./config.sh --url https://github.com/artyomashigov/ci_cd_test \
  --token <YOUR_TOKEN_HERE> \
  --labels ec2-ci \
  --unattended

sudo ./svc.sh install
sudo ./svc.sh start
sudo ./svc.sh status
```
✅ You should see “service is running” and the runner Online in GitHub.

### 🪜 Step 9 — Full CI/CD Workflow (Build → Test → Deploy)

Replace `.github/workflows/test.yml` with:

```yaml
name: test-pipeline

on:
  push:
  pull_request:

jobs:
  build:
    name: Build Docker image
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t my-backend-image:latest .

  test:
    name: Run API tests
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Build Docker image for testing
        run: docker build -t my-backend-image:latest .

      - name: Run tests with pytest
        run: docker run --rm my-backend-image:latest pytest -s -v

  deploy:
    name: Deploy to EC2
    runs-on: [self-hosted, linux, ec2-ci]   # tags from your GitHub runner
    needs: test
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Deploy with Docker Compose
        run: |
          docker compose down || true
          docker compose up -d --build
```

Push changes and watch Actions → build → test → deploy.

### 🪜 Step 10 — Test the Pipeline

Change your app message in `main.py` and your test in `test_api.py`.
Push changes to GitHub — it will:

1. rebuild the Docker image

2. rerun tests (fail if mismatch)

3. redeploy automatically when tests pass

Check your app at:

```cpp
http://<EC2_PUBLIC_IP>/
```

### ✅ Result

You now have a working FastAPI + Docker + GitHub Actions CI/CD pipeline
fully automated and hosted on your own EC2 instance! 🎉

