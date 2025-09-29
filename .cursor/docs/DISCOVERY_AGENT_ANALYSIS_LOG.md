# Discovery Agent Analysis Work Log

## Session Overview
**Date**: 2025-09-16  
**Task**: Analyze Discovery Agent codebase and create comprehensive documentation  
**Jira Task**: AE-1206 - Map out existing Disco Agent Flow & Logic / Reasoning and Suggest Changes  

## Work Completed

### 1. Initial Codebase Analysis
- **Analyzed**: Complete Discovery Agent codebase structure
- **Key Files Reviewed**:
  - `src/tasks/file-processing/index.ts` - Main entry point
  - `src/tasks/file-processing/engine/` - Core processing logic
  - `src/tasks/file-processing/agents/` - AI agent implementations
  - `src/tasks/file-processing/graphs/` - LangGraph workflows
  - `src/services/pdf.ts` - PDF processing utilities
  - `src/services/textract.ts` - OCR integration

### 2. Figma Documentation Analysis
- **Reviewed**: "Discovery Data & Process.pdf" - Figma export of intended design
- **Key Insights**:
  - Comprehensive process flow from customer upload to care team review
  - Document naming convention: "{Institution-NameDocument-Type-MMDD-YYYY}"
  - Priority-based processing (P1, P2, P3)
  - Human-in-the-loop CoPilot system requirements
  - External data source integration plans

### 3. Gap Analysis
- **Identified**: Significant gaps between current implementation and intended design
- **Critical Issues**:
  - Incorrect data extraction (beneficiary bank info from retirement forms)
  - Missing document organization and naming
  - No human review system
  - Poor duplicate detection (primary keys vs. vector-based reasoning)
  - No DoD value handling

### 4. Documentation Created

#### A. Comprehensive Summary
- **File**: `Discovery_Agent_summary.md`
- **Content**: Complete analysis combining codebase and Figma insights
- **Sections**:
  - Current implementation analysis
  - Intended process flow from Figma
  - Gap analysis and comparison
  - Data sources and integration opportunities
  - Future enhancement recommendations

#### B. Mermaid Diagrams
- **File**: `Discovery_Agent_Current_Flow.md`
- **Content**: Current implementation flow diagram
- **Includes**: AI agent prompts, business logic, limitations

- **File**: `Discovery_Agent_Intended_Flow.md`
- **Content**: Intended design flow from Figma
- **Includes**: Enhanced AI improvements, business logic, features

- **File**: `Discovery_Agent_Gap_Analysis.md`
- **Content**: Visual comparison of current vs. intended
- **Includes**: Priority matrix, technical gaps, implementation roadmap

#### C. Conversion Tools
- **File**: `convert_mermaid.sh`
- **Purpose**: Script to convert Mermaid diagrams to images
- **Status**: Created but requires Mermaid CLI installation

### 5. Memory Updates
- **Updated**: `.cursor/memory/memory.md`
- **Added**: Discovery Agent Analysis Insights section
- **Content**: Architecture patterns, AI agent patterns, implementation gaps, business logic requirements

## Key Findings

### Current Implementation Strengths
- Solid AI-powered document processing pipeline
- Comprehensive agent-based architecture
- Good error handling and monitoring
- Efficient chunked processing
- Proper database relationships and event tracking

### Critical Gaps Identified
1. **Document Organization**: No standardized naming or folder structure
2. **Human Oversight**: Missing CoPilot review system
3. **Quality Assurance**: No automated testing framework
4. **Enhanced Reasoning**: Poor duplicate detection and fact identification
5. **Business Logic**: Missing DoD value handling and context awareness

### Business Impact
- **Incorrect Data Extraction**: System extracts beneficiary info when it shouldn't
- **Poor User Experience**: No human review interface for care team
- **Quality Issues**: No systematic testing against known good results
- **Scalability Concerns**: Missing external data source integration

## Recommendations for Jira Task AE-1206

### Immediate Priorities (P1)
1. Fix incorrect data extraction logic
2. Implement DoD value handling
3. Enhance duplicate detection with reasoning
4. Add document naming convention

### Short-term Goals (P2)
1. Build CoPilot review interface
2. Implement folder organization
3. Create automated testing framework
4. Enhance AI prompts for better reasoning

### Long-term Vision (P3)
1. External data source integration
2. Advanced reasoning engine
3. Performance optimization
4. Real-time processing capabilities

## Technical Dependencies Identified
- **AWS Services**: S3, Textract, DynamoDB
- **External APIs**: Box, OpenAI, Langfuse
- **Database**: PostgreSQL with GraphQL
- **Processing**: Poppler, Sharp
- **Future**: Plaid, MissingMoney.com, credit reports, etc.

## Next Steps
1. **Present Findings**: Share comprehensive analysis with PM
2. **Prioritize Changes**: Focus on P1 critical issues first
3. **Design Solutions**: Create detailed implementation plans
4. **Build Testing Framework**: Establish QA system with sample documents
5. **Implement CoPilot**: Build human review interface

## Files Created/Modified
- `Discovery_Agent_summary.md` - Comprehensive analysis
- `Discovery_Agent_Current_Flow.md` - Current implementation diagrams
- `Discovery_Agent_Intended_Flow.md` - Intended design diagrams
- `Discovery_Agent_Gap_Analysis.md` - Gap analysis and roadmap
- `convert_mermaid.sh` - Diagram conversion script
- `.cursor/memory/memory.md` - Updated with Discovery Agent insights
- `.cursor/docs/DISCOVERY_AGENT_ANALYSIS_LOG.md` - This work log

## Detailed Discovery Agent Flow & Logic Analysis

### High-Level Processing Flow

The Discovery Agent follows a sophisticated multi-stage pipeline that transforms raw document files into structured estate data:

```
Input Files → Document Splitting → Fact Extraction → Database Storage → Post-Processing
```

### Stage 1: Input Preparation (`prepareInput`)

**Purpose**: Initialize processing session and categorize input files

**Process**:
1. **File Resolution**: Resolve Box file IDs to file metadata
2. **Page Count Detection**: Determine number of pages in each file
3. **Session Creation**: Create `DiscoveryAgentRunHistory` record with `RUNNING` status
4. **File Registration**: Create `DiscoveryAgentFileRunInfo` records for each file batch
5. **File Grouping**: Group files by type (PDF, image, unsupported)
6. **Status Tracking**: Mark unsupported files as `FAILED`

**Key Components**:
- `resolvePayloadFiles()`: Converts Box file IDs to file metadata
- `getPageCount()`: Determines page count for PDF/image files
- `groupInputFiles()`: Categorizes files by extension
- `registerBoxFiles()`: Creates database records for tracking

### Stage 2: Document Processing (`processInputData`)

**Purpose**: Convert multi-page documents into individual pages and classify them

**Process**:
1. **PDF Conversion**: Convert PDF files to individual JPEG images using Poppler
2. **Image Upload**: Upload images to S3 with signed URLs
3. **Document Splitting**: Use LangGraph workflow to split and classify pages
4. **Page Grouping**: Group related pages back into complete documents

**LangGraph Workflow - Document Splitter**:
```
START → classifyPages → sortPagesByCategories → groupPagesIntoDocuments → END
```

**Key Components**:
- `createDocumentSplitterGraph()`: Main LangGraph workflow
- `classifyPagesNode()`: Processes each page through Page Classifier
- `sortPagesByCategoriesNode()`: Categorizes pages by document type
- `groupPagesIntoDocumentsNode()`: Groups pages into complete documents

### Stage 3: Fact Extraction (`extractFacts`)

**Purpose**: Extract structured data from classified documents

**Process**:
1. **Document Analysis**: Analyze each grouped document for extractable facts
2. **Entity Identification**: Identify assets, debts, contacts, obligations
3. **Data Extraction**: Extract detailed information for each entity type
4. **Validation**: Ensure extracted data meets schema requirements

**LangGraph Workflow - Fact Extractor**:
```
START → [Death Certificate? → extractDeceased] OR [identifyFacts → extractAssets → extractDebts → extractContacts → extractObligations] → END
```

**Key Components**:
- `createFactExtractorWorkflow()`: Main fact extraction workflow
- `identifyFactsNode()`: Identifies what data can be extracted
- `extractAssetsNode()`: Extracts asset information
- `extractDebtsNode()`: Extracts debt information
- `extractContactsNode()`: Extracts contact information
- `extractObligationsNode()`: Extracts obligation information

### Stage 4: Database Storage (`saveFacts`)

**Purpose**: Persist extracted facts to database with proper relationships

**Process**:
1. **Duplicate Detection**: Check for existing entities using primary key matching
2. **Entity Creation**: Create new entities (assets, debts, contacts, obligations)
3. **Relationship Mapping**: Link entities to supporting documents
4. **Event Tracking**: Create audit trail events for data enrichment
5. **Supporting Documents**: Save document metadata and links

**Key Components**:
- `saveFactsToDB()`: Main database persistence function
- `shouldSaveComponent()`: Determines if entity should be saved
- `findAssetByPrimaryKey()`: Checks for existing assets
- `createAsset()`: Creates new asset records
- `attachSupportingDocumentsToAsset()`: Links documents to entities

### Stage 5: Post-Processing (`runPostProcess`)

**Purpose**: Perform additional processing after facts are saved

**Process**:
1. **Document Upload**: Upload processed documents to Box
2. **Status Updates**: Update file processing status to `COMPLETED`
3. **Cleanup**: Remove temporary files and resources

## Individual AI Agent Analysis

### 1. Page Classifier Agent (`pageClassifier`)

**Model**: GPT-4.1-mini with Vision capabilities
**Purpose**: Analyze individual document pages and extract comprehensive classification data

**Input**: Image URL of document page
**Output**: `ClassifierPageDescription` with:
- `textContent`: Extracted text from the page
- `pageDescription`: Human-readable description
- `isHandwritten`: Boolean indicating if page contains handwriting
- `documentName`: Identified document name
- `institutions`: Array of institutions mentioned
- `pageCategories`: Document categories (minimum 3 required)
- `dateOfDocument`: Document date
- `assets`: Array of identified assets
- `accountIdentifier`: Account numbers or identifiers
- `importantDates`: Significant dates found
- `confidence`: Overall confidence score
- `processingNotes`: Notes for document grouping

**System Instructions**: "Act as a estate settlement expert. Analyze individual document page and provide comprehensive classification data for estate processing."

**Special Features**:
- **Handwriting Detection**: Special workflow for handwritten documents
- **OCR Capability**: Extracts text directly from images using GPT-4 Vision
- **Multi-Category Classification**: Identifies multiple document types per page
- **Confidence Scoring**: Provides confidence levels for all extractions

### 2. Handwritten Document Agent (`handwritten`)

**Model**: GPT-4.1-mini with Vision capabilities
**Purpose**: Determine if a document contains machine-written text

**Input**: Image URL of document page
**Output**: `{ containsMachineText: boolean, confidence: number }`

**System Instructions**: "Does document contain machine written text?"

**Usage**: Called automatically when Page Classifier detects handwritten content

### 3. Page Grouper Agent (`pageGrouper`)

**Model**: GPT-4.1-mini
**Purpose**: Group individual pages into complete documents

**Input**: Array of `PageDescription` objects with classification data
**Output**: `GroupedDocuments` with:
- `groups`: Array of document groups
- `groupingStrategy`: Explanation of grouping approach
- `groupingConfidence`: Overall confidence in grouping

**System Instructions**: "Act as a estate settlement expert. Group pages into documents."

**Key Features**:
- **Multi-Document Pages**: Handles pages with multiple documents
- **Continuation Detection**: Identifies document continuations
- **Cross-Reference Analysis**: Links related pages
- **Institution Grouping**: Groups by institution and document type

### 4. Fact Identifier Agent (`factIdentifier`)

**Model**: GPT-4o-mini with reasoning capabilities
**Purpose**: Identify what data can be extracted from documents

**Input**: `ExtractFactsDocument` with text content and deceased name
**Output**: `FactIdentification` with:
- `assets`: Array of identified assets
- `debts`: Array of identified debts
- `obligations`: Array of identified obligations
- `contacts`: Array of identified contacts

**System Instructions**: "Act as a estate settlement expert. Analyze document and identify which data can be extracted from it. Be intelligent enough to make deductions..."

**Key Features**:
- **Reasoning Capabilities**: Uses GPT-4o-mini with medium reasoning effort
- **Business Logic**: Understands estate settlement context
- **Deduction Logic**: Makes intelligent inferences (e.g., mortgage = loan + home)
- **Context Awareness**: Distinguishes between deceased and beneficiary information

### 5. Asset Extractor Agent (`assetExtractor`)

**Model**: GPT-4.1-mini
**Purpose**: Extract detailed asset information from documents

**Input**: `AssetExtractorContext` with identified assets, document, and deceased name
**Output**: `ExtractedAssets` with detailed asset information

**System Instructions**: "Extract detailed asset information from documents. For each identified asset, extract all required fields based on the asset category."

**Key Features**:
- **Category-Specific Extraction**: Different fields based on asset type
- **Mandatory Field Validation**: Ensures all required fields are populated
- **Account Information**: Extracts account numbers and identifiers
- **Value Extraction**: Identifies asset values and dates

### 6. Debt Extractor Agent (`debtExtractor`)

**Model**: GPT-4.1-mini
**Purpose**: Extract detailed debt information from documents

**Input**: `DebtExtractorContext` with identified debts, document, and deceased name
**Output**: `ExtractedDebts` with detailed debt information

**System Instructions**: "Extract detailed debt information from documents. For each identified debt, extract all required fields based on the debt category."

**Key Features**:
- **Debt Categories**: Handles personal, business, real estate, vehicle debts
- **Account Numbers**: Extracts creditor account numbers
- **Amount Information**: Identifies debt amounts and balances
- **Status Tracking**: Determines debt status (paid, outstanding, etc.)

### 7. Contact Extractor Agent (`contactExtractor`)

**Model**: GPT-4.1-mini
**Purpose**: Extract detailed contact information from documents

**Input**: `ContactExtractorContext` with identified contacts, document, and deceased name
**Output**: `ExtractedContacts` with detailed contact information

**System Instructions**: "Extract detailed contact information from documents. For each identified contact, extract all required fields based on the contact type."

**Key Features**:
- **Contact Types**: Individual vs. institution contacts
- **Role Identification**: Estate roles (executor, beneficiary, etc.)
- **Contact Information**: Names, addresses, phone numbers
- **Relationship Mapping**: Links contacts to estate entities

### 8. Obligation Extractor Agent (`obligationExtractor`)

**Model**: GPT-4.1-mini
**Purpose**: Extract detailed obligation information from documents

**Input**: `ObligationExtractorContext` with identified obligations, document, and deceased name
**Output**: `ExtractedObligations` with detailed obligation information

**System Instructions**: "Extract detailed obligations information from documents. For each identified obligation, extract all required fields based on the obligation type."

**Key Features**:
- **Obligation Categories**: Utilities, insurance, subscriptions, etc.
- **Account Information**: Account numbers and identifiers
- **Service Details**: Service descriptions and terms
- **Status Tracking**: Active, cancelled, transferred status

### 9. Deceased Extractor Agent (`deceasedExtractor`)

**Model**: GPT-4.1-mini
**Purpose**: Extract detailed deceased information from death certificates

**Input**: `DeceasedExtractorContext` with document (death certificate)
**Output**: `Deceased` with detailed deceased information

**System Instructions**: "Act as estate settlement expert. Extract detailed deceased information from death certificate."

**Key Features**:
- **Death Certificate Specific**: Specialized for death certificate processing
- **Personal Information**: Name, date of birth, date of death
- **Demographic Data**: Sex, citizenship, marital status
- **Location Information**: Place of death, birthplace

## Data Flow and State Management

### LangGraph State Management

**Document Splitter State**:
```typescript
{
  pagesMap: Map<PageNumber, PageDescription>,
  categoriesMap: Record<RawCategories, PageNumber[]>,
  groupedDocuments: GroupedDocument[]
}
```

**Fact Extractor State**:
```typescript
{
  document: ExtractFactsDocument,
  facts: FactIdentification,
  extractedAssets: ExtractedAssets,
  extractedDebts: ExtractedDebts,
  extractedContacts: ExtractedContacts,
  extractedObligations: ExtractedObligations,
  extractedDeceased: Deceased
}
```

### Error Handling and Recovery

**Graceful Degradation**: System continues processing even if individual files fail
**Status Tracking**: Each file processing step is tracked in database
**Retry Logic**: Failed files are marked with appropriate status
**Logging**: Comprehensive logging with Langfuse integration for AI operations

### Performance Optimizations

**Chunked Processing**: Files processed in configurable chunks (default: 10)
**Parallel Processing**: Multiple files processed simultaneously
**Memory Management**: Efficient image conversion and temporary file cleanup
**S3 Integration**: Signed URLs for efficient image access
**Database Transactions**: Atomic operations for data consistency

## Critical Feature Gap Identified (2025-01-16)

### Estate Auto-Discovery Limitation
**Issue**: The current Discovery Agent requires manual estate ID specification and cannot automatically discover which estate documents belong to. This is a significant limitation that should be discussed with PM as a potential feature request.

**Current Behavior**: 
- User must manually provide `estateId` in command: `yarn run:local file-processing '{"estateId": "estate-id", ...}'`
- Discovery Agent processes documents but cannot determine estate ownership
- No automatic matching of documents to existing estates

**Intended Behavior** (Potential Feature Request):
- Analyze document content to identify deceased person's name
- Search existing estates for matching deceased names  
- Auto-assign documents to correct estate
- Create new estates if no matching estate exists

**Business Impact**: This limitation requires manual intervention and reduces the "discovery" aspect of the Discovery Agent, making it more of a "document processor" than a true discovery system.

**Recommendation**: Discuss with PM whether estate auto-discovery should be added as a future feature request to make the Discovery Agent truly autonomous.

## Status
✅ **Complete**: Comprehensive analysis and documentation created  
✅ **Detailed Flow Analysis**: Complete understanding of Discovery Agent architecture  
✅ **Individual Agent Analysis**: Detailed breakdown of each AI agent's functionality  
✅ **Critical Gap Identified**: Estate auto-discovery limitation documented for PM discussion
🔄 **Next**: Await PM feedback on Jira task requirements  
📋 **Ready**: All materials prepared for task AE-1206 deliverables  

---

*This log documents the complete Discovery Agent analysis session and provides context for future work on this system.*


