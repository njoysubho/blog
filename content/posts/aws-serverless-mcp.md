---
title: "Exploring the AWS Serverless MCP Server"
date: 2025-06-09T11:42:14+02:00
draft: false
---

AWS has recently introduced the MCP server for AWS Serverless. You can read the official announcement [here](https://aws.amazon.com/blogs/compute/introducing-aws-serverless-mcp-server-ai-powered-development-for-modern-applications/).

I’ve been testing it using the latest MCP server with continue.dev plugin and Claude Sonnet 4 as the LLM backend. While the MCP server bundles a variety of tools, I want to focus on the three that stood out the most in my usage.

🔍 1. `get_lambda_guidance`-This tool helps determine if a given use case is suitable for AWS Lambda.

I tested it with the prompt:
```
I want to make a document summarization application using LLM. Is Lambda a good choice?
```
The Agent used the MCP server correctly and triggered the get_lambda_guidance tool. The response proposed an asynchronous S3-based approach, explaining that a synchronous design might suffer from large document sizes and Lambda’s execution timeout limits. This kind of guidance is genuinely helpful, especially for developers exploring architectural trade-offs.

🧩 2. `get_serverless_templates` – This tool pulls the best-matching pattern from the Serverless Land GitHub repository. It assumes a SAM-based application, but the architectural patterns it suggests are still valuable even if you're using CDK or Terraform.

Here’s what I asked:
```
What is the serverless pattern for this kind of application if I use external LLMs?
```
The tool returned this architecture:
```
Upload: User uploads document via API Gateway

Queue: Lambda pushes job into SQS and returns a Job ID

Process: Background Lambda invokes external LLM

Notify: EventBridge triggers notifications on completion

Retrieve: User either polls results or receives a webhook
```

💡 Personal note: I’d likely skip EventBridge and instead have the processing Lambda directly update DynamoDB or trigger a notification. That said, EventBridge is still a solid, decoupled option for event-based flows.

📊 3. `get_metrics` – Arguably the most useful tool in the suite, get_metrics provides natural language access to Lambda performance data.

I used it to analyze a Java-based Lambda over the past 7 days. I had allocated too little memory, and the tool correctly flagged slow execution times due to that constraint.

While you could get similar metrics using the AWS CLI or Console, the ability to just ask a question in natural language is a clear productivity boost—especially when diagnosing or iterating quickly.

Setting up the MCP server is straightforward. Just install uvx and follow the documentation to register an MCP server for your preferred coding assistant.


There’s been a noticeable shift in AWS tooling recently toward improving the developer experience for Lambda:

- A revamped Lambda code editor

- Live tailing of CloudWatch logs

- Improved AWS Toolkit for VSCode

- Now the MCP Server

This feels like a cohesive push to empower developers building on Serverless, and MCP could very well become an indispensable part of that toolkit.