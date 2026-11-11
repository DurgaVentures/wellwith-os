# WELLWITH OS — isolated Firebase Emulator testing

## Safety boundary

This repository is pinned to the Firebase demo project `demo-wellwith-os` through `.firebaserc`.
The demo project ID is intentionally not a real Firebase project. Emulator commands below must be run with that project ID. Do not replace it with the production project ID.

**Important:** the Phase 1 POS code in `index.html` has not yet been switched to the emulator. Do not test a sale from the current page until an explicit emulator connection mode is added and verified. Opening the normal production page or using the production Firebase configuration can write to live data.

## Prerequisites

- Node.js LTS
- Java runtime supported by the Firebase Emulator Suite
- Firebase CLI: `npm install --global firebase-tools`

## Start isolated emulators

From the repository root:

```bash
firebase emulators:start --project demo-wellwith-os --only auth,firestore
```

Emulator endpoints:
- Authentication: http://127.0.0.1:9099
- Firestore: http://127.0.0.1:8080
- Emulator UI: http://127.0.0.1:4000

## Prepare test identity and data

1. Open the Emulator UI and create a test Authentication user.
2. In the Firestore emulator, create `users/{test-auth-uid}` with `active: true` and a test role.
3. Create test-only documents in `settings/pos`, `products`, and `inventoryMovements`.
4. Use clearly synthetic product names and IDs. Do not import production exports or customer data.
5. Keep the emulator running while executing browser tests.

The rules in `firestore.rules` reproduce the access model described in the currently supplied production rules. Compare them against the complete deployed rules before relying on authorization test results.

## Phase 1 test cases

- Valid sale commits sale, invoice counter, customer totals, and one outward movement per product atomically.
- Insufficient stock rejects the sale and leaves all four record groups unchanged.
- Two simultaneous sales competing for the final available unit cannot both commit.
- A retry does not duplicate inventory movements or customer totals.
- Two simultaneous sales do not receive the same invoice number.
- Invalid split payment, invalid GSTIN format, missing HSN/GST configuration, inactive user, and offline state reject before commit.
- A forced Firestore transaction error leaves the UI cart intact and creates no local success receipt.

## Current status

This is emulator configuration and a test plan only. The app's explicit emulator connection switch, deterministic seed script, automated browser tests, and transaction test execution are still pending. No production Firebase data is changed by these files.
