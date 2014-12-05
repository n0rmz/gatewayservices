#Gateway Services Protocol Walk-through

##Scenario:

- A customer of USD-Bank who only has US Dollars wants to send Euros to customer of EUR-Bank.
- Both banks are using Gateway Services Protocol to negotiate and setup an invoice, when the invoice is paid, a payment will settle through Ripple and the sender balance will be debited and the receiver's balance will be credited.
- The sender knows the account identifier for the receiver at EUR-Bank and the amount of Euros to be sent.

Three necessary information need to start the flow:

- Sender	acct:alice@usd-bank.com
- Receiver	acct:bob@eur-bank.eu
- Amount 	100.00 EUR

##1. Host-meta call to Destination  (EUR-Bank)

Based upon the receiver's account URI a Host-meta is done to the receiving domain(eur-bank.eu) to determine what the webfinger URL is. Services and other information describing the domain other than Ripple related information may be provided by Host-meta as well.

>GET https://eur-bank.eu/.well-known/host-meta.json

Response

```javascript
{
    "subject": "https://eur-bank.eu",
    "expires": "2014-01-30T09:30:00Z",
    "properties": {
        "name": "eur-bank.eu",
        "description": "A EURO bank.",
        "rl:type": "gateway",
        "rl:domain": "https://eur-bank.eu",
        "rl:accounts": [
            {
                "address": "r4tFZoa7Dk5nbEEaCeKQcY3rS5jGzkbn8a",
                "currencies": ["EUR”, "GBP"]
            },
        ],
        "rl:hotwallets": [
            "rEKuBLEX2nHUiGB9dCGPnFkA7xMyafHTjP"
        ]
    },
    "links": [
        {
            "rel": "lrdd",
            "template": "https://eur-bank.eu/.well-known/webfinger.json?q={uri}",
            "properties": {
                "rl:uri_schemes": ["acct",”ripple”]
            }
        },
        {
            "rel": "https://gatewayd.org/gateway-services/bridge_payments",
            "href": "https://api.eur-bank.eu/v1/bridge_payments",
            "properties": {
                "version": "1",
                "additional_info_definitions":{}
						}
        }
    ]
}
```

Host-Meta `"properties": {}` may contain any of the following Ripple defined fields:
- `rl:type`
- `rl:domain`
- `rl:accounts`
- `rl:hotwallets`

As a Gateway increases their functionality with Ripple, they may add more links/services. The expiration of the Host-Meta can be used to validate the last update of it.
The Gateway Services Protocol specification currently supports webfinger and bridge_payments
properties and other links are arbitrary (not all use cases are Ripple related)


##2. Webfinger Destination User based upon Host-meta URL

Now that the webfinger URL has been retreived from host-meta, the destination account details can be retrieved with the following:

>GET https://eur-bank.eu/.well-known/webfinger.json?resource=acct:bob@eur-bank.eu

Response

```javascript
{
    "subject": "acct:bob@eur-bank.eu",
    "expires": "2014-10-07T22:46:35.097Z",
    "links": [
        {
            "rel": "https://gatewayd.org/gateway-services/bridge_payments",
            "href": "https://api.eur-bank.eu/v1/bridge_payments",
            "properties": {
                "version": "1",
                "send_to":
                [
                    {
                        "uri": "acct:bob@eur-bank.eu",
                        "currencies": ["EUR", "BTC"]
                    },
                    {
                       "uri": "acct:bob+savings@eur-bank.eu",
                       "currencies": ["EUR"]
                       }
                ]
            }
        }
    ]
}
```

If you WebFinger any of the aliases or accounts at the same domain, the resulting document should be the same.
`"links": []`: should contain link elements describing where to find bridge payments API. These should match the host-meta URLs for the same APIs.


##3. Get a bridge quote for the payment

With both the proposed information and the bridge_payment link, a call will be executed to get a quote from the receiving gateway. The quote itself is meant to provide a template reponse that will describe what additional information is needed for the proposed payment and what additional instructions or details need to be included in order for the receiver to receive the funds. In some cases the receiving gateway may already know the sender from previous transactions and does not require additional information.

The role of a bridge quote (once the state of a quote is "invoiced") is to make sure the receiving gateway and the sending gateway have agreed to the instructions and details needed to ensure the payment to the receiver will be made. The receiving gateway essentially prepares an "invoice" that can be used to map against a wallet-payment(Ripple Payment) in this case with a corresponding invoice_id.

>GET https://api.eur-bank.eu/v1/bridge_payments/quotes/acct%3Aalice%40usd-bank.com/acct%3Abob%40eur-bank.eu/100+EUR`

Response

```javascript
{
    "success": true,
    "bridge_payments": [
        {
            "state": "quote",
            "created": "2014-09-23T19:20:20.000Z",
            "source": {
                "uri": "acct:alice@usd-bank.com",
                "claims_required": ["dob","us_ssn","zip"],
                "claims_jwts": [],
                "additional_info": {}
            },
            "wallet_payment": {
                "destination": "ripple:rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W?dt=29445",
                "primary_amount": {
                    "amount": "100.31",
                    "currency": "EUR",
                    "issuer": "rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W"
                },
                "invoice_id": "789f8ede3d6860283d53621e4cd26a9afa3b72f7b5634",
                "expiration": "2014-09-23T20:20:20.000Z"
            },
            "destination": {
                "uri": "acct:bob@eur-bank.eu",
                "claims_required": ["dob","us_ssn","zip"],
                "claims_jwts": ["eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmYW1pbHlfbmFtZSI6IlJlZ2luZWxsaSIsImdpdmVuX25hbWUiOiJSb21lIn0.o0l4dfizDHOlbqJonkwyqBhufIwfXX9Ltjv56Fcn9SQ", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXNzYWdlIjoiQ29uZ3JhdHVsYXRpb25zLCB5b3UgZGVjb2RlZCBhIEpXVCJ9.msBhzh2yzZabnMei9b1mchspWXWS0xOxCf4Vlhd6mAg"],
                "additional_info": {}
            },
            "destination_amount": {
                "amount": "100",
                "currency": "EUR",
                "issuer": "rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W"
            },
            "parties": {
                "inbound_bridge": "",
                "outbound_bridge": "http://eur-bank.eu",
                "sending_agent": "",
                "receiving_agent": ""
            }
        }
    ]
}
```

Key points:
- The `source` claims for the sender are blank and the required information needs to be provided when the quote is accepted.
- The `wallet_payment` is referring to a Ripple payment in this case and the Ripple details are provided including an `invoice_id` that will be used to identify the payment. If the amount does not match the quoted amount than the payment can be denied. Notice that the amount quoted for the wallet_payment is more than what the sender wants to send the receiver. This is because of the fees that the receiving gateway collets in order to make this payment.
- The `expiration` is also something to take note about as the receiving gateway may only want the quote to last for so long before it is subject to risk of outdated information. The expiration of a confirmed bridge quote can also be updated and changed after it has been accepted.
- The `destination` information regarding the receiver has been provided as the reciever is a valid end destination/customer that the receiving gateway knows and trusts.
- `destination_amount` shows the amount that will be delivered to the receiver in the end if the payment is successful

##4. Accepting the Quote

In the most straight forward cases proposing a bridge_payment will return 1 quote, however as more and more systems connect there may be opportunities to return multiple quotes. It is up to the sending party to decide which quote to accept and once accepted this quote needs to be POSTed back with any additional information needed to change the state into an "invoiced" quote .

>POST https://api.eur-bank.eu/v1/bridge_payments

```javascript
{
            "state": "quote",
            "created": "2014-09-23T19:20:20.000Z",
            "source": {
                "uri": "acct:alice@usd-bank.com",
                "claims_required": ["dob","us_ssn","zip"],
                "claims_jwts": ["eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmYW1pbHlfbmFtZSI6IlJlZ2luZWxsaSIsImdpdmVuX25hbWUiOiJSb21lIn0.o0l4dfizDHOlbqJonkwyqBhufIwfXX9Ltjv56Fcn9SQ", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXNzYWdlIjoiQ29uZ3JhdHVsYXRpb25zLCB5b3UgZGVjb2RlZCBhIEpXVCJ9.msBhzh2yzZabnMei9b1mchspWXWS0xOxCf4Vlhd6mAg"],
                "additional_info": {}
            },
            "wallet_payment": {
                "destination": "ripple:rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W?dt=29445",
                "primary_amount": {
                    "amount": "100.31",
                    "currency": "EUR",
                    "issuer": "rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W"
                },
                "invoice_id": "789f8ede3d6860283d53621e4cd26a9afa3b72f7b5634",
                "expiration": "2014-09-23T20:20:20.000Z"
            },
            "destination": {
                "uri": "acct:bob@eur-bank.eu",
                "claims_required": ["dob","us_ssn","zip"],
                "claims_jwts": ["eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmYW1pbHlfbmFtZSI6IlJlZ2luZWxsaSIsImdpdmVuX25hbWUiOiJSb21lIn0.o0l4dfizDHOlbqJonkwyqBhufIwfXX9Ltjv56Fcn9SQ", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXNzYWdlIjoiQ29uZ3JhdHVsYXRpb25zLCB5b3UgZGVjb2RlZCBhIEpXVCJ9.msBhzh2yzZabnMei9b1mchspWXWS0xOxCf4Vlhd6mAg"],
                "additional_info": {}
            },
            "destination_amount": {
                "amount": "100",
                "currency": "EUR",
                "issuer": "rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W"
            },
            "parties": {
                "inbound_bridge": "http://usd-bank.com",
                "outbound_bridge": "http://eur-bank.eu",
                "sending_agent": "",
                "receiving_agent": ""
            }
}
```
Key points:
- The quote returned is exactly the same as what was proposed other than the additional claims for the sender to provide KYC information for the receiving gateway.
- By sending all the payment details back, the sending gateway is confirming that it will deliver on these details before the receiving gateway will send the payment.

**Successful Response from EUR-Bank**

A response containing the same object back, with the following changes:
An id added
The state updated to “invoice”
Optionally, the expiration of the wallet payment extended to a later time.

```javascript
{
    “success”: true,
    “bridge_payment”: {
		"id": "9876A034899023AEDFE",
            "state": "invoice",
            "created": "2014-09-23T19:20:20.000Z",
            "source": {
                "uri": "acct:alice@usd-bank.com",
                "claims_required": ["dob","us_ssn","zip"],
                "claims_jwts": ["eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmYW1pbHlfbmFtZSI6IlJlZ2luZWxsaSIsImdpdmVuX25hbWUiOiJSb21lIn0.o0l4dfizDHOlbqJonkwyqBhufIwfXX9Ltjv56Fcn9SQ", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXNzYWdlIjoiQ29uZ3JhdHVsYXRpb25zLCB5b3UgZGVjb2RlZCBhIEpXVCJ9.msBhzh2yzZabnMei9b1mchspWXWS0xOxCf4Vlhd6mAg"],
                "additional_info": {}
            },
            "wallet_payment": {
                "destination": "ripple:rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W?dt=29445",
                "primary_amount": {
                    "amount": "100.31",
                    "currency": "EUR",
                    "issuer": "rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W"
                },
                "invoice_id": "789f8ede3d6860283d53621e4cd26a9afa3b72f7b5634",
                "expiration": "2014-09-23T20:20:20.000Z"
            },
            "destination": {
                "uri": "acct:bob@eur-bank.eu",
                "claims_required": ["dob","us_ssn","zip"],
                "claims_jwts": ["eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJmYW1pbHlfbmFtZSI6IlJlZ2luZWxsaSIsImdpdmVuX25hbWUiOiJSb21lIn0.o0l4dfizDHOlbqJonkwyqBhufIwfXX9Ltjv56Fcn9SQ", "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJtZXNzYWdlIjoiQ29uZ3JhdHVsYXRpb25zLCB5b3UgZGVjb2RlZCBhIEpXVCJ9.msBhzh2yzZabnMei9b1mchspWXWS0xOxCf4Vlhd6mAg"],
                "additional_info": {}
            },
            "destination_amount": {
                "amount": "100",
                "currency": "EUR",
                "issuer": "rUU8vXA3p6d92skbArWjaKWqfbtxyrr65W"
            },
            "parties": {
                "inbound_bridge": "http://usd-bank.com",
                "outbound_bridge": "http://eur-bank.eu",
                "sending_agent": "",
                "receiving_agent": ""
            }
			}
}
```

##5. Sending the payment

Now that a quote has been setup as an invoice, the receiving gateway will await the Ripple payment of 100.31 EUR. Once that has been received a corresponding payment will be made out to the destination account.
