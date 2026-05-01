# Lesson 8 – Logic Vulnerabilities

### Location: DVSA-ORDER-MANAGER
## Before

```javaScript
case "update":
    payload = req; 
    functionName = "DVSA-ORDER-UPDATE";
    break; 
```

## After
```javaScript
case "update":
    payload = { 
        "user": user, 
        "orderId": req["order-id"], 
        "items": req["items"] 
    };
    functionName = "DVSA-ORDER-UPDATE";
    break;
```

### Location: DVSA-ORDER-COMPLETE
## Before

```javaScript
def lambda_handler(event, context):
    orderId = event["orderId"]
    dynamodb = boto3.resource('dynamodb')
    orders_table = dynamodb.Table(os.environ["ORDERS_TABLE"])
    
    response = orders_table.query(
            KeyConditionExpression=Key('orderId').eq(orderId)
        ).get("Items", [None])
    # ... immediately processes order ...
```

## After
```javaScript
def lambda_handler(event, context):
    orderId = event["orderId"]
    dynamodb = boto3.resource('dynamodb')

    billing_table = dynamodb.Table("DVSA-BILLING") 
    payment_check = billing_table.query(
        KeyConditionExpression=Key('orderId').eq(orderId)
    ).get("Items", [])

    is_paid = any(item.get('status') == 'success' for item in payment_check)

    if not is_paid:
        return {"status": "err", "msg": "Security Alert: Payment verification failed."}

    orders_table = dynamodb.Table(os.environ["ORDERS_TABLE"])
    # ... rest of logic ...
```
