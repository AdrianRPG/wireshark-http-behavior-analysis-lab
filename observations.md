# Technical Observations – Wireshark HTTP Behavior Analysis

## Overview

This document contains detailed packet-level observations collected while analyzing HTTP behavior with Wireshark for a networking laboratory completed in **CNT 4713 – Net-Centric Computing** at **Florida International University (FIU)**.

The analysis focuses on HTTP behaviors beyond a basic request-response exchange, including:

- HTTP/1.1 request and response structure
- HTTP headers and metadata
- Conditional GET requests
- Browser cache validation
- `304 Not Modified` responses
- TCP segmentation and reassembly
- Embedded web objects
- HTTP requests across multiple servers
- HTTP redirection
- Sequential resource retrieval
- `401 Unauthorized` authentication challenges
- HTTP Basic Authentication
- Security implications of transmitting authentication information over unencrypted HTTP

The packet numbers documented below correspond to the Wireshark trace files used during the lab.

---

## 1. Basic HTTP Request and Response Analysis

### Trace

```text
http-wireshark-trace1-1.pcapng
```

### Objective

The first trace was used to establish the basic HTTP request-response behavior between a web browser and an HTTP server.

The analysis focused on identifying:

- Client and server addresses
- HTTP version
- Request method
- Response status
- HTTP metadata
- Browser language preferences

### Network Endpoints

The HTTP communication occurred between:

```text
Client IP: 10.0.0.44
Server IP: 128.119.245.12
```

The server at:

```text
128.119.245.12
```

corresponds to the HTTP server used by the Wireshark laboratory.

### HTTP Version

Both the request and response used:

```text
HTTP/1.1
```

The client generated an HTTP GET request and the server successfully returned the requested resource.

### Successful HTTP Response

The server responded with:

```text
HTTP/1.1 200 OK
```

The `200 OK` status indicates that the HTTP request was successfully processed and the requested resource was returned.

### Response Metadata

Important response headers included:

```text
Last-Modified: Sat, 30 Jan 2021 06:59:02 GMT
Content-Length: 128
Content-Type: text/html; charset=UTF-8
```

The `Last-Modified` header indicates when the requested resource was last modified on the server.

The `Content-Length` value:

```text
128
```

indicates the size of the HTTP entity body in bytes.

The `Content-Type` value:

```text
text/html; charset=UTF-8
```

identifies the returned object as an HTML document encoded using UTF-8.

### Browser Language Preferences

The HTTP request contained:

```text
Accept-Language: en-US,en;q=0.9
```

This indicates that the client prefers:

1. U.S. English (`en-US`)
2. General English (`en`) with a quality value of `0.9`

This demonstrates how HTTP headers allow clients to communicate content preferences to web servers.

### Key Observation

A basic HTTP transaction follows the application-layer pattern:

```text
Client
  |
  | HTTP GET
  v
Server
  |
  | HTTP/1.1 200 OK
  | Requested object
  v
Client
```

This initial trace established the request-response model used throughout the remainder of the analysis.

---

## 2. HTTP Conditional Requests and Browser Caching

### Trace

```text
http-wireshark-trace2-1.pcapng
```

### Objective

The second trace was analyzed to determine how HTTP caching allows a browser to avoid downloading an unchanged object multiple times.

The trace contained:

1. An initial retrieval of the resource
2. A later conditional request for the same resource

### Initial HTTP Request

The browser initially requested:

```text
GET /wireshark-labs/HTTP-wireshark-file2.html HTTP/1.1
```

The server responded successfully with:

```text
HTTP/1.1 200 OK
```

The response contained:

```text
Content-Length: 371
```

During the first request, the complete object was transferred from the server to the browser.

### Conditional GET Request

The resource was requested again later.

The second request was observed in:

```text
Packet 555
```

The request included cache-validation headers:

```text
If-None-Match: "173-5ba18a7e1ba7e"
If-Modified-Since: Sat, 30 Jan 2021 06:59:02 GMT
```

These headers allow the client to ask the server whether its locally cached copy of the object is still valid.

### `If-Modified-Since`

The header:

```text
If-Modified-Since: Sat, 30 Jan 2021 06:59:02 GMT
```

asks the server to return the complete resource only if the object has been modified after the specified timestamp.

### `If-None-Match`

The request also contained:

```text
If-None-Match: "173-5ba18a7e1ba7e"
```

This value represents the entity tag associated with the cached version of the object.

The browser can use the entity tag to determine whether the server's current representation differs from the cached representation.

### Server Cache-Validation Response

The server response was observed in:

```text
Packet 556
```

The server returned:

```text
HTTP/1.1 304 Not Modified
```

The `304 Not Modified` status indicates that the browser's cached version was still current.

Therefore, the server did not need to retransmit the complete HTML object.

### Observed Flow

```text
Initial Request

Client → GET resource
Server → 200 OK + HTML object

Later Request

Client → Conditional GET
         If-None-Match
         If-Modified-Since

Server → 304 Not Modified
```

### Networking Significance

This behavior reduces unnecessary network traffic.

Instead of downloading the same object again, the client and server exchange only the information required to determine whether the cached copy remains valid.

HTTP conditional requests can therefore reduce:

- Bandwidth consumption
- Server workload
- Repeated object transfers
- Page-loading overhead

### Wireshark Filters Used

Useful filters for isolating this behavior include:

```text
http
```

```text
http.request
```

```text
http.response.code == 304
```

For the specific conditional request-response pair:

```text
frame.number == 555 || frame.number == 556
```

### Evidence

![Conditional GET request](../screenshots/conditional-get-request.png)

*Conditional HTTP GET request containing cache-validation headers including `If-None-Match` and `If-Modified-Since`.*

![304 Not Modified response](../screenshots/conditional-get-response.png)

*Server response confirming that the cached resource remains valid through an HTTP `304 Not Modified` status.*

---

## 3. Large HTTP Response and TCP Reassembly

### Trace

```text
http-wireshark-trace3-1.pcapng
```

### Objective

This trace was analyzed to understand the relationship between HTTP application-layer messages and TCP transport-layer segments.

The requested HTML document was too large to fit inside a single TCP segment.

### HTTP GET Request

The document was requested using one HTTP GET request.

The GET was observed in:

```text
Packet 26
```

Although only one HTTP request was required, the server's HTTP response was transported across multiple TCP segments.

### TCP Segments

Wireshark showed four TCP segments contributing to the complete HTTP response.

| Packet | TCP Payload |
|---:|---:|
| 28 | 1448 bytes |
| 29 | 1448 bytes |
| 31 | 1448 bytes |
| 32 | 517 bytes |

The TCP payload sizes demonstrate that the application-layer HTTP response was divided before being transported across the network.

### TCP Reassembly

Wireshark reported:

```text
4 Reassembled TCP Segments
```

with a total reassembled length of:

```text
Reassembled TCP Length: 4861 bytes
```

The complete reassembled HTTP response was displayed in:

```text
Packet 32
```

### Observed Transport Behavior

```text
HTTP Response
     |
     | Larger than one TCP payload
     v
+-------------+
| TCP Segment |
| Packet 28   |
| 1448 bytes  |
+-------------+
       |
       v
+-------------+
| TCP Segment |
| Packet 29   |
| 1448 bytes  |
+-------------+
       |
       v
+-------------+
| TCP Segment |
| Packet 31   |
| 1448 bytes  |
+-------------+
       |
       v
+-------------+
| TCP Segment |
| Packet 32   |
| 517 bytes   |
+-------------+
       |
       v
TCP Reassembly
       |
       v
Complete HTTP Response
```

### Important Distinction

HTTP and TCP operate at different layers.

HTTP treats the response as one application-layer message.

TCP transports the bytes as a stream and may divide that application data across multiple segments.

Therefore:

```text
1 HTTP response
```

does not necessarily mean:

```text
1 TCP segment
```

In this trace:

```text
1 HTTP response
=
4 TCP segments
```

### Wireshark Filter Used

The relevant frames can be isolated with:

```text
frame.number == 28 ||
frame.number == 29 ||
frame.number == 31 ||
frame.number == 32
```

A general HTTP filter can also be used:

```text
http
```

### Evidence

![TCP reassembly](../screenshots/tcp-reassembly.png)

*Wireshark TCP reassembly information showing that a single HTTP response was reconstructed from four TCP segments with a total reassembled length of 4,861 bytes.*

---

## 4. HTML Documents and Embedded Objects

### Trace

```text
http-wireshark-trace4-1.pcapng
```

### Objective

This trace was analyzed to determine how loading one HTML page can cause a browser to generate multiple HTTP requests for additional resources.

Examples of embedded resources can include:

- Images
- Stylesheets
- Scripts
- Media
- Content hosted by other servers

### Number of HTTP GET Requests

A total of:

```text
4 HTTP GET requests
```

were identified.

Relevant packets included:

```text
Packet 95
Packet 99
Packet 118
Packet 144
```

### Primary Server Requests

Two requests were sent to:

```text
128.119.245.12
```

These requests were associated with the original web page and an embedded resource.

One request retrieved the base HTML document while another retrieved an embedded object.

### Additional Embedded Resource

A later request was sent to:

```text
178.79.137.164
```

The relevant request was observed in:

```text
Packet 118
```

and requested:

```text
/8E_cover_small.jpg
```

This demonstrates that embedded resources referenced by an HTML document do not necessarily need to be hosted on the same server as the original page.

### Redirected Resource

The server at:

```text
178.79.137.164
```

responded with:

```text
HTTP/1.1 301 Moved Permanently
```

The browser subsequently generated another HTTP request toward:

```text
104.98.115.146
```

The new request was observed in:

```text
Packet 144
```

### Observed Resource Flow

```text
Browser
   |
   | GET base HTML document
   v
128.119.245.12
   |
   | HTML references additional objects
   v
Browser
   |
   | GET embedded resource
   v
128.119.245.12

Browser
   |
   | GET /8E_cover_small.jpg
   v
178.79.137.164
   |
   | 301 Moved Permanently
   v
Browser
   |
   | New GET
   v
104.98.115.146
```

### Key Observation

Loading a single page at the application level can generate multiple network transactions.

The browser may contact:

```text
multiple IP addresses
```

and generate:

```text
multiple HTTP GET requests
```

even though the user requested only one web page.

### Wireshark Filters Used

To show only HTTP requests:

```text
http.request
```

To isolate the four relevant requests:

```text
frame.number == 95 ||
frame.number == 99 ||
frame.number == 118 ||
frame.number == 144
```

Useful Wireshark columns for this analysis included:

```text
No.
Time
Source
Destination
Protocol
Info
```

The `Destination` and `Info` columns made it possible to correlate each request with its target server and requested object.

### Evidence

![Embedded object requests](../screenshots/embedded-object-requests.png)

*Multiple HTTP GET requests generated while loading a single web page, demonstrating retrieval of embedded resources from multiple network destinations.*

---

## 5. HTTP Redirection

### Trace

```text
http-wireshark-trace4-1.pcapng
```

### Objective

The embedded-object trace also demonstrated HTTP redirection.

A browser initially requested a resource from one server, received a permanent redirect, and then issued another request toward the redirected destination.

### Initial Request

The original resource request was sent to:

```text
178.79.137.164
```

The request was observed in:

```text
Packet 118
```

and requested:

```text
/8E_cover_small.jpg
```

### Redirect Response

The server responded with:

```text
HTTP/1.1 301 Moved Permanently
```

A `301 Moved Permanently` response informs the client that the requested resource is available at a different location.

HTTP redirection may therefore cause the browser to generate an additional HTTP request automatically.

### Redirected Request

After receiving the redirect, the browser issued another request to:

```text
104.98.115.146
```

The subsequent request was observed in:

```text
Packet 144
```

### Redirect Flow

```text
Client
  |
  | GET /8E_cover_small.jpg
  v
178.79.137.164
  |
  | HTTP/1.1 301 Moved Permanently
  | Location: ...
  v
Client
  |
  | New HTTP GET
  v
104.98.115.146
```

### Networking Significance

A redirect increases the number of HTTP exchanges required to obtain a resource.

Instead of:

```text
GET
↓
200 OK
```

the browser may perform:

```text
GET
↓
301 Moved Permanently
↓
GET redirected location
↓
Final response
```

This additional request-response activity can be observed directly in a packet capture.

### Wireshark Filter Used

A useful filter for locating redirect responses is:

```text
http.response.code == 301
```

### Evidence

![HTTP redirect](../screenshots/http-redirect.png)

*HTTP redirect flow showing an initial request, a `301 Moved Permanently` response, and a subsequent request toward the redirected destination.*

---

## 6. Sequential Embedded Object Retrieval

### Trace

```text
http-wireshark-trace4-1.pcapng
```

### Objective

Packet timing and ordering were examined to determine whether embedded image objects were retrieved sequentially or in parallel.

### Observed Ordering

The observed request-response ordering followed the pattern:

```text
GET first image
        ↓
Response for first image
        ↓
GET second image
```

Because the second image request was not issued until after the first image response had been received, the observed retrieval behavior was identified as:

```text
Sequential
```

rather than parallel.

### Why Packet Timing Matters

Wireshark's packet list can be used to infer browser behavior by comparing:

```text
Packet Number
Timestamp
Source
Destination
Request
Response
```

If two object requests are sent before either response arrives, the behavior may indicate parallel retrieval.

If the browser waits for one response before issuing the next request, the behavior is sequential.

### Conceptual Comparison

#### Sequential Retrieval

```text
GET Object A
     ↓
Response A
     ↓
GET Object B
     ↓
Response B
```

#### Parallel Retrieval

```text
GET Object A
GET Object B
     ↓
Responses arrive independently
```

The packet ordering observed during the lab was consistent with the sequential pattern.

---

## 7. HTTP Basic Authentication

### Trace

```text
http-wireshark-trace5-1.pcapng
```

### Objective

The final trace was analyzed to examine how a browser interacts with a password-protected HTTP resource using Basic Authentication.

The exchange included:

1. An unauthenticated request
2. A `401 Unauthorized` authentication challenge
3. A second request containing authentication information
4. A successful `200 OK` response

### Initial HTTP Request

The first request was observed in:

```text
Packet 92
```

At this point, the request did not contain the authentication information required by the protected resource.

### Authentication Challenge

The server responded in:

```text
Packet 94
```

with:

```text
HTTP/1.1 401 Unauthorized
```

The `401 Unauthorized` status indicates that authentication is required before the server will provide access to the requested resource.

### Authenticated Request

The browser later generated another request in:

```text
Packet 478
```

This request contained:

```text
Authorization: Basic ...
```

The presence of the `Authorization` header distinguishes the authenticated request from the original unauthenticated request.

### Successful Authentication

After receiving the authentication information, the server returned:

```text
HTTP/1.1 200 OK
```

in:

```text
Packet 482
```

This indicates that the authentication information was accepted and access to the protected resource succeeded.

### Complete Authentication Flow

```text
Packet 92
Client → Initial HTTP GET

Packet 94
Server → 401 Unauthorized

Packet 478
Client → HTTP GET
         Authorization: Basic ...

Packet 482
Server → 200 OK
```

Conceptually:

```text
Client
  |
  | GET protected resource
  v
Server
  |
  | 401 Unauthorized
  v
Client
  |
  | GET protected resource
  | Authorization: Basic ...
  v
Server
  |
  | 200 OK
  v
Client
```

### Basic Authentication and Base64

HTTP Basic Authentication represents authentication credentials using Base64 encoding.

An important distinction is:

```text
Base64 encoding ≠ encryption
```

Base64 transforms data into another textual representation, but it does not provide confidentiality.

Therefore, authentication information transported through Basic Authentication over unencrypted HTTP should be protected by using:

```text
HTTPS / TLS
```

in practical deployments.

### Security Observation

Wireshark's ability to inspect:

```text
Authorization: Basic ...
```

within ordinary HTTP traffic demonstrates why authentication information should not be transmitted over an unencrypted connection.

Transport-layer encryption provided by HTTPS/TLS prevents passive packet inspection from directly exposing HTTP headers and application content in this manner.

### Wireshark Filters Used

To isolate the complete observed authentication flow:

```text
frame.number == 92 ||
frame.number == 94 ||
frame.number == 478 ||
frame.number == 482
```

A useful filter for authentication challenges is:

```text
http.response.code == 401
```

HTTP requests can also be isolated with:

```text
http.request
```

### Evidence

![HTTP Basic Authentication flow](../screenshots/basic-authentication-flow.png)

*HTTP Basic Authentication exchange showing the initial request, `401 Unauthorized` challenge, authenticated request containing an `Authorization` header, and successful `200 OK` response.*

---

## Consolidated Packet Observations

| Behavior | Trace | Relevant Packets |
|---|---|---|
| Conditional GET | `http-wireshark-trace2-1.pcapng` | 555 |
| `304 Not Modified` | `http-wireshark-trace2-1.pcapng` | 556 |
| Large HTTP document request | `http-wireshark-trace3-1.pcapng` | 26 |
| TCP response segments | `http-wireshark-trace3-1.pcapng` | 28, 29, 31, 32 |
| Reassembled HTTP response | `http-wireshark-trace3-1.pcapng` | 32 |
| Embedded-object GET requests | `http-wireshark-trace4-1.pcapng` | 95, 99, 118, 144 |
| Initial redirected resource request | `http-wireshark-trace4-1.pcapng` | 118 |
| Redirected request | `http-wireshark-trace4-1.pcapng` | 144 |
| Initial authentication GET | `http-wireshark-trace5-1.pcapng` | 92 |
| `401 Unauthorized` | `http-wireshark-trace5-1.pcapng` | 94 |
| Authenticated GET | `http-wireshark-trace5-1.pcapng` | 478 |
| Authentication success | `http-wireshark-trace5-1.pcapng` | 482 |

---

## Consolidated Protocol Findings

| Analysis Area | Observation |
|---|---|
| HTTP Version | `HTTP/1.1` |
| Client IP | `10.0.0.44` |
| Primary HTTP Server | `128.119.245.12` |
| Successful Response | `200 OK` |
| Cache Validation | `If-Modified-Since` and `If-None-Match` |
| Conditional Response | `304 Not Modified` |
| Cached Object Size | `371 bytes` |
| Large Response TCP Segments | `4` |
| Reassembled TCP Length | `4861 bytes` |
| Embedded HTTP GET Requests | `4` |
| Additional Server | `178.79.137.164` |
| Redirect Destination | `104.98.115.146` |
| Redirect Status | `301 Moved Permanently` |
| Embedded Object Retrieval | Sequential |
| Authentication Challenge | `401 Unauthorized` |
| Authentication Mechanism | HTTP Basic Authentication |
| Authentication Header | `Authorization: Basic ...` |
| Authentication Success | `200 OK` |

---

## Useful Wireshark Display Filters

### All HTTP Traffic

```text
http
```

### HTTP Requests

```text
http.request
```

### HTTP Responses

```text
http.response
```

### `304 Not Modified`

```text
http.response.code == 304
```

### `301 Moved Permanently`

```text
http.response.code == 301
```

### `401 Unauthorized`

```text
http.response.code == 401
```

### Conditional GET Pair

```text
frame.number == 555 || frame.number == 556
```

### TCP Reassembly Frames

```text
frame.number == 28 ||
frame.number == 29 ||
frame.number == 31 ||
frame.number == 32
```

### Embedded Object Requests

```text
frame.number == 95 ||
frame.number == 99 ||
frame.number == 118 ||
frame.number == 144
```

### Authentication Flow

```text
frame.number == 92 ||
frame.number == 94 ||
frame.number == 478 ||
frame.number == 482
```

---

## Key Technical Takeaways

### HTTP Caching Reduces Repeated Transfers

Conditional requests allow browsers to validate cached objects instead of downloading them repeatedly.

The combination of:

```text
If-Modified-Since
If-None-Match
```

with:

```text
304 Not Modified
```

allows a browser to reuse a cached copy when the server confirms that the resource has not changed.

---

### HTTP and TCP Operate at Different Layers

An HTTP message may represent one application-layer response while being transported through several TCP segments.

The large-response trace demonstrated:

```text
1 HTTP response
        ↓
4 TCP segments
        ↓
TCP reassembly
        ↓
1 reconstructed HTTP message
```

This reinforces the distinction between:

```text
Application Layer: HTTP
Transport Layer: TCP
```

---

### Web Pages Can Generate Multiple HTTP Requests

A single HTML document can reference several additional resources.

Therefore:

```text
1 user page request
```

can result in:

```text
multiple HTTP transactions
```

and communication with:

```text
multiple remote servers
```

The embedded-object trace demonstrated this behavior directly.

---

### Redirects Generate Additional Network Traffic

An HTTP redirect introduces another request-response step.

The observed behavior included:

```text
GET
↓
301 Moved Permanently
↓
redirected GET
```

This demonstrates how HTTP response status codes can directly influence subsequent browser behavior.

---

### Packet Ordering Can Reveal Browser Behavior

The order and timestamps of packets can be used to determine whether resources are retrieved sequentially or concurrently.

This allows packet captures to reveal browser networking behavior that may not be visible from the browser interface alone.

---

### Basic Authentication Requires Transport Protection

HTTP Basic Authentication uses Base64 encoding rather than encryption.

Therefore:

```text
Basic Authentication + HTTP
```

does not provide confidentiality for authentication information.

A more secure deployment uses:

```text
Basic Authentication + HTTPS/TLS
```

so that the HTTP request, headers, and credentials are transported through an encrypted TLS connection.

---

## Skills Practiced

This lab provided hands-on practice with:

- Wireshark packet analysis
- HTTP/1.1
- TCP/IP networking
- HTTP GET requests
- HTTP response analysis
- HTTP request and response headers
- HTTP status codes
- Client-server communication
- Browser cache validation
- Entity tags
- `If-None-Match`
- `If-Modified-Since`
- `304 Not Modified`
- TCP segmentation
- TCP reassembly
- Embedded web resources
- Multi-server HTTP communication
- HTTP redirection
- `301 Moved Permanently`
- Packet timing analysis
- Sequential resource retrieval
- HTTP authentication challenges
- `401 Unauthorized`
- HTTP Basic Authentication
- Base64 security considerations
- HTTPS/TLS security reasoning
- Wireshark display filters
- Packet-level troubleshooting
- Application-layer protocol analysis
- Transport-layer protocol analysis

---

## Repository Evidence

The repository contains the following supporting screenshots:

```text
screenshots/
├── conditional-get-request.png
├── conditional-get-response.png
├── tcp-reassembly.png
├── embedded-object-requests.png
├── http-redirect.png
└── basic-authentication-flow.png
```

Each screenshot was selected to document a specific protocol behavior rather than reproduce assignment questions, grading material, or textbook-owned content.

---

## Academic Context

This analysis originated from coursework completed for:

**CNT 4713 – Net-Centric Computing**  
Florida International University  
Knight Foundation School of Computing and Information Sciences

The laboratory used Wireshark packet traces associated with the networking laboratory materials accompanying:

**James F. Kurose and Keith W. Ross**  
*Computer Networking: A Top-Down Approach*, 8th Edition

The original packet traces and laboratory materials were created by the textbook authors.

This document contains my own packet observations, protocol interpretations, technical explanations, and analysis produced while completing the laboratory.

Original packet trace files, assignment questions, instructor materials, grading content, and textbook-owned laboratory materials are not redistributed in this repository.

---

## Final Summary

This lab demonstrated that HTTP communication involves significantly more than simple GET and `200 OK` exchanges.

Packet-level inspection showed how HTTP uses:

- Conditional requests
- Browser cache validation
- `304 Not Modified`
- TCP segmentation
- TCP reassembly
- Embedded resource requests
- Multiple remote servers
- HTTP redirection
- Sequential resource retrieval
- Authentication challenges
- HTTP Basic Authentication

The analysis also demonstrated how Wireshark can connect application-layer HTTP behavior with the TCP transport mechanisms carrying those messages.

The traces provided direct evidence of how protocol behavior can affect both network efficiency and security.

HTTP caching reduced unnecessary resource transfers, TCP reassembly demonstrated how large application messages are transported across multiple segments, embedded resources generated additional network requests, redirects changed subsequent client behavior, and Basic Authentication illustrated the importance of protecting authentication information with HTTPS/TLS.
