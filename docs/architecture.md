# System Architecture & Network Security

This document details the network design, security infrastructure, and high-level architectural decisions for the Receipt Analyzer.

## 1. High-Level Design
The architecture follows a **Hybrid Cloud** model, combining the control of a virtual server (EC2) with the scalability of serverless AWS services (Textract, Bedrock, DynamoDB).

### Logical Flow
1.  **Orchestration**: Flask App (EC2) receives request.
2.  **Storage**: Image saved to Private S3 Bucket.
3.  **OCR**: App triggers Textract to read image from S3.
4.  **AI Analysis**: Extracted text sent to Bedrock for classification.
5.  **Persistence**: Metadata stored in DynamoDB.

---

## 2. Network Infrastructure (VPC)
The application runs within a custom Virtual Private Cloud to ensure network isolation.

*   **VPC ID**: `vpc-xxxxxxxxxxxxxxxxx`
*   **CIDR Block**: `10.0.0.0/24` (Implied/Standard)

### Subnet Configuration
The VPC is divided into public and private scopes:

| Subnet Name | Type | Purpose | Route Table | Gateway Association |
|:---|:---|:---|:---|:---|
| `pubsub1` | **Public** | Hosts the EC2 Web Server | `rtb-public-xxxx` | Internet Gateway (`igw-xxxxxxxx`) |
| `prisub1` | **Private** | Backend/Database resources (Reserved) | `rtb-private-xxxx` | None (Local traffic only) |

*   **Public Access**: `pubsub1` has a route `0.0.0.0/0` pointing to the IGW, allowing inbound HTTP traffic.
*   **Private Isolation**: `prisub1` has no internet route, protecting internal resources from direct external access.

---

## 3. Security Infrastructure

### 3.1 Identity & Access Management (IAM)
We adhere to the **Principle of Least Privilege**. No hardcoded AWS credentials are used.

*   **Role Name**: `EC2-S3-App-Role`
*   **Attached Entity**: The EC2 instance.
*   **Policies**: `EC2-S3-App-Access-Policy`
    *   `s3:PutObject`, `s3:GetObject` (Specific bucket only)
    *   `dynamodb:PutItem`, `dynamodb:Query` (Specific table only)
    *   `textract:AnalyzeDocument`
    *   `bedrock:InvokeModel`

### 3.2 Security Groups
Stateful firewalls applied at the EC2 instance level.

**Security Group:** `sg2bpub`

| Type | Protocol | Port | Source | Description |
|:---|:---|:---|:---|:---|
| **HTTP** | TCP | 80 | `0.0.0.0/0` | Public web access |
| **SSH** | TCP | 22 | `YOUR_ADMIN_IP/32` | **Admin Access Only** |
| **HTTPS**| TCP | 443| `0.0.0.0/0` | Prepared for SSL support |

### 3.3 Data Encryption
*   **At Rest**:
    *   **S3**: Server-Side Encryption (SSE-S3) enabled.
    *   **DynamoDB**: Encrypted by default (AWS Owned Key).
*   **In Transit**: All calls to AWS APIs (Textract, Bedrock, DynamoDB) use TLS 1.2+.
