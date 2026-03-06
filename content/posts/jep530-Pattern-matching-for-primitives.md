---
title: "Java 26 - part 10: JEP 530 - Pattern matching for primitives"
date: 2026-03-06
draft: false
tags: ["Java", "JEP", "Java 26", "Pattern Matching"]
cover:
  image: "/images/jep530-primitive-pattern-matching.png"
  alt: "Pattern Matching for primitives"
series: ["Java 26"]
---

**Java Pattern Matching: The Missing Piece (JEP 530)**
If you’ve been following Java for the last few years, you know the language has been on a massive "glow-up" journey. 
One of the stars of the show has been Pattern Matching. 
It started as a way to stop writing those annoying casts after an instanceof, and it’s grown into a powerful tool for data-driven code.

But there was always one weird restriction: it only worked for objects. 
If you wanted to use these cool new tricks with int, double, or boolean, you were out of luck.

Enter JEP 530. It’s the "Fourth Preview" of a feature that finally treats primitive types like first-class citizens in the world of patterns.

The Backstory: How We Got Here
To appreciate JEP 530, you have to look at the "Pattern Matching Trilogy" that came before it:

Level 1: instanceof (Java 16) – We stopped doing the if (obj instanceof String) { String s = (String)obj; } dance. We got if (obj instanceof String s).

Level 2: switch (Java 21) – We got the ability to switch on types and use "guards" (the when clause), making complex logic look like a clean list of options.

Level 3: Record Patterns (Java 21) – We learned how to "destructure" records. Instead of taking a Point and calling p.x() and p.y(), we could just say case Point(int x, int y).

So, What Does JEP 530 Actually Add?
Even with all that power, primitives were still the "ugly stepchildren." 
You couldn't use `instanceof int` or use an int pattern in a switch if the input was a long.

JEP 530 fixes this by allowing primitive types in all pattern contexts. 
Here’s what that looks like in the real world:

1. Safe Casting with instanceof
   Ever worried that casting a long to an int would silently cut off half your data? Now you can ask Java to check it for you:
```
if (myLong instanceof int i) {
// This ONLY runs if the long actually fits in an int without losing data!
System.out.println("It's a small number: " + i);
}
```

2. Primitives in switch
   You can now use any primitive type in a switch. This is huge for handling data that might come in different "sizes":

```
return switch (status) {
case byte b -> "It's a tiny status: " + b;
case int i  -> "It's a standard status: " + i;
case long l -> "It's a huge status: " + l;
};
```

3. Deep Record Destructuring
   Before this, if a Record had a double, you had to match it exactly as a double. 
   Now, if that double happens to be a whole number (like 10.0), you can match it directly as an int inside the record pattern. 
   It’s like having a built-in filter.

Why Does This Matter? (The "Informal" Why)
It’s all about Uniformity.
For years, Java has felt like two different languages: one for Objects and one for Primitives. 
Project Amber (the project behind these JEPs) is trying to bridge that gap. 
JEP 530 makes the code you write for an Integer work exactly the same way for an int.

It makes your code:
- Safer: No more "oops, I truncated my data" bugs.
- Cleaner: Fewer manual checks and range comparisons.
- Smarter: The compiler now understands "exactness"—it won't let a match happen if data would be lost.

The Bottom Line
JEP 530 is essentially the "Cleanup Crew." It takes all the cool pattern-matching features we’ve grown to love and makes sure they work everywhere, for every type of data. It’s the final polish on a feature that makes modern Java feel less like a "ceremony" and more like a tool that actually has your back.