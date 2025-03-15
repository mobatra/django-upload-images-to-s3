# 📌 Full Process of Uploading an Image in Django with S3 (LocalStack)

## 🛠 1️⃣: User Sends the Request
The **frontend (or API client like Insomnia)** sends a **POST request** to upload a receipt image.

### 🔹 Example API Request (Insomnia or Postman)
**`POST /api/v1/transactions/`**
- **Headers:**
  ```
  Authorization: Bearer YOUR_ACCESS_TOKEN
  Content-Type: multipart/form-data
  ```
- **Form-Data Body:**
  | Key         | Value           | Type  |
  |------------|----------------|------|
  | amount     | `150.00`        | Text |
  | category   | `food`          | Text |
  | description | `Lunch Receipt` | Text |
  | receipt    | `Choose File` (Upload an image) | File |

---

## 🛠 2️⃣: Django Receives and Validates the Request
- The **request hits** `TransactionListCreateView` in `views.py`.
- Django **automatically detects** the uploaded file from `request.FILES`.
- The **serializer validates** that:
  - `amount`, `category`, and `description` exist.
  - The `receipt` is a **valid image** (JPEG, PNG, etc.).

### 🔹 Validation (Serializer)
```python
from rest_framework import serializers
from .models import Transaction

class TransactionSerializer(serializers.ModelSerializer):
    receipt_url = serializers.SerializerMethodField()  # ✅ Dynamically generate file URL

    class Meta:
        model = Transaction
        fields = ["id", "amount", "category", "description", "date", "receipt", "receipt_url"]

    def get_receipt_url(self, obj):
        if obj.receipt:
            return obj.receipt.url  # ✅ Get the actual file URL
        return None
```
#### 🔍 How receipt_url Works

- serializers.SerializerMethodField() creates a computed field.

- DRF automatically calls get_receipt_url(self, obj) when serializing data.

- obj.receipt.url retrieves the public URL for the uploaded file.



---

## 🛠 3️⃣: Django Saves the Image in S3
- Since **`DEFAULT_FILE_STORAGE`** is set to `S3Boto3Storage`, Django will:
  1. **Upload the image** to **S3 (LocalStack) instead of storing it locally**.
  2. **Generate a URL for the uploaded image**.
  3. **Store the URL in the database** (not the actual image).

### 🔹 How Django Stores the Image
```python
DEFAULT_FILE_STORAGE = "storages.backends.s3boto3.S3Boto3Storage"
```
- Django **does not save the file locally**.
- The **uploaded file is sent to S3 (LocalStack)**.

---

## 🛠 4️⃣: S3 Stores the Image and Returns a URL
- The **image is now stored in your S3 bucket** (`finance-receipts`).
- Django **retrieves the public URL** for the image and saves it in the database.

### 🔹 Example S3 URL (LocalStack)
```
http://localhost:4566/finance-receipts/user_123/receipt_456.jpg
```
- The image is accessible **via this URL**.

---

## 🛠 5️⃣: Database Saves the Transaction with the S3 Image URL
- The **database stores only the image URL**, not the image itself.
- Example record in `transactions_transaction` table:

| ID  | User  | Amount | Category | Description | Receipt (S3 URL) |
|----|------|---------|---------|------------|------------------|
| 1  | 123  | 150.00  | Food    | Lunch      | `http://localhost:4566/finance-receipts/user_123/receipt_456.jpg` |

---

## 🛠 6️⃣: Django Returns a Response
- The **API responds with JSON** containing the transaction details and the **S3 URL**.

### 🔹 Example API Response
```json
{
  "message": "Transaction created successfully",
  "data": {
    "id": 1,
    "amount": 150.00,
    "category": "food",
    "description": "Lunch Receipt",
    "date": "2024-03-08",
    "receipt": "http://localhost:4566/finance-receipts/user_123/receipt_456.jpg"
  }
}
```
- **Frontend (React, Vue, etc.) can display the receipt image** using the S3 URL.

---

## 🔄 What Happens When the User Retrieves Transactions?
When the user fetches their transactions:
- Django **retrieves** the stored S3 URL from the database.
- The frontend **displays the image using the S3 URL**.

### 🔹 Example GET Request
```http
GET /api/v1/transactions/
Authorization: Bearer YOUR_ACCESS_TOKEN
```

### 🔹 Example Response
```json
{
  "id": 1,
  "amount": 150.00,
  "category": "food",
  "description": "Lunch Receipt",
  "date": "2024-03-08",
  "receipt": "http://localhost:4566/finance-receipts/user_123/receipt_456.jpg"
}
```
- **No need to fetch the image from Django!**
- The frontend **just loads the image from S3**.

---

## 📌 Summary of the Full Process
1️⃣ **User uploads the receipt (image) via API request.**  
2️⃣ **Django validates the request and the file type.**  
3️⃣ **Django sends the image to S3 using `storages.backends.s3boto3.S3Boto3Storage`.**  
4️⃣ **S3 stores the image and provides a public URL.**  
5️⃣ **Django saves the transaction with the S3 URL in the database.**  
6️⃣ **The frontend retrieves the transaction and displays the receipt image using the S3 URL.**  

---
