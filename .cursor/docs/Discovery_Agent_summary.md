# Discovery Agent Analysis Summary

## Overview

The Discovery Agent (often referred to as "Disco Agent") is a sophisticated AI-powered document processing system designed to analyze estate settlement documents. It takes PDFs containing multiple scanned estate documents, separates them into individual pages/images, categorizes document types, and extracts structured information to populate various estate entities in the Alix Estate Manager system.

**Note**: This analysis combines the current codebase implementation, the intended process flow from Figma design specifications, and direct access to live Figma designs via MCP server integration.

## Figma Integration Status

### Direct Design Access
- **Figma Board**: `me4bKpF9C1rjEWvFoPO0zn` - "Discovery Data & Process"
- **MCP Server**: Successfully integrated and authenticated
- **Design Assets**: 3 high-quality Mermaid flow diagrams extracted
- **Access Method**: Real-time Figma API access (not PDF exports)

### Available Design Assets
1. **`discovery_agent_flow_1.png`** - Main process flow (513KB, 1110x4723px)
2. **`discovery_agent_flow_2.png`** - Processing pipeline (419KB, 1495x3505px)  
3. **`discovery_agent_flow_3.png`** - AI workflows (348KB, 1684x3111px)
4. **Creation Date**: All diagrams created 2025-08-29 (recent design work)
5. **Design Quality**: Professional Mermaid charts with consistent styling

## Architecture

### Core Components

The Discovery Agent consists of several key components working together:

1. **File Processing Engine** (`src/tasks/file-processing/`)
2. **AI Agent Framework** (`src/agent-framework/`)
3. **Document Processing Pipeline** (PDF → Images → Classification → Extraction → Database)
4. **LangGraph Workflows** for orchestrated AI processing
5. **GraphQL Integration** for database operations

## Intended Process Flow (From Figma Design)

### Customer Upload Process
1. **Customer Uploads Files** → Files stored in customer's "My Uploads"
2. **Care Team Receives Files** → Email with download link forwarded to Slack #ops-arcupdates
3. **File Processing** → Unzip files with password, place in Input folder
4. **Run Discovery Agent** → Automated processing begins

### Document Processing Goals
- **File Splitting**: If file contains multiple docs grouped together, split them into constituent files
- **Document Classification**: Determine institution, date (MM-DD-YYYY), and document type
- **Document Naming**: Format as "{Institution-NameDocument-Type-MMDD-YYYY}"
- **Folder Organization**: Sort files into appropriate subfolders in Output

### Data Extraction Priorities
**P1 - Common documents, multiple instances per estate, multiple components per document:**
- Bank account statements (checking, savings, money market, CDs)
- Credit card statements
- Investment/brokerage account statements
- Retirement account statements
- Insurance statements
- Utility bills and recurring subscriptions

**P2 - Common documents, single instance per estate, multiple components per document:**
- Trusts and Wills
- IRS tax transcripts and returns
- Credit reports
- Social Security earnings reports

**P3 - Less common documents, single instance per estate AND/OR only one component per document:**
- Various specialized documents and records

### Key Business Logic Requirements
- **Date of Death (DoD) Value**: Extract balance at DoD for assets
- **Estimated Value**: Use most recent discovered balance
- **Duplicate Detection**: Check against primary keys to avoid duplicates
- **Document Association**: Attach supporting documents to extracted entities
- **Folder Structure**: Organize documents into logical categories (Assets, Debts, Obligations, etc.)

## Current Implementation vs. Intended Design

### ✅ Implemented Features
- PDF to image conversion and processing
- AI-powered document classification (30+ document types)
- Page grouping and document splitting
- Fact extraction for assets, debts, contacts, obligations
- Database persistence with proper relationships
- Supporting document attachment
- Confidence scoring for data quality

### 🔄 Partially Implemented Features
- **Document Naming**: Current system doesn't implement the "{Institution-NameDocument-Type-MMDD-YYYY}" naming convention
- **Folder Organization**: Documents are stored but not organized into the intended folder structure
- **DoD Value Extraction**: System extracts current balances but may not properly handle DoD-specific values

### ❌ Missing Features
- **Human-in-the-Loop System (CoPilot)**: No review interface for care team
- **Automated Testing System**: No QA system to test against expected results
- **Enhanced Duplicate Detection**: Current system uses simple primary key matching instead of vector-based LLM reasoning
- **Better Fact Identification**: System needs improvement in asking "does this represent new information about the estate"

### 🚨 Known Issues (From Figma)
- **Incorrect Data Extraction**: System extracts beneficiary bank account info from retirement claim forms when it shouldn't
- **Folder Naming**: Need to rename "Mortgages" → "Home Loans", "Car Loans" → "Vehicle Loans"
- **Insurance Handling**: Need to separate "Life Insurance" folder from general "Insurance" folder
- **Reasoning Tasks**: Need to separate different reasoning tasks to ensure correct data extraction

### 📊 Visual Comparison Findings (From Live Figma Designs)
- **Missing CoPilot Interface**: Figma designs clearly show human review system that doesn't exist in current implementation
- **No Document Organization**: Designs show structured folder system (Assets/Bank Accounts, Debts/Home Loans, etc.) not implemented
- **Missing Care Team Workflow**: Designs include email notifications, Slack integration, and task creation not in current system
- **No DoD Value Logic**: Designs show separate handling of date of death vs. current values not implemented
- **Missing External Integration**: Designs show multiple data sources (MissingMoney.com, credit reports, etc.) not connected

## Data Flow Pipeline

### 1. Input Processing (`src/tasks/file-processing/index.ts`)

**Entry Point**: `fileProcessing()` function
- Receives `estateId` and `boxOutputFolderId` from task payload
- Retrieves deceased information from database
- Prepares input files and creates processing sessions
- Processes files in chunks using `processItemsInChunks()`

**Key Steps**:
- Creates Langfuse session for AI tracking
- Groups files by type (PDF, images, unsupported)
- Registers file sessions in database
- Handles failed file processing gracefully

### 2. File Preparation (`src/tasks/file-processing/engine/prepareFile.ts`)

**PDF Processing**:
- Downloads PDF from Box service
- Converts PDF pages to individual JPEG images using Poppler
- Uploads images to S3 for AI processing
- Creates mapping of page numbers to S3 file paths

**Image Processing**:
- Downloads individual image files
- Converts to supported formats (JPEG/PNG)
- Uploads to S3 for processing

### 3. Document Splitting (`src/tasks/file-processing/graphs/document-splitter/`)

**Page Classification** (`src/tasks/file-processing/agents/pageClassifier.ts`):
- Uses GPT-4.1-mini to analyze each page image
- Extracts text content using AWS Textract
- Classifies document type from 30+ categories (Will, BankAccountStatement, etc.)
- Identifies institutions, dates, account numbers, assets
- Detects handwritten vs. typed documents
- Provides confidence scores for all classifications

**Page Grouping** (`src/tasks/file-processing/agents/pageGrouper.ts`):
- Groups classified pages into complete documents
- Uses document continuation cues and cross-references
- Creates logical document boundaries
- Handles multi-page documents correctly

### 4. Fact Extraction (`src/tasks/file-processing/graphs/fact-extractor/`)

**Workflow Orchestration**:
- Uses LangGraph for complex AI workflow management
- Conditional routing based on document type
- Sequential processing: Identify → Extract Assets → Extract Debts → Extract Contacts → Extract Obligations

**Fact Identification** (`src/tasks/file-processing/agents/factIdentifier.ts`):
- Uses GPT-4o-mini with medium reasoning effort
- Analyzes document content to identify extractable data
- Makes intelligent deductions (e.g., mortgage = loan + home)
- Identifies assets, debts, obligations, and contacts
- Provides confidence scores for each identification

**Specialized Extractors**:
- **Deceased Extractor**: Extracts detailed deceased information from death certificates
- **Asset Extractor**: Extracts detailed asset information (real estate, vehicles, investments, etc.)
- **Debt Extractor**: Extracts debt information (mortgages, loans, credit cards, etc.)
- **Contact Extractor**: Extracts contact information (heirs, attorneys, financial advisors, etc.)
- **Obligation Extractor**: Extracts ongoing obligations (utilities, subscriptions, insurance, etc.)

### 5. Data Persistence (`src/tasks/file-processing/engine/saveFacts/`)

**Database Operations**:
- Uses distributed locking to prevent concurrent modifications
- Checks for existing entities before creating new ones
- Links extracted facts to supporting documents
- Creates events for tracking data enrichment
- Handles deceased information updates with validation

**Entity Creation**:
- **Assets**: Real estate, vehicles, bank accounts, investments, retirement accounts
- **Debts**: Mortgages, car loans, credit cards, personal loans, medical debt
- **Contacts**: Heirs, attorneys, financial advisors, insurance agents
- **Obligations**: Utilities, subscriptions, insurance payments, maintenance
- **Supporting Documents**: Links processed files to extracted entities

## Document Categories

The system recognizes 30+ document types:

**Financial Documents**:
- BankAccountStatement, CreditCardStatement, InvestmentAccountDocument
- RetirementAccountDocument, LifeInsuranceDocument, TaxForm

**Legal Documents**:
- Will, Trust, DeathCertificate, BirthCertificate, DriverLicense

**Property Documents**:
- RealEstateDocument, MortgageDocument, HomeInsuranceDocument
- VehicleRegistration, VehicleInsuranceDocument

**Other Documents**:
- MedicalDocument, UtilityBill, SocialSecurityForm, Invoice, Receipt
- HandwrittenDocument, Unidentified, MISC

## AI Models Used

- **GPT-4.1-mini**: Page classification and fact extraction
- **GPT-4o-mini**: Fact identification with medium reasoning effort
- **AWS Textract**: Text extraction from images
- **LangGraph**: Workflow orchestration and state management

## Key Features

### Intelligent Document Processing
- Handles both typed and handwritten documents
- Supports multi-page documents with continuation detection
- Cross-references between documents
- Confidence scoring for all extractions

### Robust Error Handling
- Graceful failure handling for individual files
- Retry logic for AI operations
- Comprehensive logging and monitoring
- Safe execution patterns throughout

### Data Validation
- Schema validation using Zod
- Confidence thresholds for data quality
- Duplicate detection and prevention
- Relationship validation between entities

### Scalability
- Chunked processing for large file sets
- Background processing capabilities
- Distributed locking for concurrent access
- Efficient S3 storage and retrieval

## Integration Points

### External Services
- **Box**: File storage and retrieval
- **S3**: Image storage and processing
- **AWS Textract**: OCR and text extraction
- **OpenAI**: AI model inference
- **Langfuse**: AI operation tracking and monitoring

### Database Integration
- **GraphQL**: Database operations and mutations
- **PostgreSQL**: Primary data storage
- **DynamoDB**: Distributed locking and task management

### Estate Manager Integration
- Populates estate entities (assets, debts, contacts, obligations)
- Links supporting documents to extracted facts
- Creates audit trails and events
- Maintains data consistency across the system

## Configuration

### Environment Variables
- `LOG_LEVEL`: Logging verbosity
- `PRETTY_PRINT_LOGS`: Log formatting
- `INPUT_FILES_CHUNK_SIZE`: Batch processing size
- `AUTOMATION_BUCKET_NAME`: S3 bucket for processing
- `AWS_REGION`: AWS service region
- `ENV_NAME`: Environment identifier

### AI Model Configuration
- Temperature settings for deterministic output
- Reasoning effort levels for complex tasks
- Confidence thresholds for data quality
- Retry policies for AI operations

## Performance Considerations

### Processing Efficiency
- Parallel processing of multiple files
- Chunked operations to prevent memory issues
- Efficient image conversion and storage
- Optimized database queries with proper indexing

### Cost Optimization
- Selective AI model usage based on task complexity
- Efficient image processing to minimize API calls
- Batch operations where possible
- Monitoring and alerting for cost control

## Monitoring and Observability

### Logging
- Structured logging with context propagation
- Performance metrics and timing information
- Error tracking and debugging information
- Audit trails for data modifications

### AI Tracking
- Langfuse integration for AI operation monitoring
- Token usage and cost tracking
- Model performance metrics
- Workflow execution tracking

## Security and Compliance

### Data Protection
- Secure file handling and temporary storage cleanup
- Access control through AWS IAM
- Encrypted data transmission and storage
- Audit logging for compliance

### Error Handling
- Safe execution patterns to prevent data corruption
- Graceful degradation on service failures
- Comprehensive error reporting and recovery
- Data validation at multiple levels

## Data Sources and Integration Opportunities

### Current Data Sources
- **Paper Statements**: Scanned by customer or forwarded mail scanning service
- **eStatements**: Forwarded from email
- **Direct Connections**: Plaid integration for bank accounts (limited to trust accounts)

### Planned Data Sources (From Figma)
- **State Unclaimed Property**: MissingMoney.com (49 of 50 states)
- **Federal Unclaimed Property**: Treasury Direct and various agencies
- **Social Security Earnings Reports**: SSA-7004 form
- **IRS Transcripts**: Tax transcripts and returns
- **Credit Reports**: All three bureaus
- **Insurance Policies**: NAIC Life Policy Locator
- **Employer Records**: Personnel files and benefits
- **Data Aggregators**: TLOx, idiCORE, impact360

### Integration Priorities
**Plaid Integration Considerations:**
- ~20% of population has trust accounts (likely to have credentials)
- ~50% (±20%) may have account access through saved passwords
- Total estimated access: 50-70% of population
- Challenge: Determining who has credentials vs. who doesn't

## Future Enhancements

### Immediate Priorities (From Figma)
1. **Human-in-the-Loop System (CoPilot)**: Build review interface for care team
2. **Automated Testing System**: QA system with 10-20 sample document sets
3. **Enhanced Duplicate Detection**: Vector-based LLM matching instead of primary keys
4. **Better Fact Identification**: Improve reasoning about "new information about the estate"
5. **Document Naming**: Implement "{Institution-NameDocument-Type-MMDD-YYYY}" convention
6. **Folder Organization**: Proper subfolder structure in Output

### Technical Improvements
- Enhanced handwritten document recognition
- Multi-language document support
- Advanced document relationship detection
- Real-time processing capabilities
- Enhanced confidence scoring algorithms
- Better handling of DoD vs. current values

### Business Logic Enhancements
- Separate reasoning tasks for different document types
- Improve beneficiary information handling
- Better insurance document categorization
- Enhanced expense vs. asset classification
- Improved duplicate detection with reasoning

### Scalability Considerations
- Horizontal scaling for high-volume processing
- Caching strategies for frequently accessed data
- Advanced queue management for task processing
- Performance optimization for large document sets
- Integration with external data sources

---

*This analysis provides a comprehensive overview of the Discovery Agent's architecture, functionality, and integration within the Alix Estate Manager ecosystem. The system represents a sophisticated approach to automated estate document processing using modern AI technologies and cloud services.*
