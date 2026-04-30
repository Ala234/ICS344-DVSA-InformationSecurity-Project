## Lesson 4: Insecure Cloud Configuration — Fix

### Vulnerability
S3 feedback bucket allowed public write and read access due to misconfigured bucket policy and disabled Block Public Access.

### Fix Applied
Enabled **Block all public access** on the S3 bucket.


### Verification

**Before fix:**
```bash
# Upload succeeded with no authentication
curl -X PUT "https://dvsa-feedback-bucket-878598436754-us-east-1.s3.amazonaws.com/malicious.txt" \
  --upload-file malicious.txt
```

**After fix:**
```bash
# Upload rejected
curl -X PUT "https://dvsa-feedback-bucket-878598436754-us-east-1.s3.amazonaws.com/malicious.txt" \
  --upload-file malicious.txt
# Output: Access Denied
```
