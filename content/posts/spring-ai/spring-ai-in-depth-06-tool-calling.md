---
title: "Spring AI Series: 6-Tool Calling"
date: 2026-07-10
draft: false
tags: ["Java", "Spring Boot", "AI", "Spring AI"]
cover:
  image: "/images/spring-ai-06-tool-calling.png"
  alt: "Spring AI Series: Tool Calling in Spring AI"
series: ["Spring AI in Depth"]
series_order: 6
description: "Give the model hands. Let BrightCart's classifier call real Java methods to look up live order data from a database, and learn how the tool-calling loop actually works."
categories: ["ai", "java"]
---
At the end of the last article, BrightCart's classifier did something impressive and something useless, at the same time.

Impressive: it read a chaotic customer rant and pulled out `orderNumber: "BC-48291"` as a clean, typed field. Useless: it had absolutely no idea whether order BC-48291 exists, what's in it, or where it is. It extracted that order number the same way it would extract `BC-99999` or `BC-BANANA` — by pattern-matching text. The model was working entirely from the customer's words, and customers, as we've established, are not a reliable source of truth.

To actually help with "where is my order," BrightCart needs to reach past the model and into its own systems — the order database, specifically. And here's the catch that makes this article necessary: *the model can't do that on its own.* An LLM is a text predictor frozen at its training cutoff. It has never heard of BrightCart, has no database connection, and could not run a SQL query if its loss function depended on it.

Tool calling is how we bridge that gap. We give the model a set of Java methods it's allowed to *request*, and when it decides it needs real data, it asks our application to run one. Today we give BrightCart hands.

Full source, as always, in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth) on the `06-tool-calling` branch.

## How Tool Calling Actually Works

Before we write a line of code, let's kill the most common misconception about tool calling, because it leads people badly astray.

> **Concept Primer — The Model Requests, It Doesn't Execute**
> When you hear "the AI called a function," it sounds like the model reached into your application and ran your code. It didn't. It *can't*. What actually happens is a negotiation. You tell the model, up front, "here are some tools you may use, here's what each one does and what arguments it takes." When the model decides it needs one, it doesn't run anything — it pauses and emits a structured message that says, in effect, "please run `getOrderByNumber` with argument `BC-48291` and tell me what you get back." Your application executes the method, captures the return value, and hands it back to the model. The model then continues, now armed with real data. The code always runs on your side, under your control. The model only ever *asks*.

That distinction matters because it's also the security model. The AI never has your database credentials, never touches your systems directly, never executes anything. It makes requests; your code decides whether and how to honor them. We'll come back to the implications of that near the end.

Here's the full loop, step by step, for a ticket about order BC-48291:

1. **You send the prompt plus tool definitions.** Along with the ticket text, Spring AI sends the model a description of every available tool — names, descriptions, parameter schemas. The model now knows `getOrderByNumber(String orderNumber)` exists and what it's for.

2. **The model responds with a tool request.** Instead of a normal text answer, the model replies "I need you to call `getOrderByNumber` with `BC-48291`." No prose, just a structured request.

3. **Spring AI executes the tool.** This is where your Java runs. Spring AI matches the request to your annotated method, calls it with the supplied argument, and your method hits the database and returns an `Order`.

4. **The result goes back to the model.** Spring AI serializes your `Order` to JSON and appends it to the conversation, then calls the model again — now with the original ticket, the tool request, *and* the real order data.

5. **The model produces its final answer**, grounded in the actual order. Or, if it needs another tool, it requests one and the loop repeats from step 2.

The beautiful part: in Spring AI, steps 2 through 4 happen *automatically*. You write the tool method and register it. When the model requests a call, Spring's tool-calling machinery executes it and loops back without you writing any of the orchestration. You'll see this in the logs shortly — it's oddly satisfying to watch.

Now let's build it.

## A Tool Is Just a Method

Here's the part that makes Spring AI's tool calling genuinely pleasant: a tool is an ordinary Java method with an annotation on it. No special base class, no interface to implement, no registry to configure. You write the method you'd write anyway, add `@Tool` with a description, and you're done.

```java
@Tool(description = "Look up a BrightCart order by its order number")
public Order getOrderByNumber(String orderNumber) {
    return orderRepository.findByOrderNumber(orderNumber);
}
```

That `description` is not a comment. It's the single most important line in the whole tool, because it's what the *model* reads to decide when to call this method. The model never sees your method body or your variable names — it sees the description and the parameter schema. Write the description for the model, as documentation of *when and why* to use the tool, and you'll get reliable calls. Write it lazily ("gets order") and the model will guess, sometimes wrong.

Before we can write the tool for real, though, it needs something to look data up *in*. Let's give BrightCart an actual order database.

## The Data Layer

Nothing here is Spring AI — it's the Spring Data JPA you already know, so we'll move quickly. An `Order` entity:

```java
package org.veenx.springai.demo.order;

import jakarta.persistence.Entity;
import jakarta.persistence.Id;

import java.time.LocalDate;

@Entity
@jakarta.persistence.Table(name = "orders")
public class Order {

    @Id
    private String orderNumber;

    private String product;
    private String status;
    private LocalDate orderDate;
    private LocalDate estimatedDelivery;
    private String customerEmail;

    protected Order() {
        // JPA
    }

    public Order(String orderNumber, String product, String status,
                 LocalDate orderDate, LocalDate estimatedDelivery, String customerEmail) {
        this.orderNumber = orderNumber;
        this.product = product;
        this.status = status;
        this.orderDate = orderDate;
        this.estimatedDelivery = estimatedDelivery;
        this.customerEmail = customerEmail;
    }

    public String getOrderNumber() { return orderNumber; }
    public String getProduct() { return product; }
    public String getStatus() { return status; }
    public LocalDate getOrderDate() { return orderDate; }
    public LocalDate getEstimatedDelivery() { return estimatedDelivery; }
    public String getCustomerEmail() { return customerEmail; }
}
```

(A quick note on the table name: `order` is a reserved SQL keyword, so we map the entity to a table called `orders`. The kind of detail that costs you twenty minutes the first time you hit it.)

The repository:

```java
package org.veenx.springai.demo.order;

import org.springframework.data.jpa.repository.JpaRepository;

public interface OrderRepository extends JpaRepository<Order, String> {
    Order findByOrderNumber(String orderNumber);
}
```

Add the JPA and H2 dependencies to your `pom.xml`:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

And seed a few orders at startup so we have something to look up — including our now-infamous BC-48291:

```java
package org.veenx.springai.demo.order;

import org.springframework.boot.CommandLineRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.time.LocalDate;

@Configuration
public class OrderSeedData {

    @Bean
    CommandLineRunner seedOrders(OrderRepository repository) {
        return args -> {
            repository.save(new Order("BC-48291", "Deluxe Espresso Machine", "DELIVERED",
                    LocalDate.of(2026, 6, 1), LocalDate.of(2026, 6, 3), "rita@example.com"));
            repository.save(new Order("BC-10233", "Wireless Headphones", "IN_TRANSIT",
                    LocalDate.of(2026, 6, 14), LocalDate.of(2026, 6, 19), "sam@example.com"));
            repository.save(new Order("BC-77510", "Standing Desk", "PROCESSING",
                    LocalDate.of(2026, 6, 16), LocalDate.of(2026, 6, 24), "lena@example.com"));
        };
    }
}
```

Notice something already: BC-48291 has status `DELIVERED`, but our customer swears it never arrived. The tool is about to let the model spot exactly that kind of discrepancy — the sort of thing that turns a vague ticket into an actionable one.

## The Order Lookup Tool

Now the tool itself. It's a Spring `@Component` that wraps the repository and exposes one `@Tool`-annotated method:

```java
package org.veenx.springai.demo.order;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.ai.tool.annotation.Tool;
import org.springframework.ai.tool.annotation.ToolParam;
import org.springframework.stereotype.Component;

@Component
public class OrderTools {

    private static final Logger log = LoggerFactory.getLogger(OrderTools.class);

    private final OrderRepository orderRepository;

    public OrderTools(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    @Tool(description = """
            Look up a BrightCart order by its order number. Returns the product,
            current status, order date, estimated delivery date, and customer email.
            Use this whenever a ticket references a specific order number to verify
            the order's real status.
            """)
    public Order getOrderByNumber(
            @ToolParam(description = "The order number, e.g. BC-48291") String orderNumber) {
        log.info("Tool called: getOrderByNumber({})", orderNumber);
        return orderRepository.findByOrderNumber(orderNumber);
    }
}
```

Two annotations doing the work. `@Tool` describes the method to the model — note the description tells it not just *what* the tool does but *when* to reach for it. `@ToolParam` describes the parameter, giving the model an example format so it passes `BC-48291` and not `order number 48291` or `the espresso machine order`. The `log.info` isn't decoration; it's how we'll prove the model actually called the tool rather than making something up.

The return type is our `Order` entity. Spring AI serializes it to JSON automatically before feeding it back to the model — records, entities, plain POJOs all work, as long as they're serializable.

### Registering the Tool

A tool the model doesn't know about is just a method. We make it available by passing it to the `ChatClient` via `.tools()`. You can do this per-request or, more commonly for a tool that's always relevant, as a default on the client.

Here's an enrichment service that combines everything — it asks the model to analyze a ticket *and* gives it the order tool so it can verify claims against real data:

```java
package org.veenx.springai.demo.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.stereotype.Service;
import org.veenx.springai.demo.order.OrderTools;

@Service
public class TicketEnrichmentService {

    private final ChatClient chatClient;
    private final OrderTools orderTools;

    public TicketEnrichmentService(ChatClient.Builder builder, OrderTools orderTools) {
        this.orderTools = orderTools;
        this.chatClient = builder
                .defaultSystem("""
                        You are a support assistant for BrightCart, an online retailer.
                        When a ticket references an order number, use the available tools
                        to look up the real order and compare it against what the customer
                        claims. Point out any discrepancies (for example, an order marked
                        delivered that the customer says never arrived).
                        Respond with a short, factual briefing for the support agent.
                        """)
                .defaultTools(orderTools)
                .build();
    }

    public String enrich(String ticketText) {
        return chatClient
                .prompt()
                .user(ticketText)
                .call()
                .content();
    }
}
```

`defaultTools(orderTools)` registers every `@Tool`-annotated method on that object with this client. From now on, every call through this `ChatClient` can use the order lookup — the model decides whether it needs to.

A controller to drive it:

```java
package org.veenx.springai.demo.web;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.veenx.springai.demo.service.TicketEnrichmentService;

@RestController
@RequestMapping("/api/tickets")
public class EnrichmentController {

    private final TicketEnrichmentService enrichmentService;

    public EnrichmentController(TicketEnrichmentService enrichmentService) {
        this.enrichmentService = enrichmentService;
    }

    @PostMapping("/enrich")
    public String enrich(@RequestBody String ticketText) {
        return enrichmentService.enrich(ticketText);
    }
}
```

## Watching the Loop Run

This is the fun part. Fire the espresso machine ticket at the new endpoint:

```bash
curl -X POST http://localhost:8080/api/tickets/enrich \
     -H "Content-Type: text/plain" \
     -d "I ordered the deluxe espresso machine (order BC-48291) two weeks ago. Tracking said delivered but I never got it. I want a refund or a replacement!"
```

Now look at the application logs while it runs:

```
Tool called: getOrderByNumber(BC-48291)
```

There it is. The model read the ticket, recognized it referenced an order, and *requested* the `getOrderByNumber` tool with the right argument — which Spring AI executed against H2, entirely automatically. We didn't write any code to detect the order number, decide to call the tool, or feed the result back. The model drove; Spring handled the mechanics.

And the response the agent receives:

```markdown
**Order Lookup Summary:**

| Detail | Information |
|--------|-------------|
| **Order Number** | BC-48291 |
| **Product** | Deluxe Espresso Machine |
| **Order Date** | June 1, 2026 |
| **Status** | DELIVERED |
| **Estimated Delivery** | June 3, 2026 |
| **Customer Email** | rita@example.com |

**Key Discrepancy:** The system shows this order as DELIVERED, but the customer reports never receiving it. The order was placed about two weeks ago and shows as delivered well within the estimated window.

**Recommended Action:** This is a legitimate non-delivery claim. Initiate an investigation into the delivery and offer either a replacement shipment or refund per company policy. You may want to:
- Contact the carrier for delivery confirmation details
- Request the customer check with neighbors/building management if applicable
- If delivery cannot be verified, proceed with replacement or refund authorization
```

Your output may differ depending on the model you use. But Haiku even gave us a neat table with the main fields. Compare this to the classifier's output from article 5. The classifier could only tell us what the *customer* said. This briefing tells us what's *actually true in the system* — and flags the contradiction between the two. That's the leap tool calling provides: the model stops summarizing claims and starts checking them.

Trace the loop one more time now that you've seen it work: prompt + tool definitions went out, the model requested `getOrderByNumber(BC-48291)`, Spring executed it and got a `DELIVERED` order back, that order was fed to the model, and the model wrote its final briefing noticing the DELIVERED-vs-never-arrived contradiction. Five steps, most of them invisible, exactly as the primer described.

## Adding More Tools Is More of the Same

Once you've written one tool, the rest are pure repetition — which is the point. Say BrightCart also wants shipment tracking and customer history available. Same pattern: a method, a description, register it. No new concepts.

A shipment tool (backed by whatever tracking service you have — a repository, an external API client, doesn't matter to the model):

```java
@Tool(description = """
        Get the detailed shipment tracking history for an order number,
        including carrier scans and current location. Use when a customer
        asks where their package is or disputes a delivery.
        """)
public ShipmentTracking getShipmentTracking(
        @ToolParam(description = "The order number, e.g. BC-48291") String orderNumber) {
    log.info("Tool called: getShipmentTracking({})", orderNumber);
    return shipmentService.trackByOrderNumber(orderNumber);
}
```

A customer lookup tool:

```java
@Tool(description = """
        Look up a customer's account and order history by email address.
        Use to check whether a customer is a repeat buyer or has prior issues.
        """)
public CustomerProfile getCustomerByEmail(
        @ToolParam(description = "The customer's email address") String email) {
    log.info("Tool called: getCustomerByEmail({})", email);
    return customerService.findByEmail(email);
}
```

Drop these on the same `OrderTools` component (or a new `@Component` — your call), and they're registered by the same `defaultTools(...)`. Now when a ticket says "where's my package," the model reaches for `getShipmentTracking`; when it needs to know if this is a VIP, it calls `getCustomerByEmail`. It even chains them: look up the order, see the customer email, look up the customer — each result informing the next request, all within one `.call()`. You wrote three methods; the model composes them.

That's the whole model of tool calling in Spring AI. Write methods, describe them well, register them. The intelligence about *when* to use them lives in the model, and the descriptions are how you steer it.

## A Necessary Word of Caution

Everything above should also make you slightly nervous, and that's healthy.

We just handed a language model the ability to trigger real methods in our application based on its own judgment about a customer's free-text message. Our tools so far are *read-only* — the worst case is looking up the wrong order. But the same mechanism can just as easily wrap a method that issues a refund, cancels an order, or emails a customer. The moment a tool has side effects, "the model decides when to call it" stops being a convenience and becomes a genuine risk surface: a cleverly-worded ticket could try to talk the model into calling something it shouldn't.

This is a big enough topic that it gets proper treatment later. In **article 13, the production article**, we'll dig into tool-calling safety in depth — the distinction between read-only and side-effecting tools, how to guard tools that touch real systems, keeping humans in the loop for consequential actions, and defending against prompt injection through ticket content. For now, the rule of thumb: read-only tools like today's are low-risk and a great place to start; anything that *changes* state deserves the caution we'll cover then. I'm flagging it here so it's on your radar from the first tool you write, not bolted on later.

## What's Next

BrightCart's assistant can now check its own systems. It reads a ticket, decides it needs real data, calls a Java method to get it, and grounds its answer in what's actually true. The gap between "what the customer claims" and "what the database says" is now visible to the system — which is exactly what a support agent needs.

There's a thread worth pulling in the next article. That automatic tool-calling loop — the thing that executed our method and fed the result back without us writing any orchestration — is powered by a piece of Spring AI machinery called an *advisor*. Advisors are the middleware layer of Spring AI: interceptors that sit in the request/response flow and add behavior. Tool calling is one built on top of them; so is chat memory, which we'll need very soon.

In article 7, we pull back the curtain on advisors — what they are, how the chain works, and how to write your own for cross-cutting concerns like logging, rate limiting, and sanitizing the occasionally-hostile contents of a support ticket.

The model has hands now. Next, we give it a nervous system.

*This is part 6 of a 13-part series.*

[← Previous: Spring AI Series: 5-Structured Outputs]
[Next: Spring AI Series: 7-Advisors: Spring AI's Middleware Layer →]