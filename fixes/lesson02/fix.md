# Lesson 2 – Broken Authentication: Fix

**File modified:** `order-manager.js`
**Lambda function:** `DVSA-ORDER-MANAGER`

The vulnerable handler only base64-decoded the JWT payload and trusted its claims, never verifying the signature, issuer, or expiration. The fix replaces this with full cryptographic verification against the Cognito User Pool's JWKS.

---

## Change 1 — Add JWT Verification Helpers

**Where:** Inserted immediately after `const jose = require('node-jose');`

```javascript
const https = require('https');
let _jwksCache = { keystore: null, fetchedAt: 0 };

function resp(statusCode, bodyObj) {
  return {
    statusCode,
    headers: { "Access-Control-Allow-Origin": "*" },
    body: JSON.stringify(bodyObj)
  };
}

function fetchJson(url) {
  return new Promise((resolve, reject) => {
    https.get(url, (res) => {
      let data = "";
      res.on("data", (c) => data += c);
      res.on("end", () => {
        if (res.statusCode >= 200 && res.statusCode < 300) {
          try { resolve(JSON.parse(data)); } catch (e) { reject(e); }
        } else { reject(new Error(`HTTP ${res.statusCode}`)); }
      });
    }).on("error", reject);
  });
}

async function getCognitoKeystore() {
  const now = Date.now();
  if (_jwksCache.keystore && (now - _jwksCache.fetchedAt) < 6 * 60 * 60 * 1000)
    return _jwksCache.keystore;
  const region = process.env.AWS_REGION;
  const userPoolId = process.env.userpoolid;
  const jwksUrl = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}/.well-known/jwks.json`;
  const jwks = await fetchJson(jwksUrl);
  const keystore = await jose.JWK.asKeyStore(jwks);
  _jwksCache = { keystore, fetchedAt: now };
  return keystore;
}

async function verifyCognitoJwt(jwt) {
  const region = process.env.AWS_REGION;
  const userPoolId = process.env.userpoolid;
  const issuer = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}`;
  const keystore = await getCognitoKeystore();
  const result = await jose.JWS.createVerify(keystore).verify(jwt);
  const claims = JSON.parse(result.payload.toString("utf8"));
  if (claims.iss !== issuer) throw new Error("bad issuer");
  if (typeof claims.exp === "number" && (Date.now() / 1000) > claims.exp) throw new Error("expired");
  if (claims.token_use && !["access", "id"].includes(claims.token_use)) throw new Error("bad token_use");
  return claims;
}
```

---

## Change 2 — Replace the Vulnerable Parsing Block

**Where:** Inside `exports.handler`, replacing the JWT parsing block.

### Before (vulnerable)

```javascript
var auth_header = headers.Authorization || headers.authorization;
var token_sections = auth_header.split('.');
var auth_data = jose.util.base64url.decode(token_sections[1]);
var token = JSON.parse(auth_data);
var user = token.username;
var isAdmin = false;
```

> No signature check. The payload is trusted as-is.

### After (secure)

```javascript
var auth_header = (headers.Authorization || headers.authorization || "");
var jwt = auth_header.replace(/^Bearer\s+/i, "").trim();
if (!jwt) {
  return callback(null, resp(401, { status: "err", msg: "missing authorization" }));
}
verifyCognitoJwt(jwt).then((claims) => {
  var user = claims.username || claims["cognito:username"] || claims.sub;
  if (!user) {
    return callback(null, resp(401, { status: "err", msg: "missing subject" }));
  }
  var isAdmin = false;
```

> Signature, issuer, and expiration are cryptographically verified.

---

## Change 3 — Close the Promise Chain

**Where:** Appended before the final closing brace of `exports.handler`.

```javascript
}).catch((e) => {
  console.log("JWT verify failed:", e);
  return callback(null, resp(401, { status: "err", msg: "invalid token" }));
});
```

>  Any verification failure returns a clean 401.

---

## Specific Changes Summary

- **Added:** `https` import and `_jwksCache` for caching the Cognito JWKS keystore.
- **Added:** Helper `resp()` for building consistent HTTP responses.
- **Added:** Helper `fetchJson()` to retrieve JSON over HTTPS.
- **Added:** Helper `getCognitoKeystore()` that fetches and caches the JWKS for 6 hours.
- **Added:** Helper `verifyCognitoJwt()` that cryptographically verifies the signature and validates `iss`, `exp`, and `token_use` claims.
- **Changed:** Replaced unverified base64 decoding of the JWT payload with a full call to `verifyCognitoJwt()`.
- **Added:** Strip optional `Bearer` prefix and trim the token.
- **Added:** 401 response when the `Authorization` header is missing.
- **Added:** 401 response when the verified token has no usable subject claim.
- **Added:** `.catch()` block on the verification promise to return a uniform 401 on any verification failure.

After editing, the function was saved and activated using the **Deploy** button in the Lambda console.
