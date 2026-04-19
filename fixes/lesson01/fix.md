# Lesson 1: Event Injection

## Before

```javaScript
const serialize = require('node-serialize'); 
const { LambdaClient, InvokeCommand } = require("@aws-sdk/client-lambda"); 
const { CognitoIdentityProviderClient, AdminGetUserCommand } = require("@aws-sdk/client-cognito-identity-provider"); 
const jose = require('node-jose'); 

exports.handler = (event, context, callback) => { 
 // console.log(JSON.stringify(event)); 
    var req = serialize.unserialize(event.body);  
    var headers = serialize.unserialize(event.headers); 
```

## After
```javaScript
const { LambdaClient, InvokeCommand } = require("@aws-sdk/client-lambda"); 
const { CognitoIdentityProviderClient, AdminGetUserCommand } = require("@aws-sdk/client-cognito-identity-provider"); 
const jose = require('node-jose'); 
exports.handler = (event, context, callback) => { 
    // console.log(JSON.stringify(event)); 
 var req = JSON.parse(event.body);  
 var headers = event.headers;  
 ;
```

## Specific changes made: 

- Removed: const serialize = require('node-serialize'); 

- Changed: serialize.unserialize(event.body) → JSON.parse(event.body) 

- Changed: serialize.unserialize(event.headers) → event.headers 
