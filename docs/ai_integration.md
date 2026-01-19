# AI Services Integration: Textract & Bedrock

This document details the usage of AWS Managed AI services to achieve Serverless OCR and Intelligent Categorization.

## 1. Optical Character Recognition (OCR): Amazon Textract
We use Amazon Textract's specialized `AnalyzeExpense` API, which is optimized for invoices and receipts.

### Workflow
1.  **Trigger**: `extract_receipt.py` -> `extract_receipt_data()`
2.  **Input**: Image bytes read from S3.
3.  **API Call**: `textract.analyze_expense()`
4.  **Extraction Logic**:
    *   **VENDOR_NAME**: Identifies the merchant.
    *   **TOTAL**: Identifies the final amount.
    *   **DATE**: Identifies transaction date.
    *   **Line Items**: Iterates through `LineItemGroups` to extract individual purchased items and prices.

### Why Textract?
*   No ML models to train or maintain.
*   Automatically handles skewed images and varied receipt formats.
*   Returns structured data (KV pairs), not just raw text.

---

## 2. Generative AI Classification: Amazon Bedrock
We use Amazon Bedrock to inject intelligence into the application, converting raw text into meaningful categories.

### Configuration
*   **Model**: Anthropic Claude 3 Sonnet (`anthropic.claude-3-sonnet-20240229`)
*   **Region**: `us-east-1`

### Prompt Engineering
The prompt is strictly engineered to ensure deterministic, single-word outputs suitable for database storage.

**Prompt Template**:
```text
Based on the following JSON data from a receipt, classify this transaction into one of these categories: 
Restaurant, Groceries, Transportation, Shopping, Utilities, Entertainment, or Other.

Your response MUST be a single word, which is one of the allowed categories. 
Do NOT include any other text.

<transaction_data>
{JSON_DUMP}
</transaction_data>

Category:
```

### Integration Logic (`classifier.py`)
1.  Constructs the prompt with the extracted JSON data.
2.  Invokes `bedrock-runtime` client.
3.  Parses the JSON response to simple string (e.g., "Groceries").
4.  This string is directly saved to DynamoDB as the `category` attribute.
