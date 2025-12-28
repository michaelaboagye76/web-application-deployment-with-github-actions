

# **Flask ECS Deployment with Nginx and GitHub Actions**

## **Overview**

This project demonstrates how to build, containerize, and deploy a Flask web application to **AWS ECS (Fargate)** using **Amazon ECR** as the container registry and **GitHub Actions** for automated CI/CD.
The setup includes **Nginx** as a reverse proxy for production-grade request handling.

---

## **Architecture Summary**

* **Frontend/Backend:** Flask web application (Python)
* **Web Server:** Nginx (Reverse Proxy)
* **Containerization:** Docker
* **Registry:** Amazon Elastic Container Registry (ECR)
* **Hosting:** Amazon Elastic Container Service (ECS) with Fargate
* **Automation:** GitHub Actions for CI/CD pipeline

![Alt text](diagram.jpg)
---

## **Project Structure**

```
electronics-store-web-app/
│
├── app.py
├── requirements.txt
├── Dockerfile
├── nginx.conf
├── supervisord.conf
├── templates/
│   ├── index.html
│   ├── contact.html
│   └── ...
├── static/
│   ├── css/
│   ├── js/
│   └── images/
└── .github/
    └── workflows/
        └── deploy.yml
```

---

## **1. Dockerization**

### **Dockerfile**

The application is containerized with both Flask and Nginx configured to run together.
Below is the production-ready Dockerfile used in the build:

```dockerfile
FROM python:3.11-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Copy Nginx configuration
RUN apt-get update && apt-get install -y nginx supervisor
COPY nginx.conf /etc/nginx/nginx.conf
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf

EXPOSE 80
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/conf.d/supervisord.conf"]
```

### **Nginx Configuration**

`nginx.conf` routes HTTP traffic to the Flask backend:

```nginx
events {}

http {
    server {
        listen 80;
        location / {
            proxy_pass http://127.0.0.1:5000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

### **Supervisor Configuration**

`supervisord.conf` ensures both Nginx and Flask run in the same container:

```ini
[supervisord]
nodaemon=true

[program:flask]
command=python app.py
directory=/app
autostart=true
autorestart=true

[program:nginx]
command=nginx -g 'daemon off;'
autostart=true
autorestart=true
```

---

## **2. Amazon ECR Setup**

### **Create a Repository**

1. Go to **Amazon ECR Console** → **Repositories → Create Repository**
2. Name: `flask-ecs-repo`
3. Visibility: **Private**
4. Note the repository URI:

   ```
   957839002008.dkr.ecr.us-east-1.amazonaws.com/flask-ecs-repo
   ```

### **Authenticate Docker with ECR**

Run:

```bash
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 957839002008.dkr.ecr.us-east-1.amazonaws.com
```

### **Build and Push Image**

```bash
docker build -t flask-ecs-repo .
docker tag flask-ecs-repo:latest 957839002008.dkr.ecr.us-east-1.amazonaws.com/flask-ecs-repo:latest
docker push 957839002008.dkr.ecr.us-east-1.amazonaws.com/flask-ecs-repo:latest
```

---

## **3. Amazon ECS Setup (Fargate)**

### **Cluster**

* Name: `flask-ecs-cluster`
* Launch type: **Fargate**

### **Task Definition**

* Family: `flask-ngnix-task`
* Container: `flask-ngnix-app`

  * Image: `957839002008.dkr.ecr.us-east-1.amazonaws.com/flask-ecs-repo:latest`
  * Port mappings: **80:80**
  * Log driver: `awslogs`

### **Service**

* Service name: `flask-service`
* Launch type: **Fargate**
* Desired tasks: **1**
* Cluster: `flask-ecs-cluster`
* Task Definition: `flask-ngnix-task:latest`
* Assign public IP: **ENABLED**

### **Security Group**

Allow:

* **Inbound TCP 80 (HTTP)**
* **Outbound All traffic**

---

## **4. GitHub Actions for CI/CD**

### **GitHub Secrets**

In your repository settings → **Secrets and variables → Actions**, add the following:

| Secret Name              | Description         |
| ------------------------ | ------------------- |
| `AWS_ACCESS_KEY_ID`      | Your IAM access key |
| `AWS_SECRET_ACCESS_KEY`  | Your IAM secret key |
| `AWS_REGION`             | `us-east-1`         |
| `AWS_ACCOUNT_ID`         | `957839002008`      |
| `ECR_REPOSITORY`         | `flask-ecs-repo`    |
| `ECS_CLUSTER`            | `flask-ecs-cluster` |
| `ECS_SERVICE`            | `flask-service`     |
| `TASK_DEFINITION_FAMILY` | `flask-ngnix-task`  |

---

### **GitHub Actions Workflow**

`.github/workflows/deploy.yml`
This pipeline builds the image, pushes it to ECR, and updates ECS automatically.

```yaml
name: Deploy to AWS ECS

on:
  push:
    branches: [main]

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}
  ECS_CLUSTER: ${{ secrets.ECS_CLUSTER }}
  ECS_SERVICE: ${{ secrets.ECS_SERVICE }}
  TASK_DEFINITION_FAMILY: ${{ secrets.TASK_DEFINITION_FAMILY }}

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Login to Amazon ECR
        run: |
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

      - name: Build Docker image
        run: docker build -t $ECR_REPOSITORY .

      - name: Tag Docker image
        run: docker tag $ECR_REPOSITORY:latest $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest

      - name: Push to Amazon ECR
        run: docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest

      - name: Register new ECS task definition revision
        run: |
          echo "Creating new ECS task definition..."
          aws ecs register-task-definition \
            --family $TASK_DEFINITION_FAMILY \
            --execution-role-arn arn:aws:iam::$AWS_ACCOUNT_ID:role/ecsTaskExecutionRole \
            --network-mode awsvpc \
            --requires-compatibilities FARGATE \
            --cpu "512" \
            --memory "1024" \
            --container-definitions "[
              {
                \"name\": \"flask-ngnix-app\",
                \"image\": \"$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPOSITORY:latest\",
                \"portMappings\": [{\"containerPort\":80,\"protocol\":\"tcp\"}],
                \"essential\": true,
                \"logConfiguration\": {
                  \"logDriver\": \"awslogs\",
                  \"options\": {
                    \"awslogs-group\": \"/ecs/$TASK_DEFINITION_FAMILY\",
                    \"awslogs-region\": \"$AWS_REGION\",
                    \"awslogs-stream-prefix\": \"ecs\"
                  }
                }
              }
            ]"

      - name: Update ECS service
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE \
            --force-new-deployment
```

---

## **5. Deployment Verification**

### **Check ECS**

In the AWS ECS Console:

1. Go to **Clusters → flask-ecs-cluster → Services → flask-service**
2. Confirm:

   * Desired count = 1
   * Running count = 1
   * Deployment = "PRIMARY"

### **View Logs**

Navigate to:
**CloudWatch → Log groups → /ecs/flask-ngnix-task**

You should see logs similar to:

```
* Running on http://0.0.0.0:80
```

### **Access Application**

* If using a Load Balancer: use the **DNS name** from EC2 → Load Balancers.
* If no Load Balancer: use the **Public IP** of the running ECS task.

Example:

```
http://44.204.xxx.xxx
```

---

## **6. Continuous Deployment**

Each time you push a commit to the `main` branch:

1. GitHub Actions triggers automatically.
2. Builds a new Docker image.
3. Pushes the image to Amazon ECR.
4. Registers a new ECS task definition.
5. Updates your ECS service with zero downtime.

---

## **7. Troubleshooting**

| Issue                                        | Possible Cause             | Resolution                                                          |
| -------------------------------------------- | -------------------------- | ------------------------------------------------------------------- |
| Task stops after deployment                  | Wrong port mapping         | Ensure Nginx and Flask both use port 80                             |
| ECS cannot pull image                        | Missing ECR permissions    | Attach `AmazonECSTaskExecutionRolePolicy` to `ecsTaskExecutionRole` |
| App unreachable                              | Security group restriction | Allow inbound TCP 80 (HTTP)                                         |
| GitHub Actions error: DescribeTaskDefinition | Wrong task family name     | Confirm `TASK_DEFINITION_FAMILY` secret matches ECS family name     |

---

## **8. Summary**

This workflow establishes a complete production-ready Flask deployment pipeline:

* Flask app containerized with Nginx and Supervisor
* ECR for container registry
* ECS Fargate for scalable hosting
* GitHub Actions for automated build and deploy

This setup ensures continuous integration, seamless updates, and reliable production hosting using fully managed AWS services.
