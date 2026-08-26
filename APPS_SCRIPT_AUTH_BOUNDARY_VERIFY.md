# Apps Script authentication-boundary verification

This document covers the manual checks required for the `security/apps-script-auth-boundary` change. Run these checks against a non-production copy of the Google Sheet/Apps Script deployment first. Record the deployed URL, test technician/admin accounts used, start/end timestamps, and the relevant Sheet row counts before and after each denial test.

## Preconditions

1. Copy the current Google Sheet and attach/deploy the branch version of `google-apps-script.js` and `appsscript.json` to the copy.
2. Ensure the `Technicians` sheet already exists and contains at least one active `tech` technician and one active `admin` technician with known test PINs. The public login endpoints are intentionally read-only and will no longer create this sheet/header.
3. Run `authoriseWaterOpsServices()` once from the Apps Script editor. This also initializes the session-signing key in Script Properties when missing.
4. Deploy a new web-app version and use its `/exec` URL for testing.
5. Note initial row counts for `Visit Log`, `Closed Loop Log`, `Cost Sync Events`, `Stock Used`, `Cost Settings`, `Cost Snapshots`, `Cost Reports`, `BlueRiiot Readings`, `BlueRiiot Devices`, and `BlueRiiot Links`.

## 1. Anonymous protected GET denial

Call a protected action such as `costSnapshot` directly with no `deviceId` session token. Expected: `{ ok: false, authRequired: true }` (JSONP when a callback is supplied), and no protected data is returned.

Repeat with `blueRiiotSnapshot`, `blueRiiotDevices`, and `blueRiiotHistory`. Expected: all denied before BlueRiiot or Sheet work is performed.

## 2. Anonymous POST denial and no Sheet mutation

POST a syntactically valid `costSync` payload without a valid signed session in `deviceId`. Expected: `{ ok: false, authRequired: true }` when inspected with a direct HTTP client. Confirm every Sheet row count noted in Preconditions is unchanged.

Repeat with a valid-looking `closedLoopVisit` payload and a generic/other visit payload. Confirm no Sheet row count changes.

## 3. Valid technician read/write success

From the WaterOps UI, select the active test technician and enter the correct PIN. Confirm the login response succeeds and the browser's existing `mbs-cost-sync-device-id-v1` value is replaced with a token beginning `wops1.`.

While the session is valid:

- refresh site-cost data (`costSnapshot`) and confirm it succeeds;
- perform a harmless test write against the copied Sheet (for example a clearly labelled test cost-sync/visit record) and confirm exactly the expected row(s) are created;
- confirm the stored Sheet `deviceId` is the underlying non-secret device identifier, not the `wops1.` bearer token.

## 4. Technician denied admin/setup/config operations

While signed in as the test `tech`, call each admin action with the valid session token:

- `blueRiiotDiagnostics`
- `blueRiiotLinkPool`
- `blueRiiotRefreshHistory`
- `setupBlueRiiotSheets`
- `setupTechnicianSheet`

Expected: `{ ok: false, forbidden: true }`. Confirm no Script Property, Sheet structure, BlueRiiot link, or history mutation occurs.

## 5. Admin success

Sign out/clear the technician session, sign in as the active `admin`, and confirm an admin-only read such as `blueRiiotDiagnostics` passes authorization. On the copied environment only, run one reversible setup/config action and confirm it succeeds. Restore test data if needed.

## 6. Tampered token denial

Copy a valid `wops1.` token and change one character in the payload or signature. Call `costSnapshot` with the tampered token as `deviceId`. Expected: authentication denial and no protected data.

POST a valid-looking write with the tampered token as `deviceId`. Expected: denial and no Sheet mutation.

## 7. Expired token denial

For testing only, temporarily reduce `WATEROPS_SESSION_TTL_SECONDS` in the non-production Apps Script copy to a very small value (for example 10 seconds), redeploy, log in, wait beyond expiry, then call a protected GET and POST. Expected: both are denied. For the GET path, confirm the browser clears its cached technician auth/session state and requires verification again. Restore the production TTL before final deployment.

## 8. Unknown action denial

Call an unrecognized GET action such as `action=notARealWaterOpsAction`. Expected: `{ ok: false, error: 'Unknown action.' }` and no work is performed.

## 9. Existing login/logout/session-expiry behavior

Confirm normal user flow after deployment:

1. Technician list still loads before login.
2. Correct PIN logs in and existing post-login screens/actions continue to work.
3. Incorrect PIN does not create a session.
4. Changing technician clears the previous UI auth and requires the appropriate verification flow.
5. When the signed session expires, protected GETs fail closed, the cached UI auth/session values are cleared, and the user is required to verify again.
6. A valid new login creates a fresh signed session and normal reads/writes resume.

## Evidence to retain

Keep screenshots or notes showing the denial responses, before/after Sheet row counts for denied POSTs, one successful technician operation, one denied tech admin operation, one successful admin operation, tampered/expired-token denials, and the final Apps Script deployment version used.