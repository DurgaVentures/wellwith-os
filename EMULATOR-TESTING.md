# WELLWITH OS — isolated Firebase Emulator testing

## Safety boundary

This repository is pinned to the Firebase demo project `demo-wellwith-os` through `.firebaserc`.
The demo project ID is intentionally not a real Firebase project. Emulator commands below must be run with that project ID. Do not replace it with the production project ID.

**Safety:** emulator mode is enabled only by `?environment=emulator` on `localhost` or `127.0.0.1`. The app refuses that flag on hosted domains. Opening the normal production URL uses the production Firebase configuration. Never use production customer or sales data in the emulator.

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
3. Create test-only documents in `settings/pos`, `products`, and `inventoryMovements`. The `returns` collection is also enabled by the emulator rules.
4. Use clearly synthetic product names and IDs. Do not import production exports or customer data.
5. Keep the emulator running while executing browser tests.

## Open the app in emulator mode

In a second terminal, from the repository root, serve the app locally:

```bash
python -m http.server 5500
```

Open:

```
http://127.0.0.1:5500/index.html?environment=emulator
```

The app connects to Auth emulator port 9099 and Firestore emulator port 8080, changes the Firebase project ID to `demo-wellwith-os`, and displays a red emulator-only banner. The flag is accepted only on `localhost` or `127.0.0.1`. A hosted page or normal URL does not enable emulator mode. Do not use the ordinary production URL for emulator tests.

The rules in `firestore.rules` reproduce the access model described in the currently supplied production rules. Compare them against the complete deployed rules before relying on authorization test results.

## Phase 1 test cases

- Valid sale commits sale, invoice counter, customer totals, and one outward movement per product atomically.
- Insufficient stock rejects the sale and leaves all four record groups unchanged.
- Two simultaneous sales competing for the final available unit cannot both commit.
- A retry does not duplicate inventory movements or customer totals.
- Two simultaneous sales do not receive the same invoice number.
- Invalid split payment, invalid GSTIN format, missing HSN/GST configuration, inactive user, and offline state reject before commit.
- A forced Firestore transaction error leaves the UI cart intact and creates no local success receipt.

## Returns & Exchanges test cases

- Search and select an existing test sale by invoice, customer, or mobile.
- Return a partial quantity, then confirm the remaining returnable quantity is reduced.
- Attempt to return more than the purchased quantity; the transaction must be rejected.
- Post a saleable return; verify a `return` movement increases available stock.
- Post a damaged/expired return; verify a `return_damaged` movement is recorded but does not increase saleable stock.
- Attempt two returns concurrently against the same remaining quantity; only a valid non-overlapping quantity may commit.
- Exchange an item for a replacement; verify replacement stock is checked and an `outward` movement is committed with the return record.
- Force a transaction failure; verify no return or inventory movement is partially written.
- Confirm the original sale document is unchanged.

**Production Firestore rules:** the currently supplied production rules do not include the `returns` collection. Before using Returns against production, add `returns` to the active-user business collection allowlist after reviewing the security model. The app treats return reads as optional so missing permission does not block sign-in, but posting returns will fail until that rule is deployed.

## Current status

This is emulator configuration and a test plan only. The app's explicit localhost-only emulator connection switch is implemented. Returns & Exchanges is implemented on the repair branch and has corresponding emulator rules and test cases. Deterministic seed script, automated browser tests, syntax/runtime verification, and transaction test execution are still pending. No production Firebase data is changed by these files.
