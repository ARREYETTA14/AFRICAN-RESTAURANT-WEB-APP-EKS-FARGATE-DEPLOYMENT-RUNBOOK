# 🍽️ AFRICAN RESTAURANT WEB APP — EKS + FARGATE DEPLOYMENT RUNBOOK

```plaintext
                                                        🌐 USER BROWSER
                                                               |
                                                               v
                                                 +---------------------------+
                                                 |   AWS LOAD BALANCER       |
                                                 |   (LoadBalancer Service)  |
                                                 +---------------------------+
                                                               |
                                                               v
                                                 +---------------------------+
                                                 |   FRONTEND POD (NGINX)    |
                                                 |   Fargate Serverless      |
                                                 +---------------------------+
                                                               |
                                                               v
                                                 +---------------------------+
                                                 |   BACKEND POD (FLASK)     |
                                                 |   Fargate Serverless      |
                                                 +---------------------------+
                                                               |
                                                               v
                                                 +---------------------------+
                                                 |   DATABASE POD (MYSQL)    |
                                                 |   Fargate Serverless      |
                                                 +---------------------------+
                                
                                Static Assets:
                                                 AWS S3 BUCKET
```

## STEP 1: CREATE EC2 INSTANCE

- This is your workstation where you run your commands and deploy your files. It acts as a Linux laptop in AWS.

Launch:
- Amazon Linux 2023
- Instance Type: t3.medium (recommended)
- On Sg, Open Ports:
    - 22 - SSH
- SSH into the instance

## STEP 2: INSTALL REQUIRED TOOLS ON YOUR EC2 MACHINE

**Install AWS CLI**

```bash
# install aws cli
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# verify
aws --version
```

**Configure AWS CLI**

```bash
aws configure
```

You will be prompted to enter:

```plaintext
AWS Access Key ID:     <your-access-key>
AWS Secret Access Key: <your-secret-key>
Default region name:   sa-east-1
Default output format: json
```

**Install kubectl**

```bash
# download kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# make executable
chmod +x kubectl

# move to system path
sudo mv kubectl /usr/local/bin/

# verify
kubectl version --client
```

**Install eksctl**
```bash
# download eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# move to system path
sudo mv /tmp/eksctl /usr/local/bin

# verify
eksctl version
```

**Install Docker**

```bash
sudo dnf update -y # update server
sudo dnf install -y docker # install docker
ls -la /usr/libexec/docker/ # check docker location
sudo chmod +x /usr/libexec/docker/docker-setup-runtimes.sh # give execution permissions to my shell script
sudo systemctl start docker # start docker
sudo systemctl enable docker # enables docker 
sudo systemctl status docker # check if docker is running
sudo usermod -aG docker ec2-user # add ec2-user to the docker Group
sudo reboot # reboot server to effect changes
```

## STEP 3: CREATE PROJECT STRUCTURE

```bash
mkdir restaurant-app
cd restaurant-app
mkdir frontend backend db k8s
```

Final structure:
```plaintext
restaurant-app/
│
├── frontend/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── index.html
│   └── styles.css
│
├── backend/
│   ├── Dockerfile
│   ├── app.py
│   └── requirements.txt
│
├── db/
│   └── init.sql
│
└── k8s/
    ├── secret.yml
    ├── configmap.yml
    ├── mysql-initdb-configmap.yml   ← add this
    ├── mysql-deployment.yml
    ├── mysql-service.yml
    ├── backend-deployment.yml
    ├── backend-service.yml
    ├── frontend-deployment.yml
    └── frontend-service.yml

```

## STEP 4: CREATE PROJECT FILES

- ```frontend/Dockerfile```

```dockerfile
FROM nginx:latest

COPY index.html /usr/share/nginx/html/
COPY styles.css /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
```

- ```frontend/nginx.conf```

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        index index.html;
    }

    location /order {
        proxy_pass http://backend-service:5000/order;
        proxy_set_header Content-Type application/json;
    }
}
```

- ```frontend/index.html```

```index.html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>🍛 African Restaurant</title>
  <link rel="stylesheet" href="styles.css">
</head>
<body>

  <div class="hero">
    <h1>African Restaurant</h1>
    <p class="subtitle">Taste the richness of Africa, one dish at a time</p>
  </div>

  <div class="card">
    <label for="food">Choose your dish:</label>
    <select id="food">
      <option value="">-- Select a dish --</option>
      <option value="Eru">Eru</option>
      <option value="Egusi Soup">Egusi Soup</option>
      <option value="Bunny Chow">Bunny Chow</option>
    </select>

    <div id="imageWrapper">
      <img id="foodImage" alt="Food Preview" />
    </div>

    <button id="orderBtn" onclick="placeOrder()">Place Order</button>
    <div id="message"></div>
  </div>

  <script>
    const images = {
      "Eru": "https://YOUR-S3-BUCKET.s3.amazonaws.com/images/eru.jpg",
      "Egusi Soup": "https://YOUR-S3-BUCKET.s3.amazonaws.com/images/egusi.jpg",
      "Bunny Chow": "https://YOUR-S3-BUCKET.s3.amazonaws.com/images/bunny.jpg"
    };

    document.getElementById("food").addEventListener("change", function () {
      const food = this.value;
      const img = document.getElementById("foodImage");
      const wrapper = document.getElementById("imageWrapper");
      if (images[food]) {
        img.src = images[food];
        wrapper.style.display = "block";
      } else {
        wrapper.style.display = "none";
      }
    });

    async function placeOrder() {
      const food = document.getElementById("food").value;
      const msg = document.getElementById("message");
      const btn = document.getElementById("orderBtn");

      if (!food) {
        msg.className = "error";
        msg.innerText = "⚠️ Please select a dish first.";
        return;
      }

      btn.disabled = true;
      btn.innerText = "Placing order...";

      try {
        const res = await fetch("/order", {
          method: "POST",
          headers: { "Content-Type": "application/json" },
          body: JSON.stringify({ food })
        });
        const data = await res.json();
        msg.className = "success";
        msg.innerText = "✅ " + data.message;
      } catch (err) {
        msg.className = "error";
        msg.innerText = "❌ Failed to place order. Please try again.";
      } finally {
        btn.disabled = false;
        btn.innerText = "Place Order";
      }
    }
  </script>

</body>
</html>
```

- ```frontend/styles.css```

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Arial', sans-serif;
    background: linear-gradient(135deg, #1a0a00, #3d1a00);
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 40px 20px;
}

.hero {
    text-align: center;
    margin-bottom: 30px;
}

.hero h1 {
    font-size: 2.8rem;
    color: #ffb347;
    text-shadow: 2px 2px 8px rgba(0,0,0,0.5);
}

.subtitle {
    color: #ffd9a0;
    font-size: 1rem;
    margin-top: 8px;
    font-style: italic;
}

.card {
    background: rgba(255, 255, 255, 0.05);
    backdrop-filter: blur(10px);
    border: 1px solid rgba(255,179,71,0.2);
    border-radius: 20px;
    padding: 40px;
    width: 100%;
    max-width: 480px;
    text-align: center;
    box-shadow: 0 8px 32px rgba(0,0,0,0.4);
}

label {
    display: block;
    color: #ffd9a0;
    font-size: 1rem;
    margin-bottom: 10px;
}

select {
    width: 100%;
    padding: 12px 16px;
    border-radius: 10px;
    border: 1px solid #ffb347;
    background: #1a0a00;
    color: #ffd9a0;
    font-size: 1rem;
    cursor: pointer;
    outline: none;
}

select:focus {
    border-color: #ff8c00;
    box-shadow: 0 0 0 3px rgba(255,140,0,0.2);
}

#imageWrapper {
    display: none;
    margin: 20px 0;
}

#foodImage {
    width: 100%;
    max-width: 380px;
    border-radius: 14px;
    box-shadow: 0 6px 20px rgba(0,0,0,0.5);
    transition: transform 0.3s ease;
}

#foodImage:hover {
    transform: scale(1.03);
}

button {
    margin-top: 20px;
    width: 100%;
    padding: 14px;
    background: linear-gradient(135deg, #cc5500, #ff8c00);
    color: white;
    font-size: 1.1rem;
    font-weight: bold;
    border: none;
    border-radius: 10px;
    cursor: pointer;
    transition: opacity 0.2s, transform 0.2s;
}

button:hover:not(:disabled) {
    opacity: 0.9;
    transform: translateY(-2px);
}

button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

#message {
    margin-top: 20px;
    padding: 12px 16px;
    border-radius: 10px;
    font-size: 1rem;
    font-weight: bold;
    display: block;
}

#message:empty {
    display: none;
}

.success {
    background: rgba(0, 200, 100, 0.15);
    color: #00e676;
    border: 1px solid rgba(0,200,100,0.3);
}

.error {
    background: rgba(255, 50, 50, 0.15);
    color: #ff5252;
    border: 1px solid rgba(255,50,50,0.3);
}
```

- ```backend/Dockerfile```

```dockerfile
FROM python:3.10-slim

WORKDIR /app

COPY . .

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 5000

CMD ["python", "app.py"]
```

- ```backend/requirements.txt```

```plaintext
flask
mysql-connector-python
```

- ```backend/app.py```

```python
from flask import Flask, request, jsonify
import mysql.connector
import os

app = Flask(__name__)

@app.route('/order', methods=['POST'])
def order():
    data = request.get_json()
    food = data['food']

    db = mysql.connector.connect(
        host=os.environ['DB_HOST'],
        user=os.environ['DB_USER'],
        password=os.environ['DB_PASSWORD'],
        database=os.environ['DB_NAME']
    )

    cursor = db.cursor()
    cursor.execute("INSERT INTO orders (food) VALUES (%s)", (food,))
    db.commit()
    cursor.close()
    db.close()

    return jsonify({"message": f"Order received for {food}"})

if __name__ == '__main__':
    app.run(host="0.0.0.0", port=5000)
```

- ```db/init.sql```

```sql
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    food VARCHAR(255)
);
```

## STEP 5: SETUP AWS S3 (STATIC ASSETS)

```S3 must be set up BEFORE building images so the correct URLs get baked into index.html```

Create S3 Bucket

- Go to AWS Console → S3 → Create Bucket

- Bucket name: african-restaurant-assetss

- Region: sa-east-1

- Uncheck Block Public Access

- Click Create Bucket

Add Bucket Policy

Go to your bucket → Permissions → Bucket Policy and paste:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::african-restaurant-assetss/*"
        }
    ]
}
```

- Upload Images

- Get Image URLs

- Update index.html with Real S3 URLs


## STEP 6: CREATE ECR REPOSITORIES (TO STORE YOUR DOCKER IMAGES)

ECR (Elastic Container Registry) is AWS's Docker image storage service. We need to push our frontend and backend images there.

```awscli
# create frontend repository
aws ecr create-repository --repository-name restaurant-frontend --region sa-east-1

# create backend repository
aws ecr create-repository --repository-name restaurant-backend --region sa-east-1
```

**NB:** Make sure to change the ```region``` to your desired region

You will see output like this, note your account ID:

```json
{
    "repository": {
        "repositoryUri": "123456789.dkr.ecr.sa-east-1.amazonaws.com/restaurant-frontend"
    }
}
```

## STEP 7: BUILD AND PUSH IMAGES TO ECR

Authenticate Docker to ECR:

```bash
aws ecr get-login-password --region sa-east-1 | docker login --username AWS --password-stdin <your-account-id>.dkr.ecr.sa-east-1.amazonaws.com
```
``` Edit the command and add your account_id```

Build and Push Frontend Image

```bash
cd ~/restaurant-app/frontend

docker build -t restaurant-frontend .

docker tag restaurant-frontend:latest <your-account-id>.dkr.ecr.sa-east-1.amazonaws.com/restaurant-frontend:latest

docker push <your-account-id>.dkr.ecr.sa-east-1.amazonaws.com/restaurant-frontend:latest
```

Build and Push Backend Image

```bash
cd ~/restaurant-app/backend

docker build -t restaurant-backend .

docker tag restaurant-backend:latest <your-account-id>.dkr.ecr.sa-east-1.amazonaws.com/restaurant-backend:latest

docker push <your-account-id>.dkr.ecr.sa-east-1.amazonaws.com/restaurant-backend:latest
```

Verify Images in ECR

```bash
aws ecr list-images --repository-name restaurant-frontend --region sa-east-1
aws ecr list-images --repository-name restaurant-backend --region sa-east-1
```

## STEP 8: CREATE EKS CLUSTER WITH FARGATE

```bash
eksctl create cluster \
  --name restaurant-cluster \
  --region sa-east-1 \
  --fargate
```

**Make sure to change the region and name as per your need**

This will take about 15-20 minutes. It automatically creates:

- EKS Control Plane ✅

- VPC and Subnets ✅

- Fargate Profile ✅

- IAM Roles ✅

After that, verify Cluster is Running:

```bash
kubectl get nodes
```
You should see:

```plaintext
NAME                                STATUS   ROLES
fargate-ip-xxx.compute.internal     Ready    <none>
```

**IMPORTANT:** Tag public subnets so the Load Balancer can find them.
Only tag subnets that meet BOTH conditions:
1. CIDR starts with 192.168.x.x (EKS VPC, not default VPC)
2. Public column shows True
Ignore all 172.31.x.x subnets as they belong to the default VPC


```bash
# find your public subnets
aws ec2 describe-subnets \
  --region sa-east-1 \
  --query 'Subnets[*].{ID:SubnetId,Public:MapPublicIpOnLaunch,CIDR:CidrBlock}' \
  --output table

# tag the subnets that show Public = True and have 192.168.x.x range
aws ec2 create-tags \
  --resources <public-subnet-id-1> <public-subnet-id-2> <public-subnet-id-3> \
  --tags Key=kubernetes.io/role/elb,Value=1 \
  --region sa-east-1
```

## STEP 9: INSTALL AWS LOAD BALANCER CONTROLLER

The AWS Load Balancer Controller is a Kubernetes controller that manages AWS Load Balancers for your cluster.

Why it is needed:
Without it your frontend-service of type LoadBalancer will stay in ```<pending>``` forever on EKS Fargate because the default Kubernetes controller creates Load Balancers with "instance" target type which cannot find Fargate pods since Fargate pods are serverless and not EC2 instances. The AWS Load Balancer Controller fixes this by creating Load Balancers with "ip" target type which targets pods directly by their IP address.


It also enables these important annotations in your ```frontend-service.yml```:

```plaintext
aws-load-balancer-type: "external"        → use AWS Load Balancer Controller
aws-load-balancer-nlb-target-type: "ip"  → target pods by IP not instance
aws-load-balancer-scheme: "internet-facing" → accessible from internet
```
```bash
# 1. download latest iam policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# 2. create iam policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 3. associate oidc provider with your cluster
eksctl utils associate-iam-oidc-provider \
  --cluster restaurant-cluster \
  --region sa-east-1 \
  --approve

# 4. create iam service account
eksctl create iamserviceaccount \
  --cluster restaurant-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::<your-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region sa-east-1

# 5. Get VPC_id to subtitude at the level of installing aws lb controller
# get your vpc id and copy it
aws eks describe-cluster \
  --name restaurant-cluster \
  --region sa-east-1 \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text

# 6. install helm
curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash

# 7. add eks helm repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# 8. install aws load balancer controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=restaurant-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=sa-east-1 \
  --set vpcId=<your-vpc-id>

# 9. verify controller is running
kubectl get pods -n kube-system
```

## STEP 10: CREATE FARGATE PROFILE FOR YOUR APPLICATION

A Fargate Profile tells eks:
"Hey, any Pod that belongs to the namespace called 
restaurant should run on Fargate, not on EC2".

```bash
eksctl create fargateprofile \
  --cluster restaurant-cluster \
  --name restaurant-profile \
  --namespace restaurant \
  --region sa-east-1
```
**Change region as per your desired working region**

Create the Namespace:

```bash
kubectl create namespace restaurant
```

## STEP 11: CREATE KUBERNETES MANIFEST FILES

Create a directory for the K8s files:

```bash
cd ~/restaurant-app/k8s
```

### Create the files

- ```secret.yml```

Generate the base64 encoded values first:

```bash
echo -n "root" | base64          # cm9vdA==
echo -n "restaurant" | base64    # cmVzdGF1cmFudA==
echo -n "orders" | base64        # b3JkZXJz
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: restaurant
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: cmVzdGF1cmFudA==
  MYSQL_DATABASE: b3JkZXJz
  DB_USER: cm9vdA==
  DB_PASSWORD: cmVzdGF1cmFudA==
```

- ```configmap.yml```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: restaurant
data:
  DB_HOST: mysql-service
  DB_NAME: orders
  APP_PORT: "5000"
```

- ```mysql-initdb-configmap.yml```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql-initdb-config
  namespace: restaurant
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS orders (
        id INT AUTO_INCREMENT PRIMARY KEY,
        food VARCHAR(255)
    );
```

**NB:**  NOTE: The mysql-initdb-configmap.yml replaces the need to
manually create the orders table. It automatically mounts the
init.sql script into the MySQL container at startup which is
the Kubernetes equivalent of what Docker Compose did with:

volumes:
  - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql

- ```mysql-deployment.yml```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-deployment
  namespace: restaurant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:5.7
        ports:
        - containerPort: 3306
        envFrom:
        - secretRef:
            name: mysql-secret
        - configMapRef:
            name: app-config
        volumeMounts:
        - name: initdb
          mountPath: /docker-entrypoint-initdb.d
      volumes:
      - name: initdb
        configMap:
          name: mysql-initdb-config
```

- ```mysql-service.yml```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
  namespace: restaurant
spec:
  selector:
    app: mysql
  ports:
  - protocol: TCP
    port: 3306
    targetPort: 3306
  type: ClusterIP
```

- ```backend-deployment.yml```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deployment
  namespace: restaurant
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: 932877337538.dkr.ecr.sa-east-1.amazonaws.com/restaurant-backend:latest
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: app-config
        - secretRef:
            name: mysql-secret
```

Replace ```<your-account-id>``` with the actual ID of your image in ECR.

- ```backend-service.yml```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: restaurant
spec:
  selector:
    app: backend
  ports:
  - protocol: TCP
    port: 5000
    targetPort: 5000
  type: ClusterIP
```

- ```frontend-deployment.yml```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  namespace: restaurant
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: 932877337538.dkr.ecr.sa-east-1.amazonaws.com/restaurant-frontend:latest
        ports:
        - containerPort: 80
```

Replace ```<your-account-id>``` and ```region``` with the actual ID of your image in ECR and region respectively


- ```frontend-service.yml```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: restaurant
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "external"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
spec:
  selector:
    app: frontend
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer

```

## STEP 12: DEPLOY EVERYTHING TO EKS

Deploy in this exact sequence:

```bash
# 1. deploy secret first
kubectl apply -f secret.yml

# 2. deploy configmap
kubectl apply -f configmap.yml

# 3. deploy mysql initdb configmap      ← add this
kubectl apply -f mysql-initdb-configmap.yml

# 4. deploy mysql
kubectl apply -f mysql-deployment.yml
kubectl apply -f mysql-service.yml

# 5. wait for mysql to be ready before deploying backend
kubectl get pods -n restaurant
# wait until mysql pod shows STATUS = Running

# 6. deploy backend
kubectl apply -f backend-deployment.yml
kubectl apply -f backend-service.yml

# 7. deploy frontend
kubectl apply -f frontend-deployment.yml
kubectl apply -f frontend-service.yml
```

## STEP 13: VERIFY EVERYTHING IS RUNNING

```bash 
# check all pods are running
kubectl get pods -n restaurant

# check all services
kubectl get services -n restaurant

# check deployments
kubectl get deployments -n restaurant
```

Expected output for pods:

```bash
NAME                                   READY   STATUS    
mysql-deployment-xxx                   1/1     Running   
backend-deployment-xxx                 1/1     Running   
backend-deployment-xxx                 1/1     Running   
frontend-deployment-xxx                1/1     Running   
frontend-deployment-xxx                1/1     Running   
```

## STEP 14: ACCESS YOUR APPLICATION

Get the Load Balancer URL

```bash
kubectl get services -n restaurant
```

Look for the frontend-service EXTERNAL-IP:

```plaintext
NAME                TYPE           EXTERNAL-IP
frontend-service    LoadBalancer   xxxx.sa-east-1.elb.amazonaws.com
```

Open in Browser

```plaintext
http://xxxx.sa-east-1.elb.amazonaws.com
```

**NB:** It may take 2-5 minutes for the Load Balancer URL to become active after deployment

## STEP 15: VERIFY DATABASE

```bash
# get mysql pod name
kubectl get pods -n restaurant

# access mysql pod
kubectl exec -it <mysql-pod-name> -n restaurant -- mysql -uroot -prestaurant

# inside mysql shell
USE orders;
SELECT * FROM orders;
```

## STEP 16: USEFUL TROUBLESHOOTING COMMANDS

```bash
# check pod logs
kubectl logs <pod-name> -n restaurant

# describe a pod to see errors
kubectl describe pod <pod-name> -n restaurant

# check all resources in namespace
kubectl get all -n restaurant

# delete and redeploy everything
kubectl delete namespace restaurant
kubectl create namespace restaurant
```

## STEP 17: CLEANUP (TO AVOID AWS CHARGES)

When you are done testing, delete everything to avoid charges:

```plaintext
# delete all k8s resources
kubectl delete namespace restaurant

# delete eks cluster - This takes about 10-15 minutes. Wait for it to complete before proceeding.
eksctl delete cluster --name restaurant-cluster --region sa-east-1

# delete ecr repository
aws ecr delete-repository --repository-name restaurant-frontend --force --region sa-east-1
aws ecr delete-repository --repository-name restaurant-backend --force --region sa-east-1

# delete s3 bucket and all images inside it
aws s3 rb s3://african-restaurant-assetss --force

## verify everything is deleted:

# verify cluster is gone
eksctl get cluster --region sa-east-1

# verify ecr repos are gone
aws ecr describe-repositories --region sa-east-1

# verify s3 bucket is gone
aws s3 ls# 
```

**SUMMARY**

```plaintext
SEQUENCE OF EVENTS:

1.  Install tools (AWS CLI, kubectl, eksctl, Docker)
2.  Create project files (index.html, app.py, Dockerfiles etc)
3.  Setup S3 and upload images → update index.html with real URLs
4.  Create ECR repositories
5.  Build and push images to ECR
6.  Create EKS cluster with Fargate
7.  Tag public subnets for Load Balancer
8.  Install AWS Load Balancer Controller
9.  Create Fargate profile and namespace
10. Create manifest files (secret, configmap, deployments, services)
11. Deploy to EKS in correct sequence
12. Access via Load Balancer URL


CONNECTIONS:
S3 Bucket (images)
    ↓ URLs baked into index.html
ECR (frontend + backend images)
    ↓ pulled by EKS
EKS Cluster (restaurant-cluster)
└── Fargate Profile (restaurant-profile)
    └── Namespace (restaurant)
        ├── Secret → passwords
        ├── ConfigMap → DB_HOST, APP_PORT
        ├── MySQL Deployment → mysql-service (ClusterIP:3306)
        ├── Backend Deployment → backend-service (ClusterIP:5000)
        └── Frontend Deployment → frontend-service (LoadBalancer:80)
                                          ↓
                                  AWS Load Balancer
                                          ↓
                                    User Browser

```


