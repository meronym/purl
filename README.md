# pURL

A workflow for embedding payments into HTTP calls


### Primary use cases

- Full node incentivisation (allows clients to pay Ethereum network nodes for json-rpc requests)

- Generic API monetisation


### Example

Main actors: the Client (**C**), the Server (**S**), the Hub (**H**).

1. **C** and **S** each open a payment channel with **H**.

2. **C** wants to issue a HTTP request to **S**. 

    - **C** generates a "cheque" that allows **S** to spend *at most $X* for processing the request.

    - **C** attaches the cheque as a `Payment` header to the request and sends it to **S**.

3. **S** validates the `Payment` header and processes the request.

    - **S** deducts the price for the request and generates an updated cheque that refunds the difference to **C**.

    - **S** attaches the cheque as a `Payment` header to the response and sends it to **C**.

 
### Design goals

- Open source stack (client, server, hub)

- Open payment protocol

- Carrier-agnostic (not limited to a single asset type or payment network architecture).
  
  - v0 Prototype will focus on DAI single-hub payment channels
  
- Native support for custody delegation


## Main components

### 1. The Client

Allows users to attach payments to arbitrary HTTP requests.

Implemented as a local HTTP proxy that that injects payment data into incoming requests and handles the payment exchange with the server.

- Generates a deterministic hash for each request (url + headers + body)

- Attaches a cheque that allows the server to deduct at most $X for successfully processing the request

- Parses the server response and validates the cheque update


### 2. The Server

Allows service providers to charge money in exchange of processing arbitrary HTTP requests.

Implemented as a local HTTP proxy (+ an nginx module in a future iteration) that:

- Receives an incoming request, decodes and validates the payment header

- Forwards the request to an upstream service

- Reads the response from the upstream service (including how much should the client be charged)

- Sends back an updated cheque to the client, along with the response body


### 3. The Hub

A third party service that facillitates several parts of the client & server journey:  

- Onboarding

- State channel funding / management / backup

- Custodial operations

In a future iteration, users should be able to self-host their hubs and connect to other hubs that support the same transport mechanisms.


## UX Sketch


### Client

```
# Install and start the payment proxy

$ npm install -g purl
$ purl
Opening a channel with hub.purl.io ...
You have $10 free credit.
Listening on localhost:8042

[2020-02-28 10:49:12] Forwarding #24a69c42... (max $0.001) to demo.purl.io
[2020-02-28 10:49:12] Paid $0.00042 for #24a69c42... to demo.purl.io

# Send a paid request

$ curl --proxy localhost:8042 demo.purl.io
{
    "message": "it works"
}
```


### Server

```
# Install and run the payment server proxy
$ npm install -g purl
$ purl-server
Opening a channel with hub.purl.io ...
You're all set! Listening for incoming requests.

[2020-02-28 10:49:12] Serving #24a69c42... (max $0.001) via localhost:8888
[2020-02-28 10:49:12] Charged $0.00042 for #24a69c42

# Withdraw funds
$ purl-server withdraw 0x1234
Sending $0.0042 to 0x1234 ... done!
```


### Hub

```
Install and run the payment hub server
$ npm install -g purl
$ purl-hub

[2020-02-28 10:49:12] Opening a $10 channel with client 0xcccc ... done.
[2020-02-28 10:49:12] Opening a $10 channel with server 0x5555 ... done.
[2020-02-28 10:49:13] Approving a $1 virtual channel between 0xcccc and 0x5555

```

## Challenges

- Deal with concurrency (multiple virtual channels between **C** and **S**, as each channel needs to process the request/response flow sequentially).

    - Prevent double spends via concurrent requests

- Deal with network failures at various stages (assign unique IDs to requests and establish request idempotency). Clients can retry the same request at no cost (if they didn't get back a response due to network failures). Servers can cache the responses for a limited time window.

- Deal with key security - in production scenatios, both clients and servers will most likely choose to delegate custody to a third party (can the hub act as a custodian as well?)
