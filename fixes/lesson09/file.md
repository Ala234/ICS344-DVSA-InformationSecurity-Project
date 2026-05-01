# Lesson 10: Unhandled Exceptions

## Before

```js
exports.handler = (event, context, callback) => {
    var req = JSON.parse(event.body);
    var headers = event.headers;
}
...
## After

```js
exports.handler = (event, context, callback) => {
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
...

## Specific changes made:
Added a try-catch block around JSON.parse(event.body)
Changed unhandled exception into a controlled error response
Returned a safe message: Invalid request format
