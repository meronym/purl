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

    - **C** attaches the cheque as a `Payment` header to the request and sends it to the server.

3. **S** validates the `Payment` header and processes the request.

    - **S** deducts the price for the request and generates an updated cheque that refunds the difference to the client.

    - **S** attaches the cheque as a `Payment` header to the response.

 
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

