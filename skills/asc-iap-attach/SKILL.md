---
name: asc-iap-attach
description: Attach in-app purchases and subscriptions to an app version for App Store review. Use when the user has IAPs or subscriptions in "Ready to Submit" state that need to be included with a first-time version submission. Works for both first-time and subsequent submissions.
---

# asc iap attach

Use this skill to attach in-app purchases and/or subscriptions to an app version for App Store review. This is the equivalent of checking the boxes in the "Add In-App Purchases or Subscriptions" modal on the version page in App Store Connect.

## When to use

- User is preparing an app version for submission and has IAPs or subscriptions to include
- User says "attach IAPs", "add subscriptions to version", "include in-app purchases for review", "select in-app purchases"
- The app version page in ASC shows an "In-App Purchases and Subscriptions" section with items to select
- IAPs/subscriptions have been created and are in "Ready to Submit" state
- The `asc subscriptions review submit` or `asc iap submit` commands fail with `FIRST_SUBSCRIPTION_MUST_BE_SUBMITTED_ON_VERSION`

## Background

Apple's official App Store Connect API (`POST /v1/subscriptionSubmissions`, `POST /v1/inAppPurchaseSubmissions`) returns `FIRST_SUBSCRIPTION_MUST_BE_SUBMITTED_ON_VERSION` for first-time IAP/subscription submissions. The `reviewSubmissionItems` API also does not support `subscription` or `inAppPurchase` relationship types.

This skill uses Apple's internal iris API (`/iris/v1/subscriptionSubmissions`) via `asc web` session authentication, which supports the `submitWithNextAppStoreVersion` attribute that the public API lacks. This is the same mechanism the ASC web UI uses when you check the checkbox in the modal.

## Preconditions

- Auth configured for CLI (`asc auth login` or `ASC_*` env vars).
- Web session authenticated (`asc web auth login --apple-id "EMAIL"`).
- Know your app ID (`ASC_APP_ID` or `--app`).
- IAPs and/or subscriptions already exist and are in **Ready to Submit** state.
- A build is uploaded and attached to the current app version.
- Required IAP/subscription metadata is complete:
  - Reference name, product ID, pricing, and at least one localization (display name).
  - Review screenshot uploaded (required for first submission of each IAP/subscription).

## Workflow

### 1. Identify items to attach

```bash
# List all in-app purchases for the app
asc iap list --app "APP_ID" --output table

# List subscription groups
asc subscriptions groups list --app "APP_ID" --output table

# List subscriptions within each group
asc subscriptions list --group-id "GROUP_ID" --output table
```

Look for items with state `READY_TO_SUBMIT`. Note their IDs.

### 2. Verify readiness

Before attaching, verify each item has the required metadata:

```bash
# Check IAP has at least one localization
asc iap localizations list --iap-id "IAP_ID" --output table

# Check subscription has at least one localization
asc subscriptions localizations list --subscription-id "SUB_ID" --output table
```

If a review screenshot is missing, upload one:

```bash
# Upload review screenshot for IAP
asc iap images create --iap-id "IAP_ID" --file "./review-screenshot.png"

# Upload review screenshot for subscription
asc subscriptions review screenshots create --subscription-id "SUB_ID" --file "./review-screenshot.png"
```

### 3. Ensure web session is active

```bash
asc web auth login --apple-id "EMAIL"
```

If already authenticated recently, the cached session will be reused automatically.

### 4. Attach subscriptions via iris API

For each subscription to attach, use `asc web review subscriptions attach`:

```bash
asc web review subscriptions attach --app "APP_ID" --subscription-id "SUB_ID" --confirm
```

To attach all subscriptions in a group at once:

```bash
asc web review subscriptions attach-group --app "APP_ID" --group-id "GROUP_ID" --confirm
```

**If the `asc web review subscriptions` commands are not available** in your version of `asc`, use the iris API directly via curl. Extract session cookies from the `asc` web session cache and call:

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Origin: https://appstoreconnect.apple.com' \
  -H 'Referer: https://appstoreconnect.apple.com/' \
  -b "$COOKIES" \
  -d '{
    "data": {
      "type": "subscriptionSubmissions",
      "attributes": {
        "submitWithNextAppStoreVersion": true
      },
      "relationships": {
        "subscription": {
          "data": {
            "type": "subscriptions",
            "id": "SUB_ID"
          }
        }
      }
    }
  }' \
  'https://appstoreconnect.apple.com/iris/v1/subscriptionSubmissions'
```

Session cookies can be extracted from the macOS Keychain:

```python
import json, subprocess
raw = subprocess.check_output([
    'security', 'find-generic-password',
    '-s', 'asc-web-session',
    '-a', 'asc:web-session:store',
    '-w'
]).decode()
store = json.loads(raw)
session = store['sessions'][store['last_key']]
parts = []
for domain, cookie_list in session['cookies'].items():
    for c in cookie_list:
        name, value = c.get('name', ''), c.get('value', '')
        if name and value:
            parts.append(f'{name}="{value}"' if name.startswith('DES') else f'{name}={value}')
print('; '.join(parts))
```

### 5. Attach IAPs via iris API

For in-app purchases (non-subscription), the same iris endpoint pattern applies:

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json' \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Origin: https://appstoreconnect.apple.com' \
  -H 'Referer: https://appstoreconnect.apple.com/' \
  -b "$COOKIES" \
  -d '{
    "data": {
      "type": "inAppPurchaseSubmissions",
      "attributes": {
        "submitWithNextAppStoreVersion": true
      },
      "relationships": {
        "inAppPurchaseV2": {
          "data": {
            "type": "inAppPurchases",
            "id": "IAP_ID"
          }
        }
      }
    }
  }' \
  'https://appstoreconnect.apple.com/iris/v1/inAppPurchaseSubmissions'
```

### 6. Verify attachments

List subscriptions and check the `submitWithNextAppStoreVersion` field:

```bash
# Via iris API
curl -s \
  -H 'Accept: application/json' \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Origin: https://appstoreconnect.apple.com' \
  -H 'Referer: https://appstoreconnect.apple.com/' \
  -b "$COOKIES" \
  'https://appstoreconnect.apple.com/iris/v1/apps/APP_ID/subscriptionGroups?include=subscriptions&limit=300&fields[subscriptions]=productId,name,state,submitWithNextAppStoreVersion'
```

Each subscription should show `"submitWithNextAppStoreVersion": true`.

Also verify via the public API:

```bash
asc subscriptions list --group-id "GROUP_ID" --output table
```

## Detaching items

To remove a subscription from the next version submission:

```bash
asc web review subscriptions remove --app "APP_ID" --subscription-id "SUB_ID" --confirm
```

Or via curl:

```bash
curl -s -X DELETE \
  -H 'Accept: application/json' \
  -H 'X-Requested-With: XMLHttpRequest' \
  -H 'Origin: https://appstoreconnect.apple.com' \
  -H 'Referer: https://appstoreconnect.apple.com/' \
  -b "$COOKIES" \
  'https://appstoreconnect.apple.com/iris/v1/subscriptionSubmissions/SUB_ID'
```

## Complete Example

Attach all ready subscriptions for app `6760430011`:

```bash
# 1. Authenticate web session
asc web auth login --apple-id "user@example.com"

# 2. List subscription groups and subscriptions
asc subscriptions groups list --app "6760430011" --output table
asc subscriptions list --group-id "GROUP_ID" --output table

# 3. Attach each subscription (via asc web or curl)
asc web review subscriptions attach --app "6760430011" --subscription-id "6760435895" --confirm
asc web review subscriptions attach --app "6760430011" --subscription-id "6760437550" --confirm

# 4. Verify
asc subscriptions list --group-id "GROUP_ID" --output table
```

## Common Errors

### "Subscription is already set to submit with next AppStoreVersion"
The subscription is already attached — this is safe to ignore. HTTP 409 with this message means the item was previously attached.

### "FIRST_SUBSCRIPTION_MUST_BE_SUBMITTED_ON_VERSION"
This error comes from the **public** API (`asc subscriptions review submit`). It means you must use the iris API approach documented in this skill instead.

### 401 Not Authorized (iris API)
The web session has expired. Re-authenticate with `asc web auth login --apple-id "EMAIL"`.

### "failed to cache session: invalid character..."
A known bug in `asc` versions ≤ 0.44.2 caused by stale legacy keychain data. Fix by clearing the corrupted keychain item:
```bash
security delete-generic-password -s "asc-web-session" -a "asc:web-session:last" 2>/dev/null
```
Then re-authenticate.

### Missing metadata
If the subscription is not in `READY_TO_SUBMIT` state, ensure:
1. At least one localization exists with a display name
2. Pricing is configured
3. A review screenshot is uploaded

## Agent Behavior

- Always list IAPs and subscriptions first to identify which are in `READY_TO_SUBMIT` state.
- If the user specifies particular items, match by reference name or product ID.
- If the user says "all", attach every item in `READY_TO_SUBMIT` state.
- Prefer `asc web review subscriptions attach` if available, fall back to curl + iris API.
- If iris API returns 409 "already set to submit", treat as success.
- After attachment, verify via the subscriptions list that `submitWithNextAppStoreVersion` is true.
- If web session is expired (401), prompt user to re-authenticate.

## CLI Approach (for subsequent submissions only)

For IAPs/subscriptions that have **already been approved** in a prior version and are being updated, the public API commands work:

```bash
# Submit updated IAP for review
asc iap submit --iap-id "IAP_ID" --confirm

# Submit updated subscription for review
asc subscriptions review submit --subscription-id "SUB_ID" --confirm
```

These commands will fail with `FIRST_SUBSCRIPTION_MUST_BE_SUBMITTED_ON_VERSION` for first-time submissions. In that case, use the iris API workflow above.

## Notes

- This skill handles the "attach to version" step only. Use `asc-submission-health` for the full submission flow.
- IAPs/subscriptions must be created first. Use CLI (`asc iap create`, `asc iap setup`, `asc subscriptions create`, `asc subscriptions setup`) to create them.
- The iris API (`/iris/v1`) mirrors the official ASC API resource types (same JSON:API format) but supports additional attributes like `submitWithNextAppStoreVersion` that the public API lacks.
- The iris API is rate-limited; keep a minimum 350ms interval between requests.
- Review screenshots are required for the first submission of each IAP/subscription, not for updates.
