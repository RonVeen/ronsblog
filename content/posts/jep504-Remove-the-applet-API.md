---
title: "Java 26 - part 2: JEP 504 Remove the Applet API"
date: 2026-01-17
draft: false
tags: ["Java", "JEP", "Java 26", "Applet"]
---

![Final](/images/jep504-remove-the-applet-API.png)


***

**Goodbye, Applets! (For Real This Time)**

If you've been doing Java for a while, you probably remember the "good old days" of running Java in the browser. One day, I was visiting London, from Amsterdam, and I bought my first Java book. This was in de the days before you buy anything at Amazon. The book introduced me to applets for the first time. Must have been around the time of Java 1.2, so we're talking about 1998. Fast forward to a few years later and I had a girlfriend who was studying in IT. One of her assigments was to create a functioning chess application, running in a browser via an applet. Did we have fun writing that applet. Or least, I had fun.
Well, those days of Applet-Chess are over, JEP 504 in Java 26 is finally cleaning house by permanently removing the Applet API.

This shouldn't come as a shock to anyone—the API has been "dead code walking" for years. It was deprecated back in JDK 9 (2017), the `appletviewer` tool was nuked in JDK 11, and it was explicitly marked for removal in JDK 17,. Plus, with the Security Manager being disabled in JDK 24, the sandbox mechanisms required to run applets safely are gone anyway. Since web browsers stopped supporting applets years ago, keeping this API around was essentially just hoarding obsolete code.

**What’s actually being deleted?**
The cleanup is thorough. The entire `java.applet` package is gone, which includes classes like `Applet`, `AppletContext`, `AppletStub`, and `AudioClip`. Additionally, `javax.swing.JApplet` and `java.beans.AppletInitializer` have been removed.

**Will this break anything?**
For 99% of developers, absolutely not. However, there is one small edge case to watch out for: if you have legacy code that calls `URL.getContent()` or `URLConnection.getContent()` and casts the result to an `AudioClip`, that code will no longer compile or run, as those methods can no longer return an `AudioClip` instance.

In short: It’s time to pour one out for the technology that introduced many of us to Java, but it's definitely time to let it go.