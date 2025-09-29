# Discovery Agent Local Setup Guide

## Environment Configuration

Based on your existing `.env` files, here's what you need to configure:

### ✅ **Values Already Available from Your .env**

```bash
# Core API Configuration
GRAPHQL_API_URL=http://172.21.112.1:8080/graphql
API_URL=http://172.21.112.1:8080
API_USERNAME=engineers+service-account@meetalix.com
API_PASSWORD=e554b5-HjEzsn2J8J8
MASTER_TOKEN=e554b5-HjEzsn2J8J8

# OpenAI Configuration
OPEN_AI_API_KEY=your_openai_api_key_here
```

### ⚠️ **Values You Need to Provide**

```bash
# AWS Configuration
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=your_aws_access_key_here
AWS_SECRET_ACCESS_KEY=your_aws_secret_key_here
AWS_SESSION_TOKEN=your_aws_session_token_here

# DynamoDB Tables
LOCK_TABLE_NAME=alix-lock-table
FILE_COUNTER_TABLE_NAME=alix-file-counter-table
DYNAMO_TASK_TABLE=alix-task-table

# S3 Bucket
AUTOMATION_BUCKET_NAME=alix-automation-bucket

# Box Integration
BOX_PRIVATE_KEY=your_box_private_key_here
BOX_CLIENT_SECRET=your_box_client_secret_here
BOX_PASSPHRASE=your_box_passphrase_here

# Google Sheets (Optional)
GOOGLE_SHEET_ID=your_google_sheet_id_here
GOOGLE_SERVICE_ACCOUNT_EMAIL=your_service_account_email_here
GOOGLE_PRIVATE_KEY=your_google_private_key_here
```

### 🔧 **Additional Configuration**

```bash
# Processing Settings
SAVE_ASSETS_TO=DB
INPUT_FILES_CHUNK_SIZE=10
TASK_ID=local-test-task

# Logging
LOG_LEVEL=debug
PRETTY_PRINT_LOGS=true
NODE_TLS_REJECT_UNAUTHORIZED=0

# Langfuse (Optional)
ENV_NAME=local
LANGFUSE_SECRET_KEY=optional_for_local_dev
LANGFUSE_PUBLIC_KEY=optional_for_local_dev
LANGFUSE_BASE_URL=http://localhost:3001
```

## Setup Steps

### 1. Create .env File
```bash
# Copy the template and fill in missing values
cp .env.template .env
# Edit .env with your actual AWS credentials and Box keys
```

### 2. Start Required Services

**Start alix-api system** (in separate repo):
```bash
# In your alix-api repo
# Make sure it's running on http://172.21.112.1:8080
```

**Start Langfuse** (optional but recommended):
```bash
# In this repo
docker-compose up -d
# Access Langfuse UI at http://localhost:3001
```

### 3. Install Dependencies
```bash
yarn install
```

### 4. Run Discovery Agent
```bash
# Test with a PDF file
yarn run:local file-processing '{"files": ["path/to/test.pdf"]}'
```

## Command Examples

### File Processing
```bash
# Process a single PDF
yarn run:local file-processing '{"files": ["/path/to/estate-documents.pdf"]}'

# Process multiple files
yarn run:local file-processing '{"files": ["file1.pdf", "file2.pdf"]}'
```

### Full Processing Pipeline
```bash
# Set environment variables for full processing
export FULL_PROCESSING_FILE_IDS="file1.pdf,file2.pdf"
export FULL_PROCESSING_ESTATE_ID="estate-123"
export FULL_PROCESSING_DECEASED_NAME="John Doe"
export FULL_PROCESSING_BOX_OUTPUT_FOLDER_ID="325102768414"

# Run full processing
yarn run:local full-processing
```

## Troubleshooting

### Common Issues

1. **GraphQL Connection Failed**
   - Ensure alix-api is running on `http://172.21.112.1:8080`
   - Check `MASTER_TOKEN` is correct

2. **AWS Credentials Error**
   - Verify AWS credentials are valid
   - Check DynamoDB tables exist
   - Ensure S3 bucket exists

3. **Box Integration Error**
   - Verify Box credentials
   - Check Box folder permissions

4. **OpenAI API Error**
   - Verify API key is valid
   - Check rate limits

### Debug Mode
```bash
# Enable debug logging
export LOG_LEVEL=debug
export PRETTY_PRINT_LOGS=true

# Run with verbose output
yarn run:local file-processing '{"files": ["test.pdf"]}'
```

## Next Steps

1. **Fill in missing AWS credentials** in your `.env` file
2. **Start alix-api system** in separate repo
3. **Test with sample PDF** to verify everything works
4. **Monitor Langfuse UI** for AI agent performance

The Discovery Agent should now be ready to process estate documents locally!
