# 📜 Full-Stack Flask + MongoDB App on AWS ECS Fargate

This document explains how I built and deployed a full-stack Flask application using **MongoDB Atlas**, **AWS ECS Fargate**, and **ECR**. It includes full networking, IAM, scheduling, and security details — step-by-step — so even if you're a beginner, you can follow along and deploy it yourself.

---

## 🚀 Overview

This is a **Flask-based web app** with login, signup, profile creation (with images), and search functionality. Data is stored in **MongoDB Atlas**. The app is containerized and deployed to **ECS Fargate**, running in a **public subnet**, and exposed over the internet with access **restricted to my home/office Wi-Fi public IP**.

---

## 🧱 Tech Stack

| Layer          | Technology                    |
| -------------- | ----------------------------- |
| Frontend       | HTML, CSS, Bootstrap          |
| Backend        | Python + Flask                |
| Database       | MongoDB Atlas (Cloud)         |
| Image Store    | GridFS (within MongoDB Atlas) |
| Container      | AWS ECR                       |
| Orchestration  | AWS ECS (Fargate Launch Type) |
| Scheduling     | Amazon EventBridge            |
| Access Control | AWS Security Groups           |
| Deployment     | AWS Console & CLI             |

---

## 🔧 App Functionality

* ✅ Signup/Login form
* ✅ Upload name, age, height, gender, caste, religion, and image
* ✅ Store images using MongoDB's GridFS
* ✅ Search user profiles

---

## 🗓️ MongoDB Atlas Setup

### 1. **Create MongoDB Cluster**

* Visit [https://cloud.mongodb.com](https://cloud.mongodb.com)
* Create free/shared cluster in same AWS region as ECS (e.g., `ap-south-1`)

### 2. **Network Access**

* Go to `Network Access > IP Whitelist`
* Add your **ECS public IP** or CIDR (e.g., `YOUR_STATIC_IP/32`)

### 3. **Create Database User**

* Create username/password for app

### 4. **Connection URI**

Use this URI in Flask:

```python
MONGO_URI = "mongodb+srv://<username>:<password>@cluster0.mongodb.net/HMB?retryWrites=true&w=majority"
```

Store it in ECS environment variables.

---

## ☁️ Pushing Docker Image to ECR

### 1. **Authenticate Docker with ECR**

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com
```

### 2. **Tag & Push Image**

```bash
docker tag flask-mongo-app:latest <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com/flask-mongo-app

docker push <aws_account_id>.dkr.ecr.ap-south-1.amazonaws.com/flask-mongo-app
```

---

## ⚖️ AWS Services Used

| Service              | Purpose                           |
| -------------------- | --------------------------------- |
| **ECS Fargate**      | Run containerized Flask app       |
| **ECR**              | Store Docker images               |
| **VPC**              | Networking                        |
| **Public Subnet**    | Enable internet access            |
| **Internet Gateway** | Route internet traffic            |
| **Security Groups**  | Restrict access to fixed IP/port  |
| **IAM**              | Allow ECS to pull image, run task |
| **MongoDB Atlas**    | Cloud database                    |
| **EventBridge**      | Scheduled start/stop of ECS task  |

---

## 🌐 Network Architecture

**Visual Diagram**

```plaintext
[ Home/Office IP ]
        |
        v
[ Internet Gateway ]
        |
        v
[ Public Subnet (VPC) ]
        |
        v
[ ECS Fargate Task (Flask App) ]
        |
        v
[ MongoDB Atlas (Cloud DB) ]
```

---

## 🔒 Security Setup

### ECS Security Group:

| Type       | Protocol | Port | Source              |
| ---------- | -------- | ---- | ------------------- |
| Custom TCP | TCP      | 5000 | YOUR\_STATIC\_IP/32 |

---

## 🕜 ECS Setup: Cluster, Task, and Service

### Step 1: Create ECS Cluster

* Go to **Amazon ECS > Clusters > Create Cluster**
* Choose: **Networking only (Fargate)**
* Assign VPC and Subnet

### Step 2: Create Task Definition

* Go to **Task Definitions > Create new**
* Launch Type: **Fargate**
* Add container:

  * Image URI: `<your ECR image>`
  * Port: 5000
* Set environment variables (Mongo URI, etc.)
* Assign task role (for logging/permissions)

### Step 3: Run Task

* Go to your cluster
* Run task with your task definition
* Choose the same public subnet and security group
* Enable **Auto-assign public IP**

---

## 🌏 VPC Peering (Optional for Private Subnet)

### What is VPC Peering?

It creates a **private tunnel** between your AWS VPC and MongoDB Atlas VPC so no internet access is needed.

### Steps to Set Up VPC Peering:

1. Go to MongoDB Atlas → Network Access → VPC Peering
2. Click **New Peering Connection**
3. Enter:

   * AWS Account ID
   * VPC ID
   * CIDR range
   * VPC Region
4. Accept the peering request in the AWS VPC console
5. Update route tables to route Atlas traffic through the peered connection
6. Add Atlas private IP/CIDR in ECS task security group or whitelist in Atlas

---

## 🕒 Task Scheduling with EventBridge

You used **Amazon EventBridge** to automate task start/stop like a `cron` job.

### Schedule:

| Action     | Time (IST) | UTC Cron Expression   |
| ---------- | ---------- | --------------------- |
| Start Task | 10:00 AM   | `cron(30 4 * * ? *)`  |
| Stop Task  | 12:00 PM   | via Lambda or timeout |
| Start Task | 4:00 PM    | `cron(30 10 * * ? *)` |
| Stop Task  | 6:00 PM    | via Lambda or timeout |

> Note: ECS does not natively support auto-stopping, so I wrote a **Lambda** function that stops running tasks using `ecs:stopTask()`.

---

## 🧠 Lambda to Stop Tasks

### IAM Policy:

```json
{
  "Action": ["ecs:ListTasks", "ecs:StopTask"],
  "Effect": "Allow",
  "Resource": "*"
}
```

### Python Code (Lambda):

```python
import boto3

def lambda_handler(event, context):
    ecs = boto3.client('ecs')
    tasks = ecs.list_tasks(cluster='YOUR_CLUSTER_NAME', desiredStatus='RUNNING')
    for task in tasks['taskArns']:
        ecs.stop_task(cluster='YOUR_CLUSTER_NAME', task=task, reason='Scheduled Stop')
```

---

## ✅ Final Testing

* ✔️ Access app from browser via public IP
* ✔️ Confirm restricted access outside your IP
* ✔️ Upload profile with image
* ✔️ Search and fetch MongoDB-stored profiles
* ✔️ App starts/stops on schedule

---

## 🗳️ What You’ll Need

* ✅ AWS account
* ✅ MongoDB Atlas account
* ✅ ECR repository
* ✅ Your static public IP address (home/office)
* ✅ Some patience and curiosity 😉

---

## 📈 Estimated Monthly Cost

> AWS ECS Fargate pricing is based on requested vCPU and memory.

### ⌚ Runtime Assumptions:

* **vCPU**: 0.25
* **Memory**: 0.5 GB
* **Region**: Asia Pacific (Mumbai)
* **Pricing** (as of July 2025):

  * \$0.04048 per vCPU-hour
  * \$0.004445 per GB-hour

### 📅 Cost Estimates:

| Mode         | Hours/Day | Monthly vCPU-Hours | Monthly GB-Hours | Approx Cost (USD) | Approx Cost (INR @ ₹82.0/USD) |
| ------------ | --------- | ------------------ | ---------------- | ----------------- | ----------------------------- |
| 4 hours/day  | 120       | 30                 | 60               | \~\$1.55          | \~₹130                        |
| 24 hours/day | 720       | 180                | 360              | \~\$9.30          | \~₹763                        |

> Note: MongoDB Atlas (free-tier/shared cluster) is **free**.
> EventBridge & Lambda are **mostly free** within AWS Free Tier limits.

---

## 📦 Future Improvements

* 🔐 Add HTTPS using ALB + ACM
* 🌍 Map to custom domain via Route 53
* 🔍 Add sharing & profile download features
* 🔒 Improve security (use Secrets Manager, WAF)
* 🚀 Migrate app to **private subnet** and use **VPC peering**
* 📊 Add frontend JS framework (React, Vue)
* 📊 Enable metrics/alerts with CloudWatch

---

## 🙌 Conclusion

This project helped me learn how to:

* Build a full-stack Flask app with MongoDB Atlas
* Push images to ECR
* Deploy securely using ECS Fargate
* Restrict access by network
* Connect Atlas with/without VPC peering
* Schedule app availability with EventBridge

You can absolutely replicate this. Just follow the diagrams and steps above, and feel free to tweak it for your own use case. Let’s build more cool things! 💪

