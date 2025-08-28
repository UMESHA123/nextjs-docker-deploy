
# 🚀 Deploy Next.js App on AWS ECS with Docker and ALB

This guide explains how to **containerize a Next.js app**, push it to **ECR**, and deploy it to **ECS Fargate** with an **Application Load Balancer (ALB)**.

---

## 📌 Prerequisites

* AWS Account
* IAM user with `ECS`, `ECR`, `EC2`, `IAM` permissions
* AWS CLI installed and configured:

  ```bash
  aws configure
  ```
* Docker installed
* GitHub repo for documentation

---

## 🛠️ 1. Create a Simple Next.js App

```bash
npx create-next-app myapp
cd myapp
npm run dev
```

Test locally:
👉 Open `http://localhost:3000`

---

## 📦 2. Dockerize Next.js

# Stage 1: Build the Next.js app
FROM --platform=linux/amd64 node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

# Stage 2: Run the production build
FROM --platform=linux/amd64 node:18-alpine AS runner

WORKDIR /app

COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/.next ./.next
COPY --from=builder /app/public ./public

EXPOSE 3000
CMD ["npm", "start"]


```bash
docker build -t my-next-app .
docker run -p 3000:3000 my-next-app
```

👉 Visit `http://localhost:3000`

---

## 🐳 3. Push Image to Amazon ECR

1. Create an ECR repo:

   ```bash
   aws ecr create-repository --repository-name my-next-app
   ```

2. Authenticate Docker with ECR:

   ```bash
   aws ecr get-login-password --region <your-region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<your-region>.amazonaws.com
   ```

3. Tag and push image:

   ```bash
   docker tag my-next-app:latest <account-id>.dkr.ecr.<your-region>.amazonaws.com/my-next-app:latest
   docker push <account-id>.dkr.ecr.<your-region>.amazonaws.com/my-next-app:latest
   ```

---

## ☁️ 4. Setup ECS Fargate Service

1. Go to **ECS Console → Create Cluster → Networking only (Fargate)**.
2. Create a **Task Definition**:

   * Launch type: **Fargate**
   * Add container:

     * Image: `<ECR repo URL>:latest`
     * Port mappings: `3000`
3. Create a **Service**:

   * Launch type: **Fargate**
   * Desired tasks: `1`
   * Attach **Application Load Balancer (ALB)**

---

## 🌐 5. Configure Application Load Balancer

* Create **Target Group**:

  * Target type: IP
  * Protocol: HTTP
  * Port: `3000`
  * Health check path: `/`
* Attach target group to **ECS service**
* Allow inbound rules in **ALB Security Group**:

  * HTTP `80` → `0.0.0.0/0`
  * (Optional) HTTPS `443`

---

## 🔑 6. IAM Permissions

ECS task needs an IAM role with:

* `AmazonECSTaskExecutionRolePolicy`
* `AmazonEC2ContainerRegistryReadOnly`

Attach role in **Task Execution Role**.

---

## 🌍 7. Access the App

1. Go to **EC2 → Load Balancers**.
2. Copy **DNS Name** of ALB:

   ```
   my-alb-1234567890.ap-south-1.elb.amazonaws.com
   ```
3. Open in browser → 🎉 Your Next.js app is live!

---

## ⚡ 8. Optional: Custom Domain (Route 53)

1. Create an **A Record** in Route 53:

   * Type: `Alias`
   * Target: ALB DNS
2. Access app at `https://yourdomain.com`

---

## 🛠️ 9. Common Issues

* **Health check failing** → make sure container listens on `0.0.0.0:3000`
* **CannotPullContainerError** → check platform in Docker build:

  ```bash
  docker buildx build --platform=linux/amd64 -t my-next-app .
  ```
* **Auth error with ECR** → ensure `AmazonEC2ContainerRegistryReadOnly` is attached

---

✅ That’s it! Your Next.js app is fully deployed on ECS with Docker + ALB.

