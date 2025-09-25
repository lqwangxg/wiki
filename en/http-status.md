---
title: 🌐HTTP Status Codes and Their Meanings 
description: 🌐HTTP Status Codes and Their Meanings 
published: 1
date: 2024-12-14T14:07:48.080Z
tags: http, https, status
editor: markdown
dateCreated: 2024-12-14T14:07:48.080Z
---

# HTTP Status Codes and Their Meanings 🌐

[English](/en/http-status.md) | [Japanese](/ja/http-status.md) | [Chinese](/zh/http-status.md)

Below is an explanation of HTTP status codes and the situations in which they are returned.

## 1xx: Informational Status Codes
1xx codes indicate that the request has been received and processing should continue.

- **100 Continue**
  - **Meaning**: The server has received the request headers, and the client should continue sending the body.
  - **Scenario**: Sending a large file in chunks; the server confirms to proceed.

- **101 Switching Protocols**
  - **Meaning**: The server agrees to switch protocols as requested by the client.
  - **Scenario**: Switching from HTTP to WebSocket.

---

## 2xx: Successful Status Codes ✅
2xx codes indicate that the request was successfully received, understood, and accepted.

- **200 OK**
  - **Meaning**: The request was successful, and the server is returning the data.
  - **Scenario**: GET request for data, POST request confirmation.

- **201 Created**
  - **Meaning**: The request succeeded, and a resource was created.
  - **Scenario**: Creating a new resource via POST.

- **202 Accepted**
  - **Meaning**: The request has been accepted but is not yet completed.
  - **Scenario**: Asynchronous operations requiring more processing time.

- **204 No Content**
  - **Meaning**: The request was successful, but the server is returning no content.
  - **Scenario**: Deleting a resource without needing to return data.

---

## 3xx: Redirection Status Codes 🔀
3xx codes indicate that the client must take additional action to complete the request.

- **301 Moved Permanently**
  - **Meaning**: The resource has been permanently moved to a new location.
  - **Scenario**: URL restructuring with a permanent redirect.

- **302 Found**
  - **Meaning**: The resource is temporarily available at a different location.
  - **Scenario**: Temporary redirection for maintenance.

- **304 Not Modified**
  - **Meaning**: The resource has not been modified; the client can use its cached version.
  - **Scenario**: Using ETag or other cache validation headers.

---

## 4xx: Client Error Status Codes ❌
4xx codes indicate an error caused by the client.

- **400 Bad Request**
  - **Meaning**: The request is invalid or malformed.
  - **Scenario**: Missing parameters or incorrect JSON format.

- **401 Unauthorized**
  - **Meaning**: Authentication credentials are missing or invalid.
  - **Scenario**: Accessing a resource without valid login or token.

- **403 Forbidden**
  - **Meaning**: The server refuses to process the request.
  - **Scenario**: Attempting to access a resource without proper permissions.

- **404 Not Found**
  - **Meaning**: The requested resource could not be found.
  - **Scenario**: Accessing a nonexistent URL or a deleted resource.

- **405 Method Not Allowed**
  - **Meaning**: The HTTP method used is not supported.
  - **Scenario**: Using POST on an endpoint that only supports GET.

---

## 5xx: Server Error Status Codes ⚠️
5xx codes indicate a server-side problem.

- **500 Internal Server Error**
  - **Meaning**: The server encountered an unexpected condition.
  - **Scenario**: Bugs in the server code or crashes.

- **502 Bad Gateway**
  - **Meaning**: A gateway or proxy server received an invalid response.
  - **Scenario**: Reverse proxies unable to connect to backend services.

- **503 Service Unavailable**
  - **Meaning**: The server is temporarily unavailable (overloaded or under maintenance).
  - **Scenario**: High traffic or maintenance periods.

- **504 Gateway Timeout**
  - **Meaning**: A gateway or proxy server timed out while waiting for a response.
  - **Scenario**: Slow or unresponsive backend services.
