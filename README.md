# Wireshark HTTP Behavior Analysis Lab

## Lab Overview

This lab presents a packet-level analysis of HTTP communication using Wireshark. It builds upon introductory HTTP traffic inspection by examining more advanced protocol behaviors, including conditional requests, browser caching, TCP segmentation and reassembly, embedded web objects, HTTP redirection, and Basic Authentication. The lab was completed as part of **CNT 4713 – Net-Centric Computing** at **Florida International University (FIU)** under the instruction of **Professor Xavier Caddle**.

---

## Objective

The objective of this lab was to strengthen practical understanding of how HTTP operates over TCP by inspecting real packet exchanges between a web client and multiple HTTP servers.

The analysis focused on identifying HTTP request and response behavior, interpreting protocol headers and status codes, examining cache validation mechanisms, tracing embedded resources across multiple servers, and observing how larger HTTP messages are transported across multiple TCP segments.

---

## Technologies & Tools

| Technology | Purpose |
|---|---|
| Wireshark | Packet capture inspection and protocol analysis |
| TCP | Transport protocol carrying HTTP messages |
| IPv4 | Network-layer addressing |
| PCAPNG | Packet capture trace format |

---

## Lab Methodology

### 1. Basic HTTP Request and Response Analysis

The initial trace was examined to identify the communication between the client and the HTTP server.

The browser sent an HTTP/1.1 GET request from:

```text
Client IP: 10.0.0.44
Server IP: 128.119.245.12
```

Both the client request and server response used:

```text
HTTP/1.1
```

The server successfully returned the requested resource with:

```text
HTTP/1.1 200 OK
```

The response included metadata such as:

```text
Last-Modified: Sat, 30 Jan 2021 06:59:02 GMT
Content-Length: 128
Content-Type: text/html; charset=UTF-8
```

The request also advertised the browser's language preferences:

```text
Accept-Language: en-US,en;q=0.9
```

This indicates a preference for U.S. English, followed by general English with a lower quality value.

---

### 2. HTTP Conditional Requests and Browser Caching

A second trace demonstrated how HTTP avoids unnecessarily transferring an unchanged resource.

During the first request, the browser retrieved the resource normally:

```text
GET /wireshark-labs/HTTP-wireshark-file2.html HTTP/1.1
```

The server returned:

```text
HTTP/1.1 200 OK
Content-Length: 371
```

The response contained the complete HTML object.

When the browser requested the resource again, it included cache-validation headers:

```text
If-None-Match: "173-5ba18a7e1ba7e"
If-Modified-Since: Sat, 30 Jan 2021 06:59:02 GMT
```

The server determined that the cached resource was still current and responded with:

```text
HTTP/1.1 304 Not Modified
```

No HTML entity body was retransmitted.

This behavior demonstrates how HTTP conditional requests can reduce unnecessary network traffic by allowing the browser to reuse previously cached content.

![Conditional GET request](screenshots/conditional-get-request.png)

![304 Not Modified response](screenshots/conditional-get-response.png)

---

### 3. Large HTTP Response and TCP Reassembly

The lab also examined retrieval of a larger HTML document.

Only one HTTP GET request was required to retrieve the document, but the resulting HTTP response could not be transported within a single TCP segment.

Wireshark identified four TCP segments contributing to the reassembled response:

| Packet | TCP Payload |
|---:|---:|
| 28 | 1448 bytes |
| 29 | 1448 bytes |
| 31 | 1448 bytes |
| 32 | 517 bytes |

Wireshark reported:

```text
4 Reassembled TCP Segments
Reassembled TCP Length: 4861 bytes
```

The final reassembled HTTP response was displayed in packet 32.

This illustrates the distinction between application-layer messages and transport-layer segmentation: a single HTTP response may be divided across several TCP segments before being reconstructed at the receiving endpoint.

![TCP reassembly](screenshots/tcp-reassembly.png)

---

### 4. HTML Documents and Embedded Objects

Another trace demonstrated how loading a single web page can generate multiple HTTP requests.

A total of four HTTP GET requests were observed.

Two requests were sent to:

```text
128.119.245.12
```

These retrieved the base HTML document and one embedded image.

Another request was sent to:

```text
178.79.137.164
```

The server responded with:

```text
HTTP/1.1 301 Moved Permanently
```

The browser then issued an additional request to:

```text
104.98.115.146
```

This demonstrates that a web page can retrieve embedded resources from multiple servers and that HTTP redirects can generate additional network requests.

![Embedded object requests](screenshots/embedded-object-requests.png)

---

### 5. Sequential Embedded Object Retrieval

Packet timestamps were compared to determine whether embedded image objects were requested serially or in parallel.

The relevant sequence was:

```text
GET first image
        ↓
200 OK / first image received
        ↓
GET second image
```

Because the second image request was not issued until after the first image response had been received, the observed image retrieval occurred sequentially rather than in parallel.

This demonstrates how packet timestamps and request ordering can be used to infer browser network behavior.

---

### 6. HTTP Basic Authentication

The final trace examined access to a password-protected HTTP resource.

The initial request followed this sequence:

```text
Client → GET protected resource
Server → 401 Unauthorized
```

The browser then issued another request containing an additional HTTP header:

```text
Authorization: Basic ...
```

After receiving the authentication information, the server responded with:

```text
HTTP/1.1 200 OK
```

The complete observed exchange was:

```text
Packet 92  → Initial HTTP GET
Packet 94  ← 401 Unauthorized
Packet 478 → Authenticated HTTP GET
Packet 482 ← 200 OK
```

HTTP Basic Authentication encodes credentials using Base64. Base64 provides encoding rather than encryption, meaning credentials sent through Basic Authentication should be protected by HTTPS/TLS when used in practice.

![HTTP Basic Authentication flow](screenshots/basic-authentication-flow.png)

---

## Key Results

| Analysis Area | Result |
|---|---|
| HTTP version | HTTP/1.1 |
| Client IP | 10.0.0.44 |
| Primary server IP | 128.119.245.12 |
| Successful response | 200 OK |
| Conditional response | 304 Not Modified |
| Redirect response | 301 Moved Permanently |
| Authentication challenge | 401 Unauthorized |
| Cached object size | 371 bytes |
| Large response TCP segments | 4 |
| Reassembled TCP length | 4861 bytes |
| Embedded-resource GET requests | 4 |
| Image retrieval behavior | Sequential |
| Authentication header | Authorization |

---

## Key Concepts Analyzed

This project provided practical exposure to:

- HTTP GET requests and response messages
- HTTP status codes and response phrases
- Request and response headers
- Browser language preferences
- HTTP caching and cache validation
- `Last-Modified` and `If-Modified-Since`
- Entity tags (`ETag` and `If-None-Match`)
- `304 Not Modified` responses
- TCP segmentation and reassembly
- HTTP embedded objects
- Multi-server resource retrieval
- `301 Moved Permanently` redirection
- Sequential HTTP resource retrieval
- HTTP `401 Unauthorized` responses
- HTTP Basic Authentication
- Security limitations of authentication over unencrypted HTTP

---

## Skills Demonstrated

Through this lab, I practiced using Wireshark to isolate HTTP traffic, navigate protocol headers, correlate requests with responses, interpret HTTP status codes, analyze TCP reassembly, trace resource requests across multiple servers, and evaluate authentication-related network behavior.

The lab strengthened my understanding of the relationship between application-layer HTTP messages and the underlying TCP transport mechanism while reinforcing practical packet-analysis techniques relevant to networking and cybersecurity.

---

## Academic Context

This project was completed as part of coursework in:

**CNT 4713 – Net-Centric Computing**  
Florida International University  
Knight Foundation School of Computing and Information Sciences

The activity was used as a learning exercise to reinforce concepts related to HTTP, TCP/IP networking, packet analysis, web communication, and network security.

---

## Source & Attribution

The packet traces and original laboratory exercise were provided as part of the Wireshark HTTP Lab materials associated with:

James F. Kurose and Keith W. Ross,  
*Computer Networking: A Top-Down Approach*, 8th Edition.

The original Wireshark laboratory materials and packet traces were created by the textbook authors.

This repository contains my own analysis, observations, documentation, and screenshots produced while completing the laboratory exercise.

Original trace files are not redistributed in this repository.

---

## Disclaimer

This repository is intended for educational and portfolio purposes. It documents my learning process and packet-analysis methodology and is not intended to reproduce or distribute copyrighted assignment material, textbook content, or solution sets.

---

## Conclusion

This lab expanded my understanding of HTTP beyond basic request-response communication by examining how modern web interactions depend on caching, conditional requests, TCP segmentation, embedded resources, redirection, and authentication.

Using Wireshark made it possible to observe these mechanisms directly at the packet level and connect networking concepts from coursework with their actual behavior on a TCP/IP network.

The project also reinforced how packet analysis can be used to investigate both network performance behavior and security-relevant protocol characteristics.
