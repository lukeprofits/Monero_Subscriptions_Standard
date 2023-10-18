# Monero Payment Request Standard
> Formerly the [Monero Subscription Code Standard](https://github.com/lukeprofits/Monero_Subscription_Code_Standard)

# Version 1:
## Decoding & Encoding `monero-request:` Payment Codes

This document explains how to decode and encode `monero-request:` payment codes using gzip compression and Base64 encoding.

## Decoding Monero Payment Requests
To decode a Monero Payment Request, follow these steps:

1. Remove the Monero Payment Request identifier: `monero-request:`
2. Remove the version identifier `1:`
3. Decode the string from Base64 to obtain the compressed data.
4. Decompress the compressed data using gzip to get the JSON string.
5. Parse the JSON string to extract the field values.

## Example Function To Decode Monero Subscription Code

```
import base64
import gzip
import json


def decode_monero_subscription_code(monero_subscription_code):
    # Catches user error. Code can start with "monero-request:", or ""
    code_parts = monero_subscription_code.split('-subscription:')
    if len(code_parts) == 2:
        monero_subscription_data = code_parts[1]
    else:
        monero_subscription_data = code_parts[0]
        
    # Extract the Base64-encoded string from the second part of the code
    encoded_str = monero_subscription_data
    
    # Decode the Base64-encoded string into bytes
    compressed_data = base64.b64decode(encoded_str.encode('ascii'))
    
    # Decompress the bytes using gzip decompression
    json_bytes = gzip.decompress(compressed_data)
    
    # Convert the decompressed bytes into a JSON string
    json_str = json_bytes.decode('utf-8')
    
    # Parse the JSON string into a Python object
    subscription_data_as_json = json.loads(json_str)
    
    return subscription_data_as_json


monero_subscription_code = 'monero-request:1:H4sIAGOfSWQC/x2O206DQBBAf4XwbBvKzeAbtKCxqYnQCvpCdpexoAtL9lK7a/x32b7NnDnJmV+XKCHZ2FKEgboPjtscSqdS2KkRpSCdHVzW7p3jClhWLtqfG7ZimMqgifjlTc9H9nkeFbwkInmV3HQlRJmCgovv9GPY3GfsHfdGC2YMOxRZbOrpuO8et3F6zVOc5xExRRn0y/SMxRj2W2j8ykaJ4hwmom3uVO0sQiNTk+1vknWSLGBGeoRJtkNnra+m8b35SKo623dPp+vtdYm4bDskwRq+5wcrL1z5sb3hgdJhOrdEEwqLo8XiBN7fP3qScGsYAQAA'

decoded_data = decode_monero_subscription_code(monero_subscription_code)

print(decoded_data)
```

## Encoding Monero Subscription Payment Codes
To encode a Monero subscription payment code, follow these steps:

1. Convert the payment details to a JSON object.
2. Compress the JSON object using gzip compression.
3. Encode the compressed data in Base64 format.
4. Add the Monero Subscription identifier `monero-request:` to the encoded string.


## Example Function To Create A Monero Subscription Code

```
import base64
import gzip
import json


def make_monero_subscription_code(json_data):
    # Convert the JSON data to a string
    json_str = json.dumps(json_data)
    
    # Compress the string using gzip compression
    compressed_data = gzip.compress(json_str.encode('utf-8'))
    
    # Encode the compressed data into a Base64-encoded string
    encoded_str = base64.b64encode(compressed_data).decode('ascii')
    
    # Add the Monero Subscription identifier
    monero_subscription = 'monero-request:' + encoded_str
    
    return monero_subscription
    

json_data = {
     "custom_label": "My Subscription",  # This can be any text
     "sellers_wallet": "4At3X5rvVypTofgmueN9s9QtrzdRe5BueFrskAZi17BoYbhzysozzoMFB6zWnTKdGC6AxEAbEE5czFR3hbEEJbsm4hCeX2S",
     "currency": "USD",                  # Currently supports "USD" or "XMR"
     "amount": 19.99,                    
     "payment_id": "9fc88080d1d5dc09",   # Unique identifier so the merchant knows which customer the payment relates to
     "start_date": "2023-04-26",         # If you want it to start the day of, you can use: datetime.now().strftime("%Y-%m-%d")
     "billing_cycle_days": 30            # How often it should be billed
     }
     
monero_subscription_code = make_monero_subscription_code(json_data)

print(monero_subscription_code)

```

### Monero Payment Request Protocol: Using the Optional `change_indicator_url`

The `change_indicator_url` is an optional field designed for merchants who wish to have the flexibility to request modifications to an existing payment request. This could be a one-time payment, a payment with a set number, or a recurring subscription. **It's important to note that the merchant cannot enforce these changes unilaterally.** When a change is requested, all related automatic payments are paused until the customer reviews and either confirms or rejects the changes.

#### Key Features and Constraints

- **Merchant Requests, Not Commands**: The main utility of this feature is for merchants to request changes, such as updating a wallet address or changing the price. The merchant cannot force these changes on the customer.
  
- **Automatic Pause on Changes**: The wallet will query the `change_indicator_url` before any scheduled payment. If it detects a change, automatic payments are paused and the customer is notified.

- **Customer Consent Required**: Payments remain paused until the customer actively confirms or rejects the proposed changes. If confirmed, payments resume; if rejected, the payment schedule is canceled.

#### How it Works

1. **URL Formation**: The `change_indicator_url` is constructed by appending the unique `payment_id` to a base URL specified by the merchant.
    - Example: `www.mysite.com/api/monero-request?payment_id=9fc88080d1d5dc09`

2. **Merchant Changes**: To request a change, the merchant updates the information at the `change_indicator_url`. These changes remain in the status of "requested" until approved or declined by the customer.

3. **Customer Notification and Confirmation**: Upon detecting a change, the wallet notifies the customer who then must make a decision to accept or decline. Payments stay paused until this decision is made.

#### Merchant Guidelines

- **Endpoint Setup**: Merchants should create a REST endpoint capable of handling GET requests for the `change_indicator_url`.
  
- **Initiating Changes**: To request changes, the merchant updates the content at the REST endpoint according to the Monero Payment Request Protocol format.

- **Cancellation Request**: If a merchant wishes to cancel the payment request entirely, they can specify this at the `change_indicator_url` (e.g., `"status": "cancelled"`).

- **Supplemental Customer Notification**: Though the `change_indicator_url` should trigger automatic notifications, merchants are encouraged to also notify customers through other channels as a best practice.

#### JSON Structure for Merchant Changes

Merchants can send JSON data with the following fields to initiate different types of changes:

- **To Cancel Subscription**
    ```json
    {
        "action": "cancel",
        "note": "We are going out of business."
    }
    ```

- **To Update Payment Fields**
    ```json
    {
        "action": "update",
        "fields": {
            "amount": 25.99,
            "currency": "XMR"
        },
        "note": "Price has changed due to increased costs."
    }
    ```

#### Bulk Updates

Merchants can ignore the `payment_id` query parameter to initiate blanket updates for all payment_ids associated with the `change_indicator_url`.

This optional `change_indicator_url` feature enhances the protocol's flexibility, enabling merchants to request changes while ensuring customers maintain full control over their payment options.


# Tools For Creating `monero-request` codes:
* Recommended: [Monero Subscription Code Creator Website](https://monerosub.tux.pizza/)
* [Monero Subscription Code Creator CLI Tool](https://github.com/lukeprofits/Monero_Subscription_Code_Creator)
* [MoneroSub Subscription Code Creator Pip Package](https://github.com/lukeprofits/monerosub)
* More monero-request integration tools coming soon...
