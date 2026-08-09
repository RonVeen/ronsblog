---
title: "Spring AI Series: 8-Chat Memory and Conversation State"
date: 2026-08-14
draft: true
tags: ["Java", "Spring Boot", "AI", "Spring AI"]
cover:
  image: "/images/spring-ai-08-chat-memory.png"
  alt: "Spring AI Series: Chat Memory and Conversation State"
series: ["Spring AI in Depth"]
series_order: 8
description: "Learn how Spring AI's chat memory advisors create the illusion of conversation, and give BrightCart persistent per-ticket memory backed by a database."
categories: ["ai", "java"]
---

Everything BrightCart has built so far has a peculiar kind of amnesia. Each request lands, gets handled, and vanishes from the system's mind completely. The summarizer, the classifier, the enrichment service. Every single call starts from a blank slate, as if the previous one never happened.

For one-shot ticket processing, that's fine. But watch what happens the moment a support agent tries to actually *work* a ticket through the assistant:

> **Agent:** Look up order BC-48291 and tell me what's going on.
> **Assistant:** Order BC-48291 (Deluxe Espresso Machine) is marked DELIVERED but the customer says it never arrived. Looks like a delivery discrepancy.
> **Agent:** Okay, is that customer a repeat buyer?
> **Assistant:** Which customer? I don't have any order or customer in context.

And there it is. The assistant looked up the customer's order *one sentence ago* and has already forgotten it exists.

The follow-up question, the most natural thing in the world for a human, falls flat. The assistant has no memory of the turn that came before. This article fixes that. We're giving BrightCart the ability to hold a conversation: to remember what was said earlier in a ticket thread and carry that context forward.

And as article 7 promised, the way Spring AI provides memory is through an advisor. Having learned that mechanism last time, you're about to watch it power something genuinely useful.

Source, as always, in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth).

## LLMs Don't Remember Anything

Before we add memory, you need to understand a truth that trips up nearly everyone new to this. It's the opposite of how the tools *feel*.

> **Concept Primer: The Model Is Stateless**
> When you chat with an AI assistant and it "remembers" what you said three messages ago, the model itself is remembering nothing. A large language model is a pure function: text in, text out, no memory between calls. Each request is completely independent. The illusion of a continuous conversation is created entirely by the *application*, which keeps a record of everything said so far and re-sends the whole history with every new message. The model reads that history fresh each time, as if for the first time, and responds. "Memory" is not something the model has. It's something you feed it, over and over, on every single call.

Sit with that for a second, because it reframes everything. When ChatGPT appears to remember your name from earlier in a chat, what's actually happening is that your name is being re-transmitted, in full, with every message you send. The model has no persistent state. It's Groundhog Day on every call. The model wakes up, reads the entire conversation so far, answers, and forgets it all again.

This has two consequences that the rest of the article hangs on. First, giving BrightCart "memory" means building the part that stores conversation history and re-sends it, because the model won't do it for us. Second, because we re-send the entire history every time, a long conversation means a lot of re-sent text, which means a lot of tokens, which means real money. Hold that second thought. We'll come back to it, and it's the part that'll actually cost you if you ignore it.

Spring AI gives us the machinery to manage all of this cleanly. Let's meet it.

## The Two Halves of Memory

Spring AI splits chat memory into two distinct responsibilities, and keeping them straight makes everything else clearer.

**`ChatMemory`** is the *policy*. It decides which messages remain relevant for the next model call. Keep the last 10 messages? The last 50? Summarize the old ones? That's memory policy, and the default implementation, `MessageWindowChatMemory`, keeps a sliding window of the most recent messages.

**`ChatMemoryRepository`** is the *storage*. It decides where messages physically live. A `ConcurrentHashMap`? A relational database? Redis? That's persistence, a completely separate concern from policy.

The separation is deliberate, and it's the kind of clean seam I wish more libraries bothered with. You can pair any policy with any storage: a sliding-window policy backed by an in-memory map for tests, or that same window policy backed by a database for production. Change where memory is stored without touching how much of it you keep, and the other way around. This is the behavior we as developers have come to know from other Spring project, and for AI memory, this is nothing different.

By default, Spring AI auto-configures a `MessageWindowChatMemory` backed by an `InMemoryChatMemoryRepository`, which is a `ConcurrentHashMap` keyed by conversation ID. That's fine for experiments, but it evaporates on restart and doesn't survive across multiple application instances. Since BrightCart already has an H2 database from article 6, we'll go straight to persistent JDBC storage and skip the in-memory detour. Worth knowing the in-memory default is what you're replacing, though.

## Memory Is an Advisor

Here's where article 7 pays off. The component that actually injects conversation history into your requests is `MessageChatMemoryAdvisor`, an advisor sitting in the same chain you learned about last time.

Trace what it does on each call and you'll recognize every step:

1. On the way *in*, it reads the conversation ID from the request, retrieves that conversation's history from the `ChatMemory`, and prepends those prior messages to the current prompt. Modifying the request on the way down, exactly what article 7's sanitizing advisor did.
2. It calls the next advisor in the chain, and eventually the model, which now sees the full history plus the new message.
3. On the way *back*, it saves the new user message and the model's response into memory, so they're there for next time.

No new concepts. It's an advisor that reads memory on the way in and writes memory on the way out. Everything you learned about the advisor chain applies directly, including how it orders itself relative to the tool-calling advisor. More on that later.

Wiring it into a `ChatClient` looks like this:

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
        .maxMessages(20)
        .build();

ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();
```

That's the whole integration. From here on, every call through this client automatically carries the relevant conversation history, as long as you tell it *which* conversation. We'll get to that in a moment.

## Two Flavors of Memory Advisor

Spring AI actually ships two memory advisors, and they differ in *how* they hand the history to the model. The distinction is small but worth understanding, because it affects both behavior and token usage.

**`MessageChatMemoryAdvisor`** injects the prior turns as *structured messages*. The history is added to the prompt as a proper sequence of user and assistant messages, preserving each one's role. This is the faithful, high-fidelity option: the model sees the conversation exactly as it happened, with clear turn boundaries. For most applications, and for BrightCart, this is the right default.

**`PromptChatMemoryAdvisor`** takes a different approach. It flattens the prior turns into *plain text* and folds them into the system prompt using a template. The roles are collapsed into a text blob rather than preserved as distinct messages. This is occasionally useful when you want to compact or reformat the history, or when a model handles a single rich system prompt better than a long message list.

The practical guidance: reach for `MessageChatMemoryAdvisor` unless you have a specific reason not to. Preserving message roles gives the model the clearest possible picture of who said what, which matters for a support conversation where the distinction between the customer's words and the assistant's own prior conclusions is important. We'll use `MessageChatMemoryAdvisor` throughout.

## Conversation IDs: Which Conversation Are We In?

A memory advisor is useless without knowing *which* conversation to remember. If BrightCart is handling a thousand tickets, their histories can't all blur together. Each needs its own isolated thread. That's what the conversation ID does. It's the key that separates one conversation's memory from another's.

One important 2.0 detail: Spring AI no longer auto-assigns a conversation ID. Earlier versions quietly supplied a default. Now you must provide one explicitly, or memory won't work the way you expect. You set it as an advisor parameter per request:

```java
chatClient.prompt()
        .user(message)
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
        .call()
        .content();
```

The `ChatMemory.CONVERSATION_ID` parameter tells the memory advisor which thread this message belongs to. Which history to load, and where to save the new turn.

Now, where does that ID come from? Most tutorials hardcode something like `"007"` or `"DEMO"` and wave their hands. BrightCart has a far more natural answer sitting right in its domain: a ticket already *is* a conversation. A customer writes in, an agent replies, the customer responds. That thread of messages, accumulating over the life of the ticket, is exactly what a conversation ID is meant to key. So BrightCart's conversation ID is simply the ticket ID. No artificial construct, no hand-waving. The thing that identifies the ticket identifies its conversation.

## Persisting Memory to the Database

In-memory storage forgets everything on restart, which is unacceptable for a support system where a ticket thread might span days. Let's persist memory to the H2 database we set up back in article 6.

Add the JDBC chat memory starter:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```
Yup, you  can leave it up to the Spring team to come up with a name that long. The starter brings in the JDBC repository and its dependencies, including Spring Data JDBC and H2.

The starter also auto-configures a `JdbcChatMemoryRepository` and creates the backing table for you on startup, when schema initialization is enabled. For our H2 setup:

```properties
# Let Spring AI create the chat memory schema on startup
spring.ai.chat.memory.repository.jdbc.initialize-schema=always
```

Now define the memory as a bean, backed by JDBC storage but keeping the same sliding-window policy:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.memory.MessageWindowChatMemory;
import org.springframework.ai.chat.memory.repository.jdbc.JdbcChatMemoryRepository;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MemoryConfig {

    @Bean
    public ChatMemory ticketChatMemory(JdbcChatMemoryRepository repository) {
        return MessageWindowChatMemory.builder()
                .chatMemoryRepository(repository)
                .maxMessages(20)
                .build();
    }
}
```

Notice the shape. `MessageWindowChatMemory`, the policy keeping the last 20 messages, wraps `JdbcChatMemoryRepository`, the storage in H2. The two halves from earlier, composed. Swap the repository for Redis or Postgres later and this policy line never changes.

## The Cost of Remembering

Remember the thought I asked you to hold from the Concept Primer? Here it is, because it's one of the most important practical lessons in this whole series.

Because the model is stateless, every turn re-sends the entire conversation history. Turn one sends one message. Turn ten sends all ten previous messages plus the new one. Turn fifty sends forty-nine messages of backstory before the model even reads the new question. And you pay for every token, on every call. The history isn't sent once and cached. It's re-transmitted in full each time.

This means the cost of a conversation doesn't grow linearly with its length. It grows *quadratically*. A conversation twice as long doesn't cost twice as much. It costs closer to four times as much, because each of the additional turns also carries all the extra history. For a support system processing thousands of ticket threads, that curve is not academic. It's the difference between a manageable API bill and a genuinely nasty surprise at the end of the month.

This is exactly what `maxMessages` is for. By windowing memory to the last 20 messages, `MessageWindowChatMemory` caps how much history rides along on each call. Older messages fall out of the window, the conversation stays bounded, and the per-call cost plateaus instead of climbing forever. You trade some long-range memory, the assistant won't recall what was said fifty turns ago, for predictable cost. For most support tickets, the last 20 messages is far more than enough context.

Choosing the window size is a real engineering decision. Too small and the assistant forgets relevant context mid-ticket. Too large and costs balloon on long threads. Twenty is a sensible default for BrightCart. We'll come back to token cost as a first-class production concern in article 13. For now, just internalize that memory is not free, and unbounded memory is expensive in a way that sneaks up on you.

## When Memory Meets Tools

BrightCart's enrichment client has tools from article 6, and now it wants memory too. These two advisors have to coexist in the chain, and the way they're ordered matters. Remember article 7, where we gave attention to the ordering section?

Here's the subtlety. The tool-calling loop generates its own intermediate messages: the model's request to call a tool, and the tool's result. These aren't normal conversation turns. They're the internal back-and-forth of a single response being assembled. And most chat memory repositories can't store them. A `ToolResponseMessage` or a tool-call `AssistantMessage` doesn't fit the simple user/assistant shape that memory tables expect.

Spring AI 2.0 handles this by placing the memory advisor *outside* the tool loop by default. The precedence was deliberately set so that `MessageChatMemoryAdvisor` wraps the entire `ToolCallAdvisor` rather than running inside each iteration of it. The practical effect: memory records only the clean turns, the user's message and the model's final answer, while the messy intermediate tool negotiation stays internal to the tool advisor, which manages its own working history for the duration of the loop.

This is exactly the behavior you want. When an agent later scrolls back through a ticket thread, they see "agent asked about the order" and "assistant reported the delivery discrepancy," not a confusing transcript of internal tool-call plumbing. You get this for free from the default ordering. I'm pointing it out only so that when you inspect the stored memory and find the tool chatter absent, you know it's by design, not a bug.

## BrightCart Gets Ticket Threads

Time to assemble it. We'll add an endpoint where each message posted to a ticket becomes a turn in that ticket's conversation, with the ticket ID as the conversation ID and full history persisted in H2.

The `ChatClient`, with the memory advisor plus the order tools from article 6, so the assistant can both remember *and* look things up:

```java
package org.veenx.springai.demo.config;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.client.advisor.MessageChatMemoryAdvisor;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.ai.chat.prompt.ChatOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.veenx.springai.demo.order.OrderTools;

@Configuration
public class TicketChatConfig {

    @Bean
    public ChatClient ticketConversationClient(
            ChatClient.Builder builder,
            ChatMemory ticketChatMemory,
            OrderTools orderTools) {
        return builder
                .defaultSystem("""
                        You are a support assistant for BrightCart, an online retailer.
                        You are helping a support agent work through a ticket.
                        Use the available tools to look up real order data when relevant.
                        Refer back to earlier messages in the conversation as needed.
                        Be factual and concise.
                        """)
                .defaultOptions(ChatOptions.builder()
                        .temperature(0.2)
                        .maxTokens(512))
                .defaultAdvisors(MessageChatMemoryAdvisor.builder(ticketChatMemory).build())
                .defaultTools(orderTools)
                .build();
    }
}
```

The service, which sets the conversation ID from the ticket ID on every call:

```java
package org.veenx.springai.demo.service;

import org.springframework.ai.chat.client.ChatClient;
import org.springframework.ai.chat.memory.ChatMemory;
import org.springframework.stereotype.Service;

@Service
public class TicketConversationService {

    private final ChatClient ticketConversationClient;

    public TicketConversationService(ChatClient ticketConversationClient) {
        this.ticketConversationClient = ticketConversationClient;
    }

    public String reply(String ticketId, String agentMessage) {
        return ticketConversationClient
                .prompt()
                .user(agentMessage)
                .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, ticketId))
                .call()
                .content();
    }
}
```

That single `.advisors(a -> a.param(ChatMemory.CONVERSATION_ID, ticketId))` line is what ties each message to its ticket's thread. Same client, many tickets, each with cleanly isolated memory keyed by ticket ID.

The controller, with the ticket ID in the path:

```java
package org.veenx.springai.demo.web;

import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.veenx.springai.demo.service.TicketConversationService;

@RestController
@RequestMapping("/api/tickets")
public class TicketConversationController {

    private final TicketConversationService conversationService;

    public TicketConversationController(TicketConversationService conversationService) {
        this.conversationService = conversationService;
    }

    @PostMapping("/{ticketId}/messages")
    public String postMessage(
            @PathVariable String ticketId,
            @RequestBody String agentMessage) {
        return conversationService.reply(ticketId, agentMessage);
    }
}
```

Now let's replay the conversation from the top of the article, the one that fell apart, against the memory-enabled endpoint. First message on ticket `TICKET-1001`:

```bash
curl -X POST http://localhost:8080/api/tickets/TICKET-1001/messages \
     -H "Content-Type: text/plain" \
     -d "Look up order BC-48291 and tell me what's going on."
```

```
Order BC-48291 (Deluxe Espresso Machine) is marked DELIVERED with an estimated
delivery of 3 June, but this is flagged against a customer report of non-receipt.
It looks like a delivery discrepancy worth investigating.
```

Now the follow-up that failed before. Same ticket ID, so same conversation:

```bash
curl -X POST http://localhost:8080/api/tickets/TICKET-1001/messages \
     -H "Content-Type: text/plain" \
     -d "Okay, is that customer a repeat buyer?"
```

```
The order BC-48291 we just looked at is associated with rita@example.com. Let me
check her history... [tool lookup] ... she has three prior orders, all delivered
without issues, so this appears to be her first delivery problem.
```

That's the difference. "That customer" now resolves correctly, because the assistant remembers the order it looked up in the previous turn. The history was retrieved from H2, keyed by `TICKET-1001`, and prepended to the second request automatically.

Restart the application and the thread is still there. Post a message to a *different* ticket ID and it starts fresh, with no bleed-through. BrightCart's assistant can finally hold a conversation.

## What Actually Landed in the Database

Memory is the one part of this system with no visible output. Tool calls give you a log line. Classification hands back JSON. But memory happens quietly, between requests, and so far you've had to take my word for what's being stored. Let's stop doing that and just look.

We already have the H2 console from article 6. Enable it if you haven't:

```properties
spring.h2.console.enabled=true
```

Start the app, run the two-message `TICKET-1001` conversation from above, then open `http://localhost:8080/h2-console` and query the table Spring AI created for you:

```sql
SELECT conversation_id, type, content
FROM SPRING_AI_CHAT_MEMORY
WHERE conversation_id = 'TICKET-1001'
ORDER BY "timestamp";
```

You'll see something like this:

```
CONVERSATION_ID | TYPE      | CONTENT
----------------+-----------+---------------------------------------------
TICKET-1001     | USER      | Look up order BC-48291 and tell me what's...
TICKET-1001     | ASSISTANT | Order BC-48291 (Deluxe Espresso Machine)...
TICKET-1001     | USER      | Okay, is that customer a repeat buyer?
TICKET-1001     | ASSISTANT | The order BC-48291 we just looked at is...
```

Four rows. Two user turns, two assistant turns, all tagged with the ticket ID and ordered by time. That's the entire "memory" of the conversation, sitting in a plain relational table you can read with ordinary SQL. No magic, just rows.

A few things worth reading off that result.

The `type` column is the message role. It's how `MessageChatMemoryAdvisor` reconstructs the conversation as structured messages on the next call, preserving who said what. This is the structured-message storage we contrasted with `PromptChatMemoryAdvisor` earlier, made concrete.

The `conversation_id` is our ticket ID, exactly as we set it. Post a message to a different ticket and you'll get rows with a different ID, cleanly partitioned in the same table. That's how one table serves a thousand isolated ticket threads.

And now the part that pays off the promise from "When Memory Meets Tools." Look at what is *not* in those rows. The second turn triggered a customer-history tool lookup. Yet there's no row for the model asking to call the tool, and no row for the tool's response. Just the agent's question and the assistant's final answer.

That absence is deliberate, and it happens for two reinforcing reasons. The memory advisor sits outside the tool loop, so it only ever sees the clean turns. And even if it saw more, the JDBC repository explicitly drops assistant messages containing tool calls, along with their tool-response replies, when it saves. The internal negotiation of a single answer never reaches the table. What persists is the conversation as a human would recount it, which is exactly what you want an agent to see when they scroll back through the ticket weeks later.

One honest caveat, because it's a sharp edge worth knowing about. That "drops tool messages on save" behavior is fine for our read-only lookups, where the tool result is fully reflected in the assistant's answer anyway. But if you ever build a flow where the tool exchange itself carries information the final answer doesn't capture, be aware it won't survive in memory. You'd need a custom repository for that. We don't, so we won't, but file it away.

## What's Next

BrightCart's assistant now has a memory. It holds the thread of a ticket, remembers what was looked up and concluded, and persists that across restarts. All through an advisor doing exactly what article 7 said advisors do, keyed by a conversation ID that maps naturally onto BrightCart's own tickets.

But there's still a hole in its knowledge, and it's a big one. Ask the assistant "what's BrightCart's return policy for a damaged item?" and it has no idea. It knows the conversation, it knows the order database, but it knows nothing about BrightCart's actual *documents*. The return policies, the product manuals, the shipping terms. That knowledge lives in files and wikis the model has never seen.

Closing that gap is Retrieval-Augmented Generation, RAG for short, and it's where the next two articles go. In article 9 we build a full RAG pipeline: ingesting BrightCart's documents, embedding them, and retrieving the right passage to ground the assistant's answers in real company knowledge. The assistant remembers the conversation. Soon it will actually know the policies too.

[Next: Spring AI Series: 9-RAG End to End →]