# ShopEase: Serverless E-Commerce Website on AWS

ShopEase is an online store I built and deployed on AWS without running a single server. Shoppers can browse products by category, search, add items to a cart and place an order. Every order is passed through a queue and saved in a DynamoDB table.

Built as part of the **AWS re/Start** program with **ICT Academy**.

## Architecture

```
Customer's browser
      |
      v
Amazon S3 (static website: index.html)
      |  POST /orders
      v
Amazon API Gateway (REST API: ProductOrdersAPI)
      |
      v
AWS Lambda: SubmitOrderFunction
      |  sends the order as a message
      v
Amazon SQS (ProductOrdersQueue)
      |  triggers
      v
AWS Lambda: ProcessOrderFunction
      |  put_item
      v
Amazon DynamoDB (ProductOrders table)
```

Both Lambda functions write their logs to Amazon CloudWatch Logs.

## AWS services used

| Service | What it does here |
| --- | --- |
| Amazon S3 | Hosts the store's frontend as a static website |
| Amazon API Gateway | Receives the order from the website (POST /orders) |
| AWS Lambda | Runs the order logic, no servers to manage |
| Amazon SQS | Queues each order so none are lost if traffic spikes |
| Amazon DynamoDB | Stores each order (order ID, product, quantity, date) |
| AWS IAM | Execution roles that give each Lambda only the access it needs |
| Amazon CloudWatch | Logs for testing and debugging |

## Folder structure

```
shopease-serverless-ecommerce/
├── frontend/
│   └── index.html                    store page (HTML, CSS and JavaScript in one file)
├── lambda/
│   ├── SubmitOrderFunction/
│   │   └── lambda_function.py        API Gateway -> SQS
│   └── ProcessOrderFunction/
│       └── lambda_function.py        SQS -> DynamoDB
├── aws-config/
│   ├── bucket-policy.json            public read policy for the S3 website bucket
│   └── test-event.json               Lambda console test event
└── README.md
```

## How to deploy it

All resources go in one Region (I used US East, N. Virginia).

1. **DynamoDB**: create a table named `ProductOrders` with partition key `orderId` (String).
2. **SQS**: create a Standard queue named `ProductOrdersQueue` and copy its URL.
3. **SubmitOrderFunction** (Python 3.12): paste `lambda/SubmitOrderFunction/lambda_function.py`, put your queue URL in `queue_url`, deploy. Attach `AmazonSQSFullAccess` to its role.
4. **ProcessOrderFunction** (Python 3.12): paste `lambda/ProcessOrderFunction/lambda_function.py`, deploy. Attach `AmazonSQSFullAccess` and `AmazonDynamoDBFullAccess` to its role.
5. **SQS trigger**: in the queue's Lambda triggers tab, add `ProcessOrderFunction`.
6. **API Gateway**: create a REST API `ProductOrdersAPI`, a resource `/orders` with CORS enabled, and a `POST` method with Lambda proxy integration to `SubmitOrderFunction`. Enable CORS on `/orders`, then deploy to a stage named `prod`.
7. **Frontend**: in `frontend/index.html`, set `API_URL` to your invoke URL followed by `/orders`.
8. **S3 website**: create a bucket, turn off Block all public access, enable static website hosting with `index.html`, add `aws-config/bucket-policy.json` (with your bucket name) and upload `index.html`.
9. Open the bucket website endpoint, place an order and check the item in the `ProductOrders` table.

To test the backend on its own, run `SubmitOrderFunction` in the Lambda console with `aws-config/test-event.json`.

## Problems I ran into

- **"The specified queue does not exist"**: my Lambda and my SQS queue were in different Regions. Everything has to be in the same Region.
- **"Succeeded" but statusCode 400**: the Lambda caught the error and returned it as a normal response, so the Details output has to be checked, not just the green banner.
- **CloudShell unavailable** on a new account while verification was in progress, so I tested with console test events instead.

## What I learned

- A queue between the API and the database keeps the app reliable when traffic is uneven.
- Serverless means paying only when the code runs.
- Most of the real learning came from debugging how the services connect: Regions, IAM permissions and CORS.
