# Multi-Tier Cloud Architecture for Flask CRUD Application on AWS

## 1. Introduction

This document describes the design, implementation, and deployment of a multi-tier cloud architecture for a Flask CRUD application on AWS. The project aims to make the application production-ready by using a WSGI server, proper networking, security, scalability, and best practices such as HTTPS and a bastion host for secure database access.

---

## 2. Architecture Overview

The project consists of the following three tiers:

### Frontend Tier:
- An Nginx server running on EC2 in a public subnet
- Acts as a reverse proxy, reachable from the internet

### Middle Tier:
- A Flask CRUD app running on ECS Fargate, using Gunicorn as the WSGI server
- Deployed in private subnets

### Backend Tier:
- A managed MySQL RDS database with Multi-AZ deployment
- Deployed in private subnets, only accessible from the ECS service and Bastion Host

### Additional Components:
- **Load Balancer**: Distributes traffic to the Flask app service
- **Bastion Host**: Securely accesses the database
- **NAT Gateway**: Allows private resources to reach the internet for updates
- **Internet Gateway**: Provides internet access to public subnets
- **ECR Repository**: Stores the Flask Docker image

![Architecture Diagram](https://via.placeholder.com/800x500?text=Multi-Tier+Cloud+Architecture+Diagram)

---

## 3. Security Groups Configuration

| Name | Source | Destination | Port | Protocol | Description |
|------|--------|-------------|------|----------|-------------|
| Nginx SG | 0.0.0.0/0 | EC2 Nginx | 80, 443 | TCP | Allow HTTP/HTTPS |
| Bastion SG | My IP | Bastion Host | 22 | TCP | Allow SSH from my IP |
| App SG | Nginx SG | ECS Task | 5000 | TCP | Allow app traffic |
| DB SG | App SG | RDS DB | 3306 | TCP | Allow only app to connect |
| DB SG | Bastion SG | RDS DB | 3306 | TCP | Allow Bastion to connect |

---

## 4. Routing Tables Configuration

| Name | Subnet Group | Destination | Target |
|------|--------------|-------------|--------|
| Public Route Table | Public Subnets | 0.0.0.0/0 | Internet Gateway |
| Private Route Table | Private Subnets | 0.0.0.0/0 | NAT Gateway |

*Local VPC routes remain configured automatically for internal communication.*

---

## 5. IP Addressing

| Component | IP Addressing |
|-----------|---------------|
| Nginx EC2 Instance | Public IP from Public Subnet |
| Bastion Host | Public IP from Public Subnet |
| ECS Task (Flask App) | Private IP from Private Subnet |
| RDS Database | Private IP from Private Subnet |
| Load Balancer | Public IP automatically assigned |

---

## 6. Step-by-Step Implementation

### 6.1 Create the RJ-VPC and Subnets
- Create a VPC named RJ-VPC
- Create 2 public subnets and 3 private subnets
- Enable auto-assign public IP for public subnets

### 6.2 Configure Gateways and Routing Tables
- Attach an Internet Gateway to the VPC
- Deploy a NAT Gateway in a public subnet
- Create route tables:
  - Public Route Table → 0.0.0.0/0 → Internet Gateway
  - Private Route Table → 0.0.0.0/0 → NAT Gateway
- Associate route tables to the correct subnets

### 6.3 Deploy Frontend (Nginx on EC2)
- Launch an EC2 instance (Ubuntu) in a public subnet
- Assign a public IP
- Install Nginx:
  ```bash
  sudo apt update
  sudo apt install nginx -y
  ```
- Test Nginx page by accessing the public IP
- Configure Nginx to reverse proxy requests to the Flask app running inside ECS:

  **Example Nginx Configuration:**
  ```nginx
  server {
      listen 80;
      server_name your-domain.com;

      location / {
          proxy_pass http://PRIVATE-IP-OF-FLASK-APP:80;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_set_header X-Forwarded-Proto $scheme;
      }
  }
  ```

### 6.4 Deploy Backend (RDS MySQL Database)
- Create an RDS MySQL instance with Multi-AZ enabled
- Choose private subnets for RDS subnet group
- Create a security group to allow inbound MySQL traffic only from the App SG and Bastion SG
- Create a database called notes

### 6.5 Prepare Flask Application for Production
- Modify Dockerfile to use Gunicorn instead of flask run
- Update config.py to use the RDS Writer Endpoint

  **Config.py:**
  ```python
  import os
  basedir = os.path.abspath(os.path.dirname(__file__))
  class Config(object):
      SECRET_KEY = 'do-or-do-not-there-is-no-try'
      # You can uncomment the next line if you're using environment variables for your SECRET_KEY
      # SECRET_KEY = os.environ.get('SECRET_KEY') or 'do-or-do-not-there-is-no-try'
      SQLALCHEMY_DATABASE_URI = 'mysql+pymysql://admin:admin123@db-flask.cluster-cz3newuf9dtv.us-east-1.rds.amazonaws.com:3306/notes'
      SQLALCHEMY_TRACK_MODIFICATIONS = False
  ```

  **Dockerfile:**
  ```dockerfile
  # Use official Python image
  FROM python:3.9
  # Set the working directory inside the container
  WORKDIR /app
  # Copy the current directory contents into the container at /app
  COPY . .
  # Install system dependencies
  RUN apt-get update && apt-get install -y libpq-dev gcc
  # Install the Python dependencies
  RUN pip install --upgrade pip && pip install -r requirements.txt
  # Set Flask environment variables
  ENV FLASK_APP=crudapp.py
  ENV FLASK_RUN_HOST=0.0.0.0
  # Run database migrations
  RUN flask db init && flask db migrate -m 'entries table' && flask db upgrade
  # Expose port 80 for the application
  EXPOSE 80
  # Start Gunicorn with the Flask app (use Gunicorn for production)
  CMD ["gunicorn", "--bind", "0.0.0.0:80", "crudapp:app"]
  ```

  **Push to ECR:**
  ```bash
  docker build -t flask-crud-app .
  docker tag flask-crud-app:latest ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/your-repository
  docker push ACCOUNT_ID.dkr.ecr.REGION.amazonaws.com/your-repository
  ```

### 6.6 Deploy Flask App to ECS
- Create an ECS Cluster (Fargate Launch Type)
- Create a Task Definition pointing to the ECR image
- Create a Service to launch the task into private subnets
- Configure security groups to allow traffic only from Nginx EC2

### 6.7 Setup SSL Certificates
- SSH into the Nginx EC2 instance
- Install Certbot:
  ```bash
  sudo apt install certbot python3-certbot-nginx -y
  ```
- Generate SSL Certificate:
  ```bash
  sudo certbot --nginx -d your-domain.com
  ```
- Automatically redirect HTTP to HTTPS using Certbot options

### 6.8 Setup Bastion Host
- Deploy a Bastion EC2 instance in a public subnet
- Create a security group allowing SSH (port 22) from your IP only
- Install MySQL Client:
  ```bash
  sudo apt update
  sudo apt install mysql-client -y
  ```
- Connect to the RDS database:
  ```bash
  mysql -h your-rds-writer-endpoint.rds.amazonaws.com -u admin -p
  ```
- Verify the notes database and tables

### 6.9 Configure Load Balancer
- Create an Application Load Balancer
- Create a Target Group and add the ECS Service
- Attach the ALB to your ECS Service
- ALB listens on ports 80 and 443

![AWS Infrastructure](https://via.placeholder.com/800x400?text=AWS+Infrastructure+Diagram)

---

## 7. Technology Choices and Justifications

| Component | Choice | Reason |
|-----------|--------|--------|
| Frontend | Nginx on EC2 | Easy control and SSL termination |
| Middle | Flask app on ECS (Fargate) | Scalable and serverless |
| Backend | RDS MySQL Multi-AZ | High availability and reliability |
| Bastion | EC2 Instance | Secure remote DB access |
| Load Balancer | ALB | Scalable routing and SSL offloading |

---

## 8. Conclusion

Through this project, we designed and deployed a production-ready cloud architecture based on a three-tier model.

**Key achievements:**
- Secure, scalable deployment using AWS ECS and RDS
- Reverse proxy with HTTPS through Nginx and Let's Encrypt
- Secure database access via Bastion Host
- Ensured data persistence after ECS task restarts
- Implemented load balancing for fault tolerance

The application is now robust, scalable, secure, and cloud-native.

---

## 9. Screenshots

*[The following screenshots would be included in the final document]*

- VPC/Subnets overview
- Routing Tables
- Security Groups
- EC2 Nginx setup
- SSL Certificate (Certbot)
- RDS Database info
- ECS Task Definition
- ECS Service Health
- Bastion Host connected to RDS
- Load Balancer Target Group
- Flask App working over HTTPS
