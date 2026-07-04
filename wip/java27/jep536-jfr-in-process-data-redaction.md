---
title: "JEP 536 - JFR In-Process Data Redaction"
description: "Learn how JDK Flight Recorder now redacts secrets from command-line arguments, environment variables, and system properties before they ever hit disk."
date: 2026-07-06    
draft: false
tags: ["Java", "Security", "JFR"]
cover:
  image: "/images/jep436-jfr-in-process-data-redaction.png"
  alt: "JEP 536 - JFR In-Process Data Redaction"
series: ["Java 27"]
series_order: 7
categories: ["java"]
---

A few years back, someone on my team attached a `.jfr` file to a vendor support ticket. Nothing unusual — Flight Recorder is the go-to tool when you need to show "here's what the JVM was actually doing." Except that recording also happily included the database password we'd passed in as a `-D` system property on startup.

Nobody noticed until the ticket had already left the building.

That's the problem JEP 536 fixes. It's not glamorous. It won't make a conference keynote slide. But if you've ever shipped a JFR recording to a vendor, uploaded one to a shared bucket, or attached one to a bug report, you've probably leaked something you didn't mean to.

## What JFR Was Quietly Recording

JDK Flight Recorder captures a lot of startup context so you can reconstruct *how* a process was configured after the fact. Three events in particular are the culprits here:

- `jdk.InitialEnvironmentVariable` — every environment variable present at JVM startup
- `jdk.InitialSystemProperty` — every system property, including anything passed via `-D`
- `jdk.JVMInformation` — the full command line, arguments and all

That's fantastic for debugging. It's also exactly where people put `--dbpassword`, `-Djavax.net.ssl.keyStorePassword=`, or an `ACCESS_TOKEN` environment variable. None of that was ever meant to be readable by whoever opens the recording next.

Here's what that recording looks like today, without redaction:

```
$ jfr print --events InitialSystemProperty,InitialEnvironmentVariable dump.jfr

jdk.InitialSystemProperty {
  key   = "javax.net.ssl.keyStorePassword"
  value = "Sup3rSecret!"
}
jdk.InitialEnvironmentVariable {
  key   = "ACCESS_TOKEN"
  value = "ghp_xxxxxxxxxxxxxxxxxxxx"
}
```

That's the whole secret, sitting in plain text, in a file that's designed to be shared around for diagnostics.

## The Fix: Redact Before It Leaves the Process

JEP 536 changes the default behavior of JFR so that sensitive values are stripped out *before* they're written to the recording — not scrubbed afterward. That distinction matters. Post-processing tools like `jfr scrub` exist today, but they only clean up a file that already contains the secret. Until scrubbing runs, the unredacted file is sitting there, and if you forget to scrub — or scrub the wrong copy — the secret's already out.

With JEP 536, there's nothing to forget. The redaction happens in-process, as the event is captured.

As of JDK 27, this is **on by default**. No flags needed. Start a recording the way you always have, and JFR now redacts using a built-in filter list.

```
$ jfr print --events InitialSystemProperty,InitialEnvironmentVariable dump.jfr

jdk.InitialSystemProperty {
  key   = "javax.net.ssl.keyStorePassword"
  value = "[REDACTED]"
}
jdk.InitialEnvironmentVariable {
  key   = "ACCESS_TOKEN"
  value = "[REDACTED]"
}
```

Same recording, same diagnostic value for everything that *isn't* sensitive — your `-Xmx2G` heap setting is still right there — but the secret is gone.

## How JFR Decides What's Sensitive

There are two new sub-options on the existing `-XX:FlightRecorderOptions` flag: `redact-key` and `redact-argument`. Each takes a filter list — plain glob patterns with `*` and `?` wildcards.

`redact-key` matches against system property and environment variable *names*. The default filter list catches the usual suspects: `*password*`, `*passwd*`, `*pwd*`, `*secret*`, `*token*`, `*credential*`, `*private*key*`, `*api*key*`, `*client*secret*`, `*jwt*`, `*jaas*config*`, `*passphrase*`, and `*auth*`.

`redact-argument` matches against command-line arguments and uses almost the same list — with one deliberate omission. It drops `*auth*`, because on the command line that pattern would also match perfectly innocent things like `--author`. A minor detail, but it's the kind of detail that tells you someone actually thought about false positives instead of just shipping the broadest possible filter.

If you want to add your own filters on top of the defaults — say, your organization uses `confidential` as a convention — prefix the value with `+`:

```
$ java -XX:FlightRecorderOptions:'redact-key=+confidential' -jar app.jar
```

And if you'd rather load a longer list from a file instead of cramming it onto the command line, both options accept an `@filename` reference:

```
$ java '-XX:FlightRecorderOptions:redact-argument=@args.txt,redact-key=@keys.txt' -jar app.jar
```

That's the detail I actually like most here. Redaction rules become something you can version-control and reuse across every service in your fleet, instead of a command line nobody wants to touch.

## Opting Out

Sometimes you genuinely want the old behavior — maybe you're debugging locally and you *want* to see the real value. You can turn redaction off entirely:

```
$ java -XX:FlightRecorderOptions:'redact-argument=none,redact-key=none' -jar app.jar
```

Worth saying plainly: recordings from JDK 27 onward will differ from what you're used to, since redacted fields now show up as `[REDACTED]` instead of the real value. If a script or dashboard parses JFR output and expects an actual password there for some reason — first, please reconsider that setup — this is the flag that gets you back to the old output.

## Where This Actually Matters

This one's easy to underestimate if you've never been on the receiving end of a leak. But think about how many places a `.jfr` file travels once it's created:

- Attached to a support ticket for a vendor
- Uploaded to a shared drive for a colleague to look at
- Archived alongside a heap dump after an incident
- Picked up automatically by continuous profiling tooling

Every one of those is a place a secret can end up somewhere it shouldn't. JEP 536 doesn't fix all of them — heap dumps still need their own care, for instance — but it closes off the specific path where command-line arguments, environment variables, and system properties leak through JFR by default.

## Wrapping Up

JEP 536 is targeted for JDK 27, and it's the kind of feature that earns its keep by never showing up in a postmortem. You won't notice it working. You'll only notice the day it *would* have saved you from shipping a password to a support portal.

If you're running JDK 27 and haven't set any `redact-key` or `redact-argument` options, you're already covered by the defaults — that's the whole point. Worth double-checking your own naming conventions against that default list, though. If your team calls its secrets `envelope` or `sealed`, JFR has no way of knowing that.

*This is part 4 of the Java 27 series.*