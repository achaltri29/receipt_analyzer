# Data Storage: S3 & DynamoDB

This document outlines the persistence layer of the application, covering both object storage (files) and NoSQL database (metadata).

## 1. Object Storage: Amazon S3
Amazon Simple Storage Service (S3) is used to store the raw receipt images uploaded by users.

*   **Bucket Strategy**: Private storage with secure, temporary access.
*   **Bucket Name**: (Configured via `S3_BUCKET` env var, e.g., `receipt-analyzer-receipts`)

### Configuration & Security
1.  **Block Public Access**: **Enabled (All)**. The bucket is NOT public website hosting.
2.  **Encryption**: Server-Side Encryption (SSE-S3) enabled (AES-256).
3.  **Access Pattern**:
    *   **Upload**: Backend (EC2) puts object using IAM role permissions.
    *   **Textract Read**: Textract service reads object directly from S3.
    *   **User View**: Backend generates a **Pre-signed URL** (valid for 1 hour). This allows the frontend to display the image securely without exposing the bucket.

### File Structure
Objects are stored with the following key pattern:
`receipts/<filename>_<timestamp>.<ext>`

---

## 2. Metadata Database: Amazon DynamoDB
Amazon DynamoDB provides serverless, single-digit millisecond latency storage for the application state (receipt metadata).

*   **Table Name**: (Configured via `DYNAMODB_TABLE` env var)
*   **Capacity Mode**: On-Demand (or Provisioned usually within Free Tier limits).

### Schema Design

| Attribute | Type | Role | Description |
|:---|:---|:---|:---|
| `receipt` | String | **Partition Key (PK)** | Unique ID for the receipt entry. |
| `user` | String | GSI Partition Key | The username (e.g., "owais"). |
| `date` | String | GSI Sort Key | Date of transaction (YYYY-MM-DD). |
| `s3_key` | String | Attribute | Reference path to S3 object. |
| `total` | Number | Attribute | Extracted total amount. |
| `category` | String | Attribute | AI-assigned category. |
| `items` | List | Attribute | List of extracted line items. |

### Indexes
To support user-specific dashboards and date filtering, a **Global Secondary Index (GSI)** is used.

*   **GSI Name**: `user-index`
*   **Key Schema**:
    *   Partition Key: `user`
    *   Sort Key: `date`
*   **Purpose**: Allows efficient querying of "All receipts for User X, sorted by Date" without scanning the entire table.
