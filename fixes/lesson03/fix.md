# Lesson 3 – Sensitive Data Exposure: Fix

**File modified:** `admin_get_receipts.py`
**Lambda function:** `DVSA-ADMIN-GET-RECEIPT`

The vulnerable handler performed privileged work — packaging all order receipts from S3 and returning a presigned download URL — without verifying the caller's identity or admin status. Any actor able to invoke the function could obtain receipts belonging to other users. The fix requires a `caller_username` in the event payload and confirms `custom:is_admin = true` in Cognito before any S3 reads occur.

---

## Change 1 — Add `is_admin_user` Helper

**Where:** Inserted at the top of the file, immediately after the imports.

```python
def is_admin_user(username):
    """Check if the given username is an admin in Cognito."""
    cognito = boto3.client('cognito-idp')
    try:
        user = cognito.admin_get_user(
            UserPoolId=os.environ['userpoolid'],
            Username=username
        )
        for attr in user.get('UserAttributes', []):
            if attr['Name'] == 'custom:is_admin' and attr['Value'] == 'true':
                return True
        return False
    except Exception as e:
        print("Admin check failed:", e)
        return False
```

---

## Change 2 — Add Authorization Block at the Top of `lambda_handler`

**Where:** Inside `lambda_handler`, inserted as the first lines of the function — before any S3 client creation or downloads.

### Before (vulnerable)

```python
def lambda_handler(event, context):
    client = boto3.client('s3')
    resource = boto3.resource('s3')
    y = event["year"]
    # ... downloads receipts immediately ...
```

>  No identity check, no admin check. Any caller able to invoke the function gets every user's receipts.

### After (secure)

```python
def lambda_handler(event, context):
    caller = event.get("caller_username")
    if not caller:
        return {
            "status": "err",
            "message": "Missing caller_username"
        }
    if not is_admin_user(caller):
        return {
            "status": "err",
            "message": "Unauthorized: admin only"
        }
    client = boto3.client('s3')
    resource = boto3.resource('s3')
    y = event["year"]
    # ... downloads receipts immediately ...
```

>  Caller is required and verified as admin via Cognito before any privileged work runs.

---

## Specific Changes Summary

- **Added:** Helper function `is_admin_user(username)` that queries Cognito (`admin_get_user`) for the `custom:is_admin` attribute and returns `True` only when its value equals `'true'`.
- **Added:** Required `caller_username` field at the top of `lambda_handler`.
- **Added:** Rejection with `Missing caller_username` when the field is absent.
- **Added:** Rejection with `Unauthorized: admin only` when the caller is not an admin.
- **Removed:** Implicit acceptance of any caller able to invoke the function — privileged S3 reads no longer run before authorization succeeds.

After editing, the function was saved and the **Deploy** button was pressed in the Lambda console, producing a *Successfully deployed* confirmation.
