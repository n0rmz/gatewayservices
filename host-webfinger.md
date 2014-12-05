#Replacing Ripple.txt and Federation Protocol with Host-meta and Webfinger

Both Ripple.txt and the Federation Protocol play important roles for Ripple to help federate and provide a standard upon which Ripple entities and universal support across different user systems can be achieved.

In order to provide a more widely adopted and robust standard for this, we believe that everything that Ripple.txt and Federation protocol provided for functionality can be duplicated directly with Host-meta and should be relatively easy to transition over.

The concept of Host-meta and webfinger work exactly the same as Ripple.txt and the federation protocol. Let's say we have a destination of bob@omegabank.com, in the current implementation the following would occur:

1. Retrieve ripple.txt from https://omegabank.com/ripple.txt
2. Find `[federation_url]` in ripple.txt
3. Use the federation_url to query against "bob@omegabank.com"
4. A single destination tag or ripple address would be provided

With Host-meta and webgfinger the flow is exactly the same for bob@omegabank.com

1. Retrieve the host-meta from https://omegabank.com/well-known/host-meta
2. Find the `lrdd` link that points to the webfinger URL
3. Use the webfinger URL to query against "bob@omegabank.com"
4. Information either for the destination tag or additional links to use will be provided

##Here are examples of host meta and webfinger calls.

>GET https://omegabank.com/.well-known/host-meta.json

Response

```javascript
{
    "subject": "https://omegabank.com",
    "expires": "2014-01-30T09:30:00Z",
    "properties": {
        "name": "Omega Bank",
        "description": "The last bank in the world.",
        "rl:type": "gateway",
        "rl:domain": "https://omegabank.com",
        "rl:accounts": [
            {
                "address": "r4tFZoa7Dk5nbEEaCeKQcY3rS5jGzkbn8a",
                "currencies": ["EUR”, "GBP", "USD"]
            },
        ],
        "rl:hotwallets": [
            "rEKuBLEX2nHUiGB9dCGPnFkA7xMyafHTjP"
        ]
    },
    "links": [
        {
            "rel": "lrdd",
            "template": "https://omegabank.com/.well-known/webfinger.json?q={uri}",
            "properties": {
                "rl:uri_schemes": ["acct",”ripple”]
            }
        }
    ]
}
```

Now that we know the webfinger URL we can make a call against that to get the information we need.

>GET https://eur-bank.eu/.well-known/webfinger.json?resource=acct:bob@omegabank.com

Response

```javascript
{
    "subject": "acct:bob@omegabank.com",
    "expires": "2014-10-07T22:46:35.097Z",
    "properties": {
        "username": "bob@omegabank.com",
        "rl:accounts": [
            {
                "address": "r4tFZoa7Dk5nbEEaCeKQcY3rS5jGzkbn8a",
                "destination_tag": "12345"
            },
        ]
    },
    "links": []
}
```
