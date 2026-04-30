## Lesson 5: Broken Access Control — Fix

### Vulnerability
DVSA-ORDER-MANAGER used the unsafe node serliaze library to parse user input, allowing attackers to embed executable JavaScript code in API requests.

### Fix Applied
Replaced  noder serliaze  with safe JSON.Parse() in DVSA-ORDER-MANAGER.

### Code Change

**Before:**
```javascript
var req = serialize.unserialize(event.body);
var headers = serialize.unserialize(event.headers);
```

**After (fixed):**
```javascript
var req = JSON.parse(event.body);
var headers = event.headers;
```

### Verification
**Before fix — exploit worked:**
```bash
curl -s -X POST "$API" \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d '{"action": "_$$ND_FUNC$$_function(){...}()", "cart-id": ""}' 
# Result: order status changed to paid with total $0
```

**After fix — exploit blocked:**
```bash
curl -s -X POST "$API" \
  -H "Content-Type: application/json" \
  -H "Authorization: $TOKEN" \
  -d '{"action": "_$$ND_FUNC$$_function(){...}()", "cart-id": ""}'
# Result: {"status":"err","msg":"unknown action"} — injection blocked
```

