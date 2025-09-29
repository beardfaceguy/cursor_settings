# AE-1252 Implementation Notes: Estate ↔ Box Folder Reconciliation

## Goal & Scope

***GOAL***: Review every estate in production, capture their `boxEstateRootFolderId`, confirm the corresponding “Estate of X” folder exists in Box, and create/store the ID when missing.

- Include estates even if `boxEstateRootFolderId` is `null` (those must be flagged).
- For populated IDs, confirm the folder is present via Box API; note missing folders.
- Produce a consolidated report for follow-up/remediation.

## Current Findings

### Database Storage
- `EstateService.setEstateBoxFolderId` updates `Estate.boxEstateRootFolderId` via Prisma.
- Prisma schema: `boxEstateRootFolderId String?` (migration `20241016231316_add_folder_ids_to_estate_record`).
- Similar helpers exist for executor/beneficiary folders (`boxExecutorFolderId`, `boxBeneficiaryFolderId`).

### Reporting Script (`alix-api/scripts/estate-box-audit.ts`)
- Added a manual script to audit estate/Box folder linkage.
- Workflow:
  1. Fetch estates with Prisma (`findMany`) including `boxEstateRootFolderId`.
  2. Initialize shared Box JWT client (`initializeBoxClient`).
  3. For each estate ID, call `boxClient.folders.getFolderById` (requesting `path_collection`).
  4. Compile results into JSON, flagging:
     - `missing_id`: estate has no folder ID stored.
     - `missing_in_box`: ID stored but Box returns 404.
     - `error`: other Box/API failures.
  5. Output `estate-box-audit-output.json` with summary counts and record detail.
- Run manually via `npx ts-node -r tsconfig-paths/register scripts/estate-box-audit.ts` (no automation/UI hooks).

### Box Workflow Timing Study
- Used `box-test/measure-workflow-timing.ts` to profile folder creation latency after estate creation.
- Ten-run sample (5s gaps) indicated:
  - `Executor Documents` & `Internal Care Team Documents` arrive ≈6.0–6.2 s post-creation (avg ~6.08 s).
  - `Beneficiary Documents` typically similar, but one outlier took 11.56 s (avg ~6.63 s).
- Recommendation: wait ~7 s before checking for Box workflow folders; allow up to ~12 s for rare slow cases.

### Existing Script Patterns
- Other maintenance scripts (`backfill-estate-google-groups`, `ssn-fields-encrypt-existing-values`, S3 correction scripts, etc.):
  - Manual invocation, typically via `package.json` scripts or direct `ts-node` runs.
  - Console logging, optional flags (`--dry-run`, `--limit`).
  - No automated scheduling or UI entry points; results consumed via console or generated files.

## Next Steps / Considerations
- Evaluate whether audit script should also write CSV/Slack notifications for ops.
- Decide on remediation workflow for estates flagged `missing_id` or `missing_in_box` (manual fix vs. scripted backfill).