---
title: "JEP 517: HTTP/3 is Finally Here (Natively)!"
date: 2026-01-29
draft: false
tags: ["Java", "JEP", "Java 26", "HTTP-Client", "HTTP/3"]
---

![Final](/images/jep516-HTTP3-for-the-HTTP-Client-API.png)


If you’ve been waiting for Java to catch up with the modern web, the wait is over. Java 26 updates the `java.net.http.HttpClient` (the one we’ve loved since JDK 11) to officially support **HTTP/3**.

Here is an example of how to use the API
```java
var client = HttpClient.newHttpClient(); 
var request = HttpRequest.newBuilder(URI.create("https://openjdk.org/"))
                  .GET()
                  .build(); 
var response = client.send(request, HttpResponse.BodyHandlers.ofString()); 
assert response.statusCode() == 200; 
String htmlText = response.body();
```


**Why should you care?**
HTTP/3 isn't just a version bump; it swaps out TCP for **QUIC** (built on UDP). This is a big deal for performance because it solves "head-of-line blocking"—where one lost packet holds up the entire line of data. It basically means faster handshakes and much more reliable connections, especially if your users are on flaky networks with high packet loss.

**How do I use it?**
Here is the good news: you don't need to learn a new API. However, because HTTP/3 isn't supported everywhere yet (only about a third of websites use it), Java won't use it by default. You have to **opt-in**.

You can enable it globally for your client:

```java
var client = HttpClient.newBuilder()
    .version(HttpClient.Version.HTTP_3)  // Request HTTP/3
    .build();
```


Or, you can enable it for just a specific request:

```java
var request = HttpRequest.newBuilder(URI.create("https://openjdk.org/"))
    .version(HttpClient.Version.HTTP_3)  // Try HTTP/3 for this call only
    .GET()
    .build();
```


**What if the server doesn't support it?**
Don't worry about your code crashing. The API is smart enough to handle discovery. If the target server doesn't support HTTP/3, the client will **transparently downgrade** to HTTP/2 or HTTP/1.1 automatically. You get the speed if it's available, and safety if it's not.