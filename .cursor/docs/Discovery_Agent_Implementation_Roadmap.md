# Discovery Agent Implementation Roadmap

## Executive Summary

Based on comprehensive analysis of the current codebase, Figma design specifications, and direct access to live Figma designs, this roadmap outlines the critical gaps and implementation priorities for the Discovery Agent system.

## Current State Assessment

### ✅ What's Working Well
- **Solid AI Pipeline**: Multi-stage document processing with specialized agents
- **Robust Architecture**: LangGraph workflows, chunked processing, distributed locking
- **Comprehensive Extraction**: Assets, debts, contacts, obligations extraction
- **Error Handling**: Graceful failure handling and comprehensive logging
- **Database Integration**: Proper relationships and event tracking

### ❌ Critical Gaps (Visual Evidence from Figma)
1. **Missing Human Review System** - CoPilot interface shown in designs
2. **No Document Organization** - Structured folder system in designs
3. **Missing Care Team Workflow** - Email/Slack integration in designs
4. **No DoD Value Logic** - Separate value handling in designs
5. **Missing External Integration** - Multiple data sources in designs

## Implementation Phases

### Phase 1: Critical Fixes (P1 - Immediate)
**Timeline**: 2-4 weeks
**Priority**: Fix incorrect behavior and missing core features

#### 1.1 Fix Incorrect Data Extraction
- **Issue**: Extracts beneficiary bank info from retirement forms
- **Solution**: Enhance fact identification prompts with context awareness
- **Files**: `src/tasks/file-processing/agents/factIdentifier.ts`
- **Impact**: Prevents incorrect data pollution

#### 1.2 Implement DoD Value Logic
- **Issue**: No distinction between date of death and current values
- **Solution**: Add DoD value extraction and storage
- **Files**: Asset/debt extractors, database schema
- **Impact**: Accurate estate valuation

#### 1.3 Enhance Duplicate Detection
- **Issue**: Simple primary key matching
- **Solution**: Vector-based LLM reasoning for semantic similarity
- **Files**: `src/tasks/file-processing/engine/saveFacts/`
- **Impact**: Better duplicate prevention

#### 1.4 Add Document Naming Convention
- **Issue**: No standardized naming
- **Solution**: Implement "{Institution-NameDocument-Type-MMDD-YYYY}" format
- **Files**: Document processing pipeline
- **Impact**: Consistent file organization

### Phase 2: Core Features (P2 - Short-term)
**Timeline**: 4-8 weeks
**Priority**: Build missing systems shown in Figma designs

#### 2.1 Build CoPilot Review Interface
- **Requirement**: Human review system for care team
- **Components**: Review UI, approval workflow, feedback system
- **Integration**: Estate Manager frontend
- **Impact**: Quality control and human oversight

#### 2.2 Implement Folder Organization
- **Requirement**: Structured subfolder system
- **Structure**: Assets/Bank Accounts, Debts/Home Loans, Obligations/Utilities
- **Implementation**: Box API integration, folder creation logic
- **Impact**: Organized document storage

#### 2.3 Create Care Team Workflow
- **Requirement**: Email notifications, Slack integration, task creation
- **Components**: Notification system, Slack webhooks, task management
- **Integration**: External services (email, Slack, task system)
- **Impact**: Streamlined care team operations

#### 2.4 Build Automated Testing Framework
- **Requirement**: QA system with sample document sets
- **Components**: Test data sets, comparison logic, performance metrics
- **Implementation**: Separate testing environment, automated validation
- **Impact**: Quality assurance and regression prevention

### Phase 3: Advanced Features (P3 - Medium-term)
**Timeline**: 8-16 weeks
**Priority**: External integration and advanced capabilities

#### 3.1 External Data Source Integration
- **Sources**: MissingMoney.com, credit reports, SSA, IRS
- **Implementation**: API integrations, data aggregation
- **Impact**: Comprehensive estate discovery

#### 3.2 Advanced Reasoning Engine
- **Capability**: Context-aware fact identification
- **Implementation**: Enhanced AI prompts, reasoning chains
- **Impact**: More accurate data extraction

#### 3.3 Real-time Processing
- **Capability**: Live updates and progress tracking
- **Implementation**: WebSocket connections, progress indicators
- **Impact**: Better user experience

#### 3.4 Performance Optimization
- **Focus**: Scalability and efficiency
- **Implementation**: Caching, parallel processing, resource optimization
- **Impact**: Handle larger document sets

## Technical Implementation Details

### Database Schema Changes
```sql
-- Add DoD value fields to assets and debts
ALTER TABLE "Asset" ADD COLUMN "dateOfDeathValue" DECIMAL(10,2);
ALTER TABLE "Debt" ADD COLUMN "dateOfDeathValue" DECIMAL(10,2);

-- Add document organization fields
ALTER TABLE "RemoteFile" ADD COLUMN "folderPath" TEXT;
ALTER TABLE "RemoteFile" ADD COLUMN "standardizedName" TEXT;
```

### API Endpoints Needed
```typescript
// CoPilot review endpoints
POST /api/discovery/review/approve
POST /api/discovery/review/reject
GET /api/discovery/review/pending

// Document organization endpoints
POST /api/discovery/organize/folder
GET /api/discovery/organize/structure

// External data endpoints
POST /api/discovery/external/missing-money
POST /api/discovery/external/credit-report
```

### Configuration Updates
```typescript
// Add to config.ts
export const config = {
  // ... existing config
  COPILOT_ENABLED: process.env.COPILOT_ENABLED === 'true',
  SLACK_WEBHOOK_URL: process.env.SLACK_WEBHOOK_URL,
  EMAIL_SERVICE_URL: process.env.EMAIL_SERVICE_URL,
  EXTERNAL_DATA_SOURCES: {
    MISSING_MONEY_API_KEY: process.env.MISSING_MONEY_API_KEY,
    CREDIT_REPORT_API_KEY: process.env.CREDIT_REPORT_API_KEY,
  }
};
```

## Success Metrics

### Phase 1 Metrics
- **Data Accuracy**: 95%+ correct extraction (no beneficiary info in wrong places)
- **DoD Values**: 100% of assets/debts have DoD values when available
- **Duplicate Prevention**: 90%+ reduction in duplicate entities
- **File Naming**: 100% of processed files follow naming convention

### Phase 2 Metrics
- **Review Efficiency**: 50%+ reduction in manual review time
- **Organization**: 100% of files organized into proper folders
- **Care Team Satisfaction**: 4.5+ rating on workflow efficiency
- **Test Coverage**: 95%+ test coverage with sample document sets

### Phase 3 Metrics
- **Data Completeness**: 30%+ increase in discovered assets/debts
- **Processing Speed**: 50%+ improvement in processing time
- **User Experience**: 4.5+ rating on real-time updates
- **System Performance**: Handle 10x current document volume

## Risk Mitigation

### Technical Risks
- **AI Model Changes**: Version control and rollback procedures
- **External API Failures**: Graceful degradation and retry logic
- **Performance Issues**: Load testing and monitoring
- **Data Quality**: Validation and error handling

### Business Risks
- **User Adoption**: Training and documentation
- **Compliance**: Audit trails and data protection
- **Scalability**: Infrastructure planning
- **Cost Control**: Resource monitoring and optimization

## Resource Requirements

### Development Team
- **Backend Developer**: 1 FTE for core implementation
- **Frontend Developer**: 0.5 FTE for CoPilot interface
- **DevOps Engineer**: 0.25 FTE for infrastructure
- **QA Engineer**: 0.5 FTE for testing framework

### Timeline Summary
- **Phase 1**: 2-4 weeks (Critical fixes)
- **Phase 2**: 4-8 weeks (Core features)
- **Phase 3**: 8-16 weeks (Advanced features)
- **Total**: 14-28 weeks for complete implementation

## Next Steps

1. **Stakeholder Review**: Present roadmap to PM and technical team
2. **Resource Allocation**: Assign development team members
3. **Phase 1 Kickoff**: Begin critical fixes immediately
4. **Design Review**: Validate CoPilot interface design with care team
5. **Testing Strategy**: Develop comprehensive testing approach

---

*Implementation roadmap completed: 2025-01-16*
*Based on: Codebase analysis, Figma designs, and visual comparison*

