# Lesson 10: Unhandled Exceptions

## Before

```js
const { LambdaClient, InvokeCommand } = require("@aws-sdk/client-lambda");
const { CognitoIdentityProviderClient, AdminGetUserCommand } = require("@aws-sdk/client-cognito-identity-provider");
const jose = require('node-jose');

exports.handler = (event, context, callback) => {
    // console.log(JSON.stringify(event));
    var req = JSON.parse(event.body);
    var headers = event.headers;
}
```

## After

```js
const { LambdaClient, InvokeCommand } = require("@aws-sdk/client-lambda");
const { CognitoIdentityProviderClient, AdminGetUserCommand } = require("@aws-sdk/client-cognito-identity-provider");
const jose = require('node-jose');

exports.handler = (event, context, callback) => {
    // console.log(JSON.stringify(event));
    let req;

    try {
        req = JSON.parse(event.body);
    } catch (e) {
        console.log("Invalid JSON request:", e.message);
        return callback(null, {
            statusCode: 400,
            headers: { "Access-Control-Allow-Origin": "*" },
            body: JSON.stringify({
                status: "err",
                msg: "Invalid request format"
            })
        });
    }

    var headers = event.headers;
}
```

## Specific changes made:

- Added a `try-catch` block around `JSON.parse(event.body)`

- Changed unhandled exceptions into a controlled error response

- Returned a safe message: `Invalid request format`
