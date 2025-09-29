# AE-1222: Ensure Integrity of Default Box Folder Structure in Estates

## Status: ✅ PM Approved Scope – Proceed with Implementation

## Executive Summary

**PM DECISION**: Proceed acknowledging Box cannot block folder renames and Care Team members may manually create folders under `Discovery Agent Input`. Implementation will focus on locking move/delete actions and enforcing rules through the Estate Manager front end.

**Critical Constraint**: Box folder locks only prevent `move` and `delete`. No API support exists for renaming restrictions across all Box SDKs.

### Expected vs Actual Folder Structure

**Expected (from requirements):**
- Executor Documents
- Internal Care Team Documents  
- Beneficiary Documents
- Discovery Agent Input
- Discovery Agent Output

**Actual (from Box workflow):**
- Executor Documents ✅ **EXPECTED**
- Beneficiary Documents ✅ **EXPECTED**
- Internal Care Team Documents ✅ **EXPECTED**
- Discovery Agent Input ✅ **EXPECTED** (created by Discovery Agent)
- Discovery Agent Output ✅ **EXPECTED** (created by Discovery Agent)

**Note**: "Sorted Files" and "Files To Be Sorted" are artifacts from an older workflow and are not part of the current standard folder structure.

### Folder Structure Clarification

**RESOLVED**: The folder structure is now correctly understood. All expected folders are created as intended.

**Current Understanding**:
- **Standard Folders**: Executor Documents, Beneficiary Documents, Internal Care Team Documents are created by Box workflow
- **Discovery Agent Folders**: Discovery Agent Input and Discovery Agent Output are created by the Discovery Agent (lambdas)
- **Legacy Artifacts**: "Sorted Files" and "Files To Be Sorted" are from an older workflow and can be ignored
- **Implementation**: Focus folder locking on the 5 standard folders that are actually created

## Implementation Questions Resolved

✅ **Scope Question**: Lock ALL folders (main + 5 default + 45+ subfolders)  
✅ **Error Handling**: Option B (Permissive) - log error but continue  
✅ **Performance**: Accept 2x API call overhead for security benefits  
✅ **Notification Strategy**: Defer to AE-1161 ticket for error notification system  
✅ **Internal Care Team Coverage**: `Internal Care Team Documents` now managed alongside other default folders  

## Box API Limitation Analysis

**Root Cause**: Box folder locks only support `move` and `delete` operations. No `rename` operation is available in the API.

**Impact**: Users with editor permissions can rename folders despite folder locks being in place.

**Investigation Completed**:
- ✅ Reviewed current box-typescript-sdk-gen implementation
- ✅ Analyzed Box API schema and documentation  
- ✅ Reviewed newer box-node-sdk for additional capabilities
- ✅ Searched for alternative Box API methods to restrict folder operations
- ✅ Confirmed limitation exists across all Box SDK versions

**Potential Solutions** (awaiting PM decision):
1. **Accept the limitation** - Focus on preventing move/delete operations only
2. **Change user permissions** - Switch to "viewer" role but lose file editing capabilities  
3. **Implement monitoring** - Build system to detect and potentially revert unauthorized renames  

## Implementable Scope
- **Move/Delete Locks**: Lock estate root, default folders (including `Internal Care Team Documents`), and Discovery Agent–generated folders against move/delete operations across alix-api and lambdas.
- **Default Structure Seeding**: Ensure the executor base tree is created and locked during estate provisioning; Discovery Agent Output mirrors executor structure when files are discovered.
- **Care Team Through Estate Manager**: Direct Care Team actions through the front end so we control which operations are available; still allow new folders where permitted.
- **Customer Restrictions**: Keep customers limited to `My Uploads` via restricted roles and app-driven uploads.
- **Automation Guardrails**: Prevent our services from creating subfolders inside `Discovery Agent Input`; manual additions remain possible but discouraged.
- **Legacy Estates**: Leave existing estates untouched unless a future backfill is approved.

## Known Limitations (Accepted)
- **Folder Renaming**: Cannot be prevented technically; rely on UI constraints and monitoring.
- **Manual Subfolders in Discovery Agent Input**: Care Team can still add subfolders directly in Box if given access; mitigated by front-end workflow guidance (UI mitigation deferred to follow-up task).
- **Box Workflow Reliability**: Need to confirm how Box Relay behaves when asked to create folders that already exist; options include pausing before verification or eliminating the workflow entirely.

## Next Steps

1. **Implement** folder locking updates in alix-api (move/delete focus) and add any missing executor subfolders during provisioning.
2. **Integrate** identical locking logic into Discovery Agent lambdas for newly created folders.
3. **Investigate** Box Relay duplicate-folder behavior and choose mitigation (pause vs. remove workflows).
4. **Test** end-to-end with new estate creation plus Discovery Agent runs.
5. **Deploy & Monitor** once feature parity confirmed.

*Front-end updates for Care Team workflows will be handled in a future task.*

## Technical Details

- **Box API**: Successfully tested folder creation
- **Authentication**: JWT authentication working correctly
- **Environment**: Test environment (`box-test`) operational
- **Folder Creation**: Confirmed Box workflow creates 3 standard folders + 2 Discovery Agent folders = 5 total folders

---

**Last Updated**: December 19, 2024  
**Status**: ✅ PM Approved Scope – Proceed with Implementation  
**Next Action**: Implement move/delete locking and front-end controls under accepted limitations
