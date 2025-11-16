## **ShopSmart – Serverless E-Commerce Web Application (AWS Re/Start Capstone Project)**

A fully serverless e-commerce application built using AWS Lambda, API Gateway, DynamoDB, and S3, designed to run 100% on AWS Free Tier.

## **Overview**

ShopSmart is a cloud-native, serverless e-commerce web application designed to demonstrate real-world AWS architecture skills.
The system allows users to browse products, add items to a shopping cart, and complete checkout — all powered by AWS Lambda (Python), API Gateway, DynamoDB, and a static frontend hosted on S3.
This project highlights modern serverless patterns, least-privilege IAM design, and event-driven application workflows.

## **Objectives & Learning Outcomes**

By completing this project, we will learn:

How to design and deploy a serverless architecture on AWS

How to build a Python backend using AWS Lambda

How to expose APIs using API Gateway HTTP APIs

How to model and store application data using DynamoDB

How to host static websites using Amazon S3

How frontends call APIs in serverless applications

How IAM roles, permissions, and policies secure workloads

How to monitor serverless apps using CloudWatch Logs

How to explain a technical cloud project in interviews

## **Architecture**
<img width="1536" height="600" alt="be32f711-08e1-4055-a073-793eacb30927" src="https://github.com/user-attachments/assets/078fc5a8-1331-47d9-94fe-19208ae40483" />

### Architecture Summary:
User → S3 Static Website → API Gateway → Lambda (Python) → DynamoDB
The ShopSmart application is built using a fully serverless, AWS-native architecture designed to run 100% on the AWS Free Tier. It uses Amazon S3 for static web hosting, API Gateway for HTTP endpoints, AWS Lambda (Python) for backend logic, and DynamoDB for persistent storage. This architecture follows modern cloud engineering principles such as scalability, high availability, least-privilege IAM design, and fully managed services.


## **Technologies Used**

### Frontend
The frontend of this project is a lightweight static website built using HTML, CSS, and JavaScript, hosted on Amazon S3 Static Website Hosting. It includes a product listing page and a shopping cart page that communicate with the backend via API Gateway using secure HTTPS requests. The frontend is simple, fast, and fully serverless, making it ideal for AWS Free Tier deployments.


## **Commands & Steps**

```bash
# ======================================================
# Step 1: Create DynamoDB Tables
# ======================================================

# Table 1: Products
# Partition key: productId (String)

# Table 2: Carts
# Partition key: userId (String)
# Sort key: productId (String)

# Table 3: Orders
# Partition key: orderId (String)

# Add sample products manually via AWS Console (DynamoDB Items)
# Example item:
# {
#   "productId": "p1",
#   "name": "Red Hoodie",
#   "price": 39.99,
#   "description": "Warm hoodie",
#   "imageUrl": "https://via.placeholder.com/150"
# }

# ======================================================
# Step 2: Create an IAM Role for Lambda
# ======================================================

# Create role: ShopSmartLambdaRole
# Attach:
#   - AWSLambdaBasicExecutionRole
#   - AmazonDynamoDBFullAccess (training purpose)

# ======================================================
# Step 3: Create Lambda (Python 3.10)
# ======================================================

# Environment variables:
# PRODUCTS_TABLE=Products
# CARTS_TABLE=Carts
# ORDERS_TABLE=Orders

# Paste full lambda_function.py code into console editor
# (Included in "Backend Code" section below)

# ======================================================
# Step 4: Create API Gateway (HTTP API)
# ======================================================

# Routes:
# GET /products           -> Lambda
# GET /products/{id}      -> Lambda
# GET /cart               -> Lambda
# POST /cart              -> Lambda
# POST /checkout          -> Lambda

# Enable CORS:
# Allowed origins: *
# Allowed methods: GET, POST, OPTIONS
# Allowed headers: Content-Type

# Copy your API Invoke URL for frontend usage

# ======================================================
# Step 5: Build Frontend (index.html + cart.html)
# ======================================================

# Replace:
# const API_BASE_URL = "YOUR_API_GATEWAY_URL";

# Upload to S3:
# - index.html
# - cart.html

# Enable Static Website Hosting

# Go to Permissions → Block Public Access → Disable (for demo)
# Make index.html & cart.html public

# Access site using S3 Website Endpoint URL

# ======================================================
# Step 6: Test End-to-End
# ======================================================

# Open S3 website:
# - Load product list
# - Add to cart
# - View cart
# - Checkout
# - Verify "Orders" table in DynamoDB

# ======================================================
# Step 7: Cleanup (to avoid charges)
# ======================================================
# Delete S3 bucket, API Gateway, Lambda, DynamoDB tables

```

## **Backend Code (Lambda Function)**

Save as backend/lambda_function.py

```python
import json
import os
import boto3
from decimal import Decimal
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource("dynamodb")
products_table = dynamodb.Table(os.environ["PRODUCTS_TABLE"])
carts_table = dynamodb.Table(os.environ["CARTS_TABLE"])
orders_table = dynamodb.Table(os.environ["ORDERS_TABLE"])

class DecimalEncoder(json.JSONEncoder):
    def default(self, o):
        if isinstance(o, Decimal):
            return float(o)
        return super().default(o)

def make_response(status, body):
    return {
        "statusCode": status,
        "headers": {
            "Content-Type": "application/json",
            "Access-Control-Allow-Origin": "*"
        },
        "body": json.dumps(body, cls=DecimalEncoder)
    }

def lambda_handler(event, context):
    path = event.get("rawPath") or event.get("path", "")
    method = (event.get("requestContext", {}).get("http", {}) or {}).get("method") \
             or event.get("httpMethod", "")

    user_id = "demo-user"

    if method == "OPTIONS":
        return make_response(200, {"ok": True})

    if path == "/products" and method == "GET":
        data = products_table.scan(Limit=50)
        return make_response(200, data.get("Items", []))

    if path.startswith("/products/") and method == "GET":
        product_id = path.split("/")[-1]
        item = products_table.get_item(Key={"productId": product_id}).get("Item")
        if not item:
            return make_response(404, {"message": "Product not found"})
        return make_response(200, item)

    if path == "/cart" and method == "GET":
        data = carts_table.query(KeyConditionExpression=Key("userId").eq(user_id))
        return make_response(200, data.get("Items", []))

    if path == "/cart" and method == "POST":
        body = json.loads(event.get("body") or "{}")
        pid = body.get("productId")
        qty = int(body.get("qty", 1))
        if not pid:
            return make_response(400, {"message": "productId required"})
        carts_table.put_item(Item={"userId": user_id, "productId": pid, "qty": qty})
        return make_response(200, {"message": "Added to cart"})

    if path == "/checkout" and method == "POST":
        import time
        items = carts_table.query(KeyConditionExpression=Key("userId").eq(user_id)).get("Items", [])
        if not items:
            return make_response(400, {"message": "Cart empty"})

        total = 0
        details = []

        for line in items:
            pid = line["productId"]
            qty = int(line["qty"])
            prod = products_table.get_item(Key={"productId": pid}).get("Item")
            if prod:
                price = float(prod["price"])
                details.append({"productId": pid, "qty": qty, "price": price})
                total += price * qty

        order_id = f"ord_{int(time.time())}"

        orders_table.put_item(Item={
            "orderId": order_id,
            "userId": user_id,
            "items": details,
            "total": Decimal(str(total))
        })

        with carts_table.batch_writer() as batch:
            for it in items:
                batch.delete_item(Key={"userId": user_id, "productId": it["productId"]})

        return make_response(200, {"orderId": order_id, "total": total})

    return make_response(404, {"message": "Not found"})

```

## **Screenshots**

Products Table Screenshot
![Products Table](assets/screenshots/products-table.png)

Lambda Function Screenshot
![Lambda](assets/screenshots/lambda.png)

API Gateway Routes
![API Gateway](assets/screenshots/api-routes.png)

S3 Static Website Hosting
![S3 Hosting](assets/screenshots/s3-hosting.png)

Application UI – Products Page
![Products Page](assets/screenshots/products-ui.png)

Application UI – Cart Page
![Cart Page](assets/screenshots/cart-ui.png)


## **Tools Used**

AWS Lambda

Amazon API Gateway

DynamoDB

Amazon S3

IAM

CloudWatch

Python 3.10

HTML / CSS / JavaScript

GitHub

## **Key Takeaways**

Built a fully serverless application using S3, API Gateway, Lambda (Python), and DynamoDB.

Learned how to design and deploy modern cloud architectures using AWS best practices.

Implemented secure, scalable, and cost-efficient components that run fully on the AWS Free Tier.

Gained hands-on experience with NoSQL data modeling, API development, and frontend–backend integration.

Practiced AWS operational skills such as IAM permissions, CORS configuration, and CloudWatch monitoring.


## **What Actually Happened** 

Created three DynamoDB tables to store products, carts, and orders.

Wrote a Python Lambda backend to perform CRUD operations.

Created an API Gateway HTTP API with five routes.

Enabled CORS so the frontend can call the API.

Built a simple HTML/JS frontend to call the API Gateway.

Hosted frontend on Amazon S3 Static Website Hosting.

Connected everything end-to-end:

S3 frontend ↔ API Gateway ↔ Lambda ↔ DynamoDB

Verified data flow inside DynamoDB (products, cart items, orders).

Tested UI functionality (browse → add to cart → checkout).

Documented the entire project and added an architecture diagram + screenshots.



## **Authors:**

Hawi Jordan - Cloud Engineer

Olusegun Ajayi-Johnson - Cloud Engineer

Rory Mclean - Cloud Engineer

MD Shohel Khan (Sohel) - Cloud Engineer

Amarachi Emeziem - Cloud Engineer
