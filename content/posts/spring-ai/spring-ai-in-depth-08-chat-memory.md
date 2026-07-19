---
title: "Spring AI Series: 8-Chat Memory"
date: 2026-07-24
draft: true
cover:
  image: "/images/spring-ai-08-chat-memory.png"
  alt: "Spring AI Series: Chat Memory and Conversation State"
series: ["Spring AI in Depth"]
series_order: 8
tags: ["Java", "Spring Boot", "AI", "Spring AI"]
categories: ["ai", "java"]
description: "LLMs remember nothing between calls. Learn how Spring AI's chat memory advisors create the illusion of conversation, and give BrightCart persistent per-ticket memory."
---

Everything BrightCart has built so far has a peculiar kind of amnesia. Each request lands, gets handled, and vanishes from the system's mind entirely. The summarizer, the classifier, the enrichment service — every single call starts from a blank slate, as if the previous one never happened.

For one-shot ticket processing, that's fine. But watch what happens the moment a support agent tries to actually *work* a ticket through the assistant:

> **Agent:** Look up order BC-48291 and tell me what's going on.
> **Assistant:** Order BC-48291 (Deluxe Espresso Machine) is marked DELIVERED but the customer says it never arrived. Looks like a delivery discrepancy.
> **Agent:** Okay, is that customer a repeat buyer?
> **Assistant:** Which customer? I don't have any order or customer in context.

And there it is. The assistant looked up the customer's order *one sentence ago* and has already forgotten it exists. The follow-up question — the most natural thing in the world for a human — falls flat, because the assistant has no memory of the turn that came before.

This article fixes that. We're giving BrightCart the ability to hold a conversation: to remember what was said earlier in a ticket thread and carry that context forward. And as article 7 promised, the way Spring AI provides memory is through an advisor — so having learned that mechanism, you're about to watch it power something genuinely useful.

Source, as always, in the [GitHub repository](https://github.com/RonVeen/spring-ai-in-depth).

## LLMs Don't Remember Anything

Before we add memory, you need to understand a truth that trips up nearly everyone new to this, because it's the opposite of how the tools *feel*.

> **Concept Primer — The Model Is Stateless**
> When you chat with an AI assistant and it "remembers" what you said three messages ago, the model itself is remembering nothing. A large language model is a pure function: text in, text out, no memory between calls. Each request is completely independent. The illusion of a continuous conversation is created entirely by the *application* — it keeps a record of everything said so far and re-sends the whole history with every new message. The model reads that history fresh each time, as if for the first time, and responds. "Memory," in other words, is not something the model has. It's something you feed it, over and over, on every single call.

Sit with that for a second, because it reframes everything. When ChatGPT appears to remember your name from earlier in a chat, what's actually happening is that your name is being re-transmitted, in full, with every message you send. The model has no persistent state. It's Groundhog Day on every call — the model wakes up, reads the entire conversation so far, answers, and forgets it all again.

This has two immediate consequences that the rest of the article hangs on. First, giving BrightCart "memory" means building the part that stores conversation history and re-sends it — the model won't do it for us. Second, because we re-send the entire history every time, a long conversation means a lot of re-sent text, which means a lot of tokens, which means real money. Hold that thought; we'll return to it.

Spring AI gives us the machinery to manage all of this cleanly. Let's meet it.

## The Two Halves of Memory

Spring AI splits chat memory into two distinct responsibilities, and keeping them straight makes everything else clearer.

**`ChatMemory`** is the *policy* — it decides which messages remain relevant for the next model call. Should we keep the last 10 messages? The last 50? Summarize old ones? That's memory policy, and the default implementation, `MessageWindowChatMemory`, keeps a sliding window of the most recent messages.

**`ChatMemoryRepository`** is the *storage* — it decides where messages physically live. In a `ConcurrentHashMap`? A relational database? Redis? That's persistence, and it's a completely separate concern from policy.

The separation is deliberate and useful. You can pair any policy with any storage: a sliding-window policy backed by an in-memory map for tests, or that same window policy backed by a JDBC database for production. Change where memory is stored without touching how much of it you keep, and vice versa.

By default, Spring AI auto-configures a `MessageWindowChatMemory` (the policy) backed by an `InMemoryChatMemoryRepository` (the storage) — a `ConcurrentHashMap` keyed by conversation ID. That's fine for experiments, but it evaporates on restart and doesn't survive across multiple application instances. Since BrightCart already has an H2 database from article 6, we'll go straight to persistent JDBC storage and skip the in-memory detour. But it's worth knowing the in-memory default is what you're replacing.

## Memory Is an Advisor

Here's where article 7 pays off. The component that actually injects conversation history into your requests is `MessageChatMemoryAdvisor` — an advisor, sitting in the same chain you learned about last time.

Trace what it does on each call, and you'll recognize every step:

1. On the way *in*, it reads the conversation ID from the request, retrieves that conversation's history from the `ChatMemory`, and prepends those prior messages to the current prompt. (Modifying the request on the way down — exactly what article 7's sanitizing advisor did.)
2. It calls the next advisor in the chain, and eventually the model, which now sees the full history plus the new message.
3. On the way *back*, it saves the new user message and the model's response into memory, so they're there for next time.

No new concepts. It's an advisor that reads memory on the way in and writes memory on the way out. Everything you learned about the advisor chain applies directly — including, as we'll see shortly, how it orders itself relative to the tool-calling advisor.

Wiring it into a `ChatClient` looks like this:

```java
ChatMemory chatMemory = MessageWindowChatMemory.builder()
        .maxMessages(20)
        .build();

ChatClient chatClient = ChatClient.builder(chatModel)
        .defaultAdvisors(MessageChatMemoryAdvisor.builder(chatMemory).build())
        .build();
```

That's the whole integration. From here on, every call through this client automatically carries the relevant conversation history — as long as you tell it *which* conversation. We'll get to that in a moment.

## Two Flavors of Memory Advisor

Spring AI actually ships two memory advisors, and they differ in *how* they hand the history to the model. The distinction is small but worth understanding, because it affects both behavior and token usage.

**`MessageChatMemoryAdvisor`** injects the prior turns as *structured messages* — the history is added to the prompt as a proper sequence of user and assistant messages, preserving each one's role. This is the faithful, high-fidelity option: the model sees the conversation exactly as it happened, with clear turn boundaries. For most applications, and for BrightCart, this is the right default.

**`PromptChatMemoryAdvisor`** takes a different approach — it flattens the prior turns into *plain text* and folds them into the system prompt, using a template. The roles are collapsed into a text blob rather than preserved as distinct messages. This is occasionally useful when you want to compact or reformat the history, or when a model handles a single rich system prompt better than a long message list.

The practical guidance: reach for `MessageChatMemoryAdvisor` unless you have a specific reason not to. Preserving message roles gives the model the clearest possible picture of who said what, which matters for a support conversation where the distinction between the customer's words and the assistant's own prior conclusions is important. We'll use `MessageChatMemoryAdvisor` throughout.

## Conversation IDs: Which Conversation Are We In?

A memory advisor is useless without knowing *which* conversation to remember. If BrightCart is handling a thousand tickets, their histories can't all blur together — each needs its own isolated thread. That's what the conversation ID does: it's the key that separates one conversation's memory from another's.

One important 2.0 detail: **Spring AI no longer auto-assigns a conversation ID.** Earlier versions quietly supplied a default; now you must provide one explicitly, or memory won't work as you expect. You set it as an advisor parameter per request:

```java
chatClient.prompt()
        .user(message)
        .advisors(a -> a.param(ChatMemory.CONVERSATION_ID, conversationId))
        .call()
        .content();
```

The `ChatMemory.CONVERSATION_ID` parameter tells the memory advisor which thread this message belongs to — which history to load, and where to save the new turn.

Now, where does that ID come from? Most tutorials hardcode something like `"007"` or `"DEMO"` and wave their hands. BrightCart has a far more natural answer sitting right in its domain: **a ticket already is a conversation.** A customer writes in, an agent replies, the customer responds — that thread of messages, accumulating over the life of the ticket, is exactly what a conversation ID is meant to key. So BrightCart's conversation ID is simply the ticket ID. No artificial construct, no hand-waving. The thing that identifies the ticket identifies its conversation.

## Persisting Memory to the Database

In-memory storage forgets everything on restart, which is unacceptable for a support system where a ticket thread might span days. Let's persist memory to the H2 database we set up back in article 6.

Add the JDBC chat memory starter:

```xml
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-chat-memory-repository-jdbc</artifactId>
</dependency>
```

The starter auto-configures a `JdbcChatMemoryRepository` and, in a schema-init-enabled setup, creates the backing table for you automatically. You can control schema initialization through properties; for our H2 setup the default behavior creates the table on startup:

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

Notice the shape: `MessageWindowChatMemory` (the policy, keeping the last 20 messages) wraps `JdbcChatMemoryRepository` (the storage, in H2). The two halves from earlier, composed. Swap the repository for Redis or Postgres later and this policy line never changes.

## The Cost of Remembering

Remember the thought I asked you to hold from the Concept Primer? Here it is, because it's one of the most important practical lessons in this whole series.

Because the model is stateless, *every turn re-sends the entire conversation history.* Turn one sends one message. Turn ten sends all ten previous messages plus the new one. Turn fifty sends forty-nine messages of backstory before the model even reads the new question. And you pay for every token, on every call — the history isn't sent once and cached, it's re-transmitted in full each time.

This means the cost of a conversation doesn't grow linearly with its length — it grows *quadratically*. A conversation twice as long doesn't cost twice as much; it costs closer to four times as much, because each of the additional turns also carries all the extra history. For a support system processing thousands of ticket threads, that curve is not academic. It's the difference between a manageable API bill and a nasty surprise.

This is exactly what `maxMessages` is for. By windowing memory to the last 20 messages, `MessageWindowChatMemory` caps how much history rides along on each call. Older messages fall out of the window; the conversation stays bounded; the per-call cost plateaus instead of climbing forever. You trade some long-range memory (the assistant won't recall what was said fifty turns ago) for predictable cost — and for most support tickets, the last 20 messages is far more than enough context.

Choosing the window size is a real engineering decision: too small and the assistant forgets relevant context mid-ticket; too large and costs balloon on long threads. Twenty is a sensible default for BrightCart. We'll come back to token cost as a first-class production concern in article 13 — for now, just internalize that memory is not free, and unbounded memory is expensive in a way that sneaks up on you.

## When Memory Meets Tools

BrightCart's enrichment client has tools (article 6) *and* now wants memory. These two advisors have to coexist in the chain, and the way they're ordered matters — which rewards your attention to article 7's ordering section.

Here's the subtlety. The tool-calling loop generates its own intermediate messages: the model's request to call a tool, and the tool's result. These aren't normal conversation turns — they're the internal back-and-forth of a single response being assembled. And most chat memory repositories can't store them: a `ToolResponseMessage` or a tool-call `AssistantMessage` doesn't fit the simple user/assistant shape that memory tables expect.

Spring AI 2.0 handles this by placing the memory advisor *outside* the tool loop by default. The precedence was deliberately set so that `MessageChatMemoryAdvisor` wraps the entire `ToolCallAdvisor` rather than running inside each iteration of it. The practical effect: memory records only the clean turns — the user's message and the model's final answer — while the messy intermediate tool negotiation stays internal to the tool advisor, which manages its own working history for the duration of the loop.

This is exactly the behavior you want. When an agent later scrolls back through a ticket thread, they see "agent asked about the order" and "assistant reported the delivery discrepancy" — not a confusing transcript of internal tool-call plumbing. You get this for free from the default ordering; I'm spelling it out only so that when you inspect the stored memory and find the tool chatter absent, you know it's by design, not a bug.

## BrightCart Gets Ticket Threads

Time to assemble it. We'll add an endpoint where each message posted to a ticket becomes a turn in that ticket's conversation, with the ticket ID as the conversation ID and full history persisted in H2.

The `ChatClient` — memory advisor plus the order tools from article 6, so the assistant can both remember *and* look things up:

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

Now let's replay the conversation from the top of the article — the one that fell apart — against the memory-enabled endpoint. First message on ticket `TICKET-1001`:

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

Now the follow-up that failed before — same ticket ID, so same conversation:

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

*That's* the difference. "That customer" now resolves correctly, because the assistant remembers the order it looked up in the previous turn — the history was retrieved from H2, keyed by `TICKET-1001`, and prepended to the second request automatically. Restart the application and the thread is still there; post a message to a *different* ticket ID and it starts fresh, with no bleed-through. BrightCart's assistant can finally hold a conversation.

## What's Next

BrightCart's assistant now has a memory. It holds the thread of a ticket, remembers what was looked up and concluded, and persists that across restarts — all through an advisor doing exactly what article 7 said advisors do, keyed by a conversation ID that maps naturally onto BrightCart's own tickets.

But there's still a hole in its knowledge, and it's a big one. Ask the assistant "what's BrightCart's return policy for a damaged item?" and it has no idea. It knows the conversation, it knows the order database, but it knows nothing about BrightCart's actual *documents* — the return policies, the product manuals, the shipping terms. That knowledge lives in files and wikis the model has never seen.

Closing that gap is Retrieval-Augmented Generation, and it's where the next two articles go. In article 9 we build a full RAG pipeline — ingesting BrightCart's documents, embedding them, and retrieving the right passage to ground the assistant's answers in real company knowledge. The assistant remembers the conversation; soon it will actually know the policies too.

*This is part 8 of a 13-part series.*

[← Previous: Spring AI Series: 7-Advisors: Spring AI's Middleware Layer]
[Next: Spring AI Series: 9-RAG End to End →]