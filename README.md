ShopSmart – Serverless E-Commerce Web Application (AWS Capstone Project)

A fully serverless e-commerce platform built with AWS Lambda (Python), API Gateway, DynamoDB, and a static website hosted on S3.
Designed to run 100% on AWS Free Tier and created as a capstone project for cloud engineering.

🚀 Project Overview

ShopSmart is a simple but fully functional e-commerce web application demonstrating cloud-native architecture using serverless technologies.

Users can:

View a list of products

Add products to a cart

View their shopping cart

Checkout and place an order

The backend is powered by AWS Lambda and DynamoDB, while the frontend is a lightweight HTML/JavaScript website hosted on Amazon S3.

🏗️ Architecture Diagram

(Optional – add your PNG here once you draw it in draw.io)
Example:

User Browser → S3 Static Website → API Gateway → Lambda (Python) → DynamoDB

🧰 Technologies Used
AWS Services

AWS Lambda (Python 3.10+)

Amazon API Gateway (HTTP API)

Amazon DynamoDB

Amazon S3 (Static Website Hosting)

IAM (Roles, Permissions)

CloudWatch (Logs)

Languages & Tools

Python

HTML / CSS / JavaScript

AWS Console

GitHub

📂 Project Structure
.
├── backend
│   └── lambda_function.py
├── frontend
│   ├── index.html
│   └── cart.html
├── architecture
│   └── shopsmart-architecture.png   (optional)
└── README.md

⚙️ Backend – AWS Lambda (Python)

All backend logic is handled by one Lambda function.
It supports:

Method	Path	Description
GET	/products	List all products
GET	/products/{id}	Get product details
GET	/cart	Get user cart
POST	/cart	Add item to cart
POST	/checkout	Checkout & create order
⭐ Lambda Environment Variables
PRODUCTS_TABLE=Products
CARTS_TABLE=Carts
ORDERS_TABLE=Orders

⭐ Lambda Code (backend/lambda_function.py)
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

🌐 Frontend – Amazon S3 Static Website
/frontend/index.html

(Displays products & adds to cart)

<script>
const API_BASE_URL = "YOUR_API_GATEWAY_URL";

/frontend/cart.html

(Views cart items & completes checkout)

<script>
const API_BASE_URL = "YOUR_API_GATEWAY_URL";

🛢️ DynamoDB Tables
Products
Key	Type
productId	String

Example item:

{
  "productId": "p1",
  "name": "Red Hoodie",
  "price": 39.99,
  "description": "Warm hoodie",
  "imageUrl": "https://via.placeholder.com/150"
}

Carts

userId (Partition key)

productId (Sort key)

Orders

orderId (Partition key)

🧪 Testing the App

Open the S3 static website endpoint

View the product list

Add items to the cart

Visit cart.html

Click Checkout

View new order in DynamoDB Orders table

📌 Key Learnings

How serverless architectures work

How Lambda + API Gateway + DynamoDB integrate

How to host static sites on S3

How to build real cloud apps without servers

How to stay within AWS Free Tier

How to structure and document cloud projects for recruiters

👩‍💻 Author

Amarachi Emeziem
Cloud Engineer | AWS | Python
GitHub: your link
LinkedIn: your link
