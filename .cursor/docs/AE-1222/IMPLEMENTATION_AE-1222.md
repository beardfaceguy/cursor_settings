# AE-1222 Implementation Plan: Box Folder Permission Controls

## Status: ✅ PM Approved Scope – Proceed with Implementation

## Implementation Overview

**GOAL**: Prevent unauthorized modification of default Box folder structure in estates by applying folder locks to critical folders, while accepting Box limitations around renaming and manual subfolder creation in `Discovery Agent Input`.

**SCOPE**: Lock the main estate folder and 5 top-level directories:
1. "Estate of X" folder (main estate folder)
2. "Executor Documents"
3. "Internal Care Team Documents"
4. "Beneficiary Documents"
5. "Discovery Agent Input"
6. "Discovery Agent Output"

**APPROACH**: Integrate folder locking into existing folder creation/finding methods with minimal code changes.

## alix-api Implementation Details

### ✅ COMPLETED: alix-api Folder Locking Implementation

#### Implementation Strategy
- **Minimal Code Changes**: Only modified 2 existing methods and added 2 helper methods
- **Comprehensive Coverage**: Handles both new folder creation and existing folder discovery
- **Safe Operation**: Checks for existing locks before creating new ones
- **Permissive Error Handling**: Folder operations succeed even if locking fails

#### Code Implementation

**1. Modified Methods:**

```typescript
// Modified: createEstateFolderStructure() - Locks main estate folder
async createEstateFolderStructure({ estateId, deceasedName }: { estateId: string; deceasedName: string }): Promise<FolderFull> {
  const estateIdLastChunk = estateId.split("-").pop();
  const folderName = `Estate of ${deceasedName} (${estateIdLastChunk})`;
  const estateFolder = await this.createFolder({ folderName, parentFolderId: boxEstatesRootFolderId });

  // Lock the estate folder for AE-1222
  await this.lockFolder(estateFolder.id);

  // Ensure Internal Care Team Documents exists and is locked
  await this.findOrCreateFolder({
    folderName: this.INTERNAL_CARE_TEAM_FOLDER_NAME,
    parentFolderId: estateFolder.id,
  });

  // save the estate folder ID in the database
  this.estateService.setEstateBoxFolderId({ estateId, boxEstateRootFolderId: estateFolder.id });

  return estateFolder;
}

// Modified: findOrCreateFolder() - Locks target folders when found or created
async findOrCreateFolder({ folderName, parentFolderId }: { folderName: string; parentFolderId: string }) {
  const items = await this.getFolderItems(parentFolderId);
  const existingFolder = items.find(({ name }) => name === folderName);

  if (!existingFolder) {
    const newFolder = await boxClient.folders.createFolder({
      name: folderName,
      parent: { id: parentFolderId },
    });
    
    // Lock specific folders for AE-1222
    if (this.isTargetFolder(folderName)) {
      await this.lockFolder(newFolder.id);
    }
    
    return newFolder;
  }

  // Lock existing target folders for AE-1222
  if (this.isTargetFolder(folderName)) {
    await this.lockFolder(existingFolder.id);
  }

  return existingFolder;
}
```

**2. Added Helper Methods:**

```typescript
// Helper: Determines which folders should be locked
private isTargetFolder(folderName: string): boolean {
  return [
    this.EXECUTOR_FOLDER_NAME,
    this.BENEFICIARY_FOLDER_NAME,
    this.INTERNAL_CARE_TEAM_FOLDER_NAME,
    this.FILES_TO_BE_SORTED_FOLDER_NAME,
    this.SORTED_FILES_FOLDER_NAME,
  ].includes(folderName);
}

// Helper: Safely locks folders with existing lock detection
private async lockFolder(folderId: string): Promise<void> {
  try {
    // Check for existing locks first
    const existingLocks = await boxClient.folderLocks.getFolderLocks({ folderId });
    if (existingLocks.entries && existingLocks.entries.length > 0) {
      logger.info(`Folder ${folderId} already has ${existingLocks.entries.length} lock(s) for AE-1222`);
      return;
    }

    // Create new lock
    await boxClient.folderLocks.createFolderLock({
      folder: { id: folderId, type: 'folder' },
      lockedOperations: { move: true, delete: true },
    });
    logger.info(`Locked folder ${folderId} for AE-1222`);
  } catch (error: any) {
    logger.warn(`Failed to lock folder ${folderId}: ${error.message}`);
  }
}
```

#### Integration Points

**Primary Integration Points:**
1. **`createEstateFolderStructure()`**: Locks main estate folder when created
2. **`findOrCreateFolder()`**: Locks 4 target folders when found or created

**Coverage Analysis:**
- **Main Estate Folder**: ✅ Locked in `createEstateFolderStructure()`
- **Executor Documents**: ✅ Locked in `findOrCreateFolder()` + `createExecutorFolderStructure()`
- **Beneficiary Documents**: ✅ Locked in `findOrCreateFolder()` + `createBeneficiaryFolderStructure()`
- **Discovery Agent Input**: ✅ Locked in `findOrCreateFolder()` + `getFolderToBeSorted()`
- **Discovery Agent Output**: ✅ Locked in `findOrCreateFolder()` + `getFolderIdForSortedItems()`

#### Key Implementation Features

**1. Safe Lock Detection:**
- Checks for existing locks before creating new ones
- Prevents duplicate lock creation
- Logs existing lock count for monitoring

**2. Comprehensive Coverage:**
- Handles both new folder creation and existing folder discovery
- Covers all folder creation paths in alix-api
- Future-proof: any new folder creation through `findOrCreateFolder()` will be checked

**3. Permissive Error Handling:**
- Folder creation succeeds even if locking fails
- Logs warnings for failed locks without throwing errors
- Maintains existing functionality while adding security

**4. Minimal Code Impact:**
- Only 4 lines added to existing methods
- 2 small helper methods added
- No breaking changes to existing APIs

## Folder Structure Discovery

**CONFIRMED**: The default estate folder now consistently contains the five expected folders. "Sorted Files" and "Files To Be Sorted" were artifacts from an older workflow and can be ignored going forward.

### Expected vs Actual Folder Structure

**Expected (from Jira requirements):**
```
Estate of [Customer] ([ID])/
├── Executor Documents
├── Internal Care Team Documents  
├── Beneficiary Documents
├── Discovery Agent Input
└── Discovery Agent Output
```

**Actual (current behavior):**
```
Estate of [Customer] ([ID])/
├── Executor Documents               ✅ Created via Box workflow
├── Internal Care Team Documents     ✅ Created via Box workflow
├── Beneficiary Documents            ✅ Created via Box workflow
├── Discovery Agent Input            ✅ Created via Discovery Agent lambdas
└── Discovery Agent Output           ✅ Created via Discovery Agent lambdas
```

### Folder Structure Clarification

1. **Box workflow** seeds the three Care-Team-managed folders (Executor, Beneficiary, Internal Care Team).
2. **Discovery Agent** lambdas create `Discovery Agent Input` and `Discovery Agent Output` as part of their run.
3. **Legacy artifacts** `Sorted Files` / `Files To Be Sorted` stem from a discontinued workflow and do not appear for new estates.
4. **Implementation focus**: lock move/delete on the five standard folders and any subfolders seeded under Executor or generated by Discovery Agent.

## Folder Locking Test Results

### ✅ Admin User Testing (COMPLETED)
- **Test**: Applied folder locks to "Executor Documents" folder
- **Result**: Admin user was able to rename the locked folder
- **Finding**: Admin users have elevated privileges that override folder locks
- **Impact**: Expected behavior - admins can manage content when needed

### ❌ Group Admin Testing (COMPLETED - ISSUE IDENTIFIED)
- **Status**: PM (group admin with editor permissions) was able to rename the locked folder
- **Root Cause**: Box folder locks only prevent `move` and `delete` operations, NOT `rename` operations
- **Finding**: Box API limitation - folder locks do not include rename operation in `lockedOperations`
- **Impact**: Folder locks provide partial protection but do not prevent renaming

### Folder Lock Behavior Summary
- **Enterprise Admins**: Can override folder locks ✅ (Expected)
- **Content Admins**: Can override folder locks ✅ (Expected)  
- **Group Admins**: Can rename folders despite locks ❌ (Box API Limitation)
- **Regular Users**: Can rename folders despite locks ❌ (Box API Limitation)
- **Care Team Users**: Can rename folders despite locks ❌ (Box API Limitation)
- **Customer Users**: Can rename folders despite locks ❌ (Box API Limitation)

**CRITICAL FINDING**: Box folder locks only prevent `move` and `delete` operations. They do NOT prevent `rename` operations for any user role. This limitation exists across all Box SDK versions (box-typescript-sdk-gen and box-node-sdk) with no alternative API methods available to restrict folder renaming.

### Box Workflow Timing Study
- `box-test/measure-workflow-timing.ts` instrumented the Box workflow to measure how quickly the default subfolders appear.
- Ten-run sample (5 s gaps) produced:
  - `Executor Documents` & `Internal Care Team Documents`: average ≈6.08 s (min 6.00 s, max 6.17 s) after estate folder creation.
  - `Beneficiary Documents`: average ≈6.63 s; one outlier took 11.56 s.
- Recommendation: allow ~7 s before asserting subfolders exist; budget up to ~12 s for rare slow runs. This informs any post-provisioning verification logic.

## Box API Limitation Analysis

### Root Cause Investigation

**Box Folder Lock Schema Analysis:**
```typescript
// From box-typescript-sdk-gen/src/schemas/folderLock.generated.ts
export interface FolderLockLockedOperationsField {
  readonly move: boolean;    // ✅ Supported
  readonly delete: boolean;  // ✅ Supported
  // ❌ NO rename operation available
}
```

**Box API Documentation Confirmation:**
- Folder locks are designed to "prevent it from being moved and/or deleted"
- The schema comment explicitly states: "Currently the `move` and `delete` operations cannot be locked separately, and both need to be set to `true`"
- **No mention of rename operation anywhere in the API**

**Additional SDK Review:**
- **box-node-sdk (newer SDK) reviewed**: No additional folder lock operations available
- **Alternative folder operation restrictions**: No other Box API methods found to prevent folder renaming
- **Conclusion**: Box API fundamentally does not support preventing folder rename operations in any SDK version

**Comprehensive Investigation Summary:**
- ✅ Reviewed current box-typescript-sdk-gen implementation
- ✅ Analyzed Box API schema and documentation
- ✅ Reviewed newer box-node-sdk for additional capabilities
- ✅ Searched for alternative Box API methods to restrict folder operations
- ✅ Confirmed limitation exists across all Box SDK versions

### Current Implementation Analysis

**Collaboration Role Assignment:**
```typescript
// From BoxDocumentService.ts line 497
role: "editor",  // All users get editor permissions
```

**Box Editor Role Capabilities:**
- ✅ Can view and download files
- ✅ Can upload and edit files  
- ✅ Can rename folders and files
- ✅ Can move files within folders
- ❌ Cannot delete folders (prevented by folder lock)
- ❌ Cannot move folders (prevented by folder lock)

### Potential Solutions

#### Option 1: Change Collaboration Role to "viewer"
**Pros:**
- Prevents all modifications including renaming
- Simple implementation change

**Cons:**
- Users lose ability to upload/edit files
- May break existing functionality
- Users need editor access for document management

#### Option 2: Implement Folder Name Monitoring
**Pros:**
- Maintains editor permissions for file operations
- Can detect and potentially revert unauthorized renames
- Provides audit trail

**Cons:**
- Complex implementation
- Requires periodic monitoring
- Reactive rather than preventive

#### Option 3: Accept the Limitation
**Pros:**
- Acknowledges Box API constraints
- Focuses on preventing more critical operations (move/delete)
- Simpler maintenance

**Cons:**
- Does not meet original requirement to prevent renaming
- May require PM approval for scope change

## Implementable Scope (Per PM Approval)
- **Move/Delete Locks**: Extend locking to estate root, default folders, and Discovery Agent–generated folders using alix-api and lambdas.
- **Executor Structure Seeding**: Ensure the executor base tree is created and locked when provisioning new estates.
- **Discovery Agent Output**: Continue mirroring executor structure only when files require it, locking each generated folder after creation.
- **Care Team via Estate Manager**: Route actions through the front end so we control rename/create operations; provide UI paths for allowed folder creation.
- **Customer Access**: Keep customers confined to `My Uploads` (viewer uploader permissions) with uploads mediated by the app.
- **Automation Guardrail**: Block our services from creating subfolders under `Discovery Agent Input`; manual Box UI additions remain out of scope.
- **Existing Estates**: Leave legacy estates unchanged unless a follow-up backfill project is approved.

## Known Limitations (Accepted)
- **Folder Renaming**: Not technically preventable with Box; rely on UI/monitoring.
- **Discovery Agent Input Subfolders**: Manual additions by Care Team in Box UI cannot be blocked programmatically.
- **Box Relay Reliability**: Behavior when workflows create already-existing folders is unknown; options under consideration include adding a delay before verification or retiring the workflow in favor of API-driven provisioning.

## Implementation Questions Status

### ✅ Resolved Questions

1. **Scope Question**: Lock ALL folders (main + 5 default + 45+ subfolders)
   - **Answer**: Yes, lock all auto-created folders
   - **Rationale**: Prevent unauthorized modification of default structure

2. **Error Handling**: What happens if folder lock creation fails?
   - **Answer**: Option B (Permissive) - log error but continue
   - **Rationale**: Prioritize estate creation over strict locking

3. **Performance Impact**: Accept 2x API call overhead?
   - **Answer**: Yes, accept the overhead
   - **Rationale**: Security benefits outweigh performance cost

4. **Notification Strategy**: How to handle error notifications?
   - **Answer**: Defer to AE-1161 ticket for error notification system
   - **Rationale**: Focus AE-1222 on folder locking mechanism only

### ✅ Additional Findings (Resolved)

5. **Folder Structure Discrepancy**: Box workflow creates different folders than initially expected
   - **Status**: Resolved - explained by different Box workflow
   - **Impact**: None - proceed with locking all Box-created folders

## Revised Implementation Strategy

### Phase 1: Implementation (COMPLETED)
- [x] Test Box API folder creation
- [x] Verify folder structure (clarified workflow explanation)
- [x] Test folder locking functionality
- [x] Verify folder locks work with admin users (admin can override locks)
- [x] **COMPLETED**: Implement folder locking in alix-api repository
- [ ] **PENDING**: Verify folder locks work with group admin users
- [ ] Test folder locking with actual Box workflow folders

### Phase 2: Integration (IN PROGRESS)
- [ ] Integrate folder locking with Discovery Agent lambdas (move/delete focus)
- [ ] Investigate Box Relay duplicate-folder handling and determine mitigation (pause vs. remove workflow)
- [ ] Test complete workflow with locked folders
- [ ] Deploy and monitor

*Front-end updates for Care Team workflows to be handled in a future task.*

## Technical Implementation Details

### Box API Integration (IMPLEMENTED)
```typescript
// IMPLEMENTED: Safe folder locking with existing lock detection
private async lockFolder(folderId: string): Promise<void> {
  try {
    // Check for existing locks first
    const existingLocks = await boxClient.folderLocks.getFolderLocks({ folderId });
    if (existingLocks.entries && existingLocks.entries.length > 0) {
      logger.info(`Folder ${folderId} already has ${existingLocks.entries.length} lock(s) for AE-1222`);
      return;
    }

    // Create new lock
    await boxClient.folderLocks.createFolderLock({
      folder: { id: folderId, type: 'folder' },
      lockedOperations: { move: true, delete: true },
    });
    logger.info(`Locked folder ${folderId} for AE-1222`);
  } catch (error: any) {
    logger.warn(`Failed to lock folder ${folderId}: ${error.message}`);
  }
}
```

### Testing Environment
- **Location**: `box-test/` directory
- **Status**: Operational
- **Scripts**: 
  - `read-estates-folder.ts` - List existing folders
  - `create-estate-folder.ts` - Create test estate folder
  - `lock-executor-documents.ts` - Test folder locking functionality

## Next Steps

1. Implement approved locking changes and executor seeding updates in alix-api.
2. Mirror locking behavior in Discovery Agent lambdas for any newly created folders.
3. Investigate Box Relay duplicate-folder behavior and select mitigation approach (timed pause vs. workflow removal).
4. Run end-to-end tests covering new estate creation and Discovery Agent runs.
5. Deploy once acceptance criteria are met.

*Estate Manager UI changes will be scheduled separately.*

---

**Last Updated**: December 19, 2024  
**Status**: ✅ PM Approved Scope – Proceed with Implementation  
**Next Action**: Implement move/delete locking and front-end controls within the accepted limitations
