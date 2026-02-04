---
title: "JEP 500: Prepare Final Mean Final"
date: 2025-12-29
draft: false
tags: ["Java", "JEP", "Java 26"]
---

![Final](/images/jep500-Finals.jpg)

When you have been writing Java for a while, you may have noticed that you are writing `final` a lot.
You typically use it when defining class or instance variables. In those situations the compiler will ensure that need to assign a value before the class constructor completes.
Or you could use it with parameters, to indicate that the values of these parameters should not be tinkered with.

It is Java's version of defining a constant. Or at least, that is what we have been thinking.
But in real life, a final variable can be changed.
Take a look at the following example:

```java
import java.lang.reflect.Field;

public class MutableFinal {
    public static void main(String[] args) throws Exception {
        DataHolder holder = new DataHolder();
        
        Field field = DataHolder.class.getDeclaredField("message");
        field.setAccessible(true);
        
        field.set(holder, "Updated via Reflection");

        System.out.println(holder.getMessage());
    }
}

class DataHolder {
private final String message = new String("Original Message");

    public String getMessage() { return message; }
}
```

Now, this only works for a number of reasons. The most important is that we use `new String("Original Message")`. If not, the compiler would have replaced every mention of `message` with `"Original Message"`.
But by creating it at runtime, the JVM will look at the memory address of the `DataHolder` instance and there we have just changed its value.
There are other ways to achieve this, for instance, by using the Unsafe API.

### Why This Works (Historically)
- setAccessible(true) disables Java language access checks
- The JVM historically allowed reflective writes to final fields
- No bytecode verification prevented this mutation

### Why This Is a Real Problem

At first glance, reflective mutation of final fields feels like an edge case. Something only frameworks or “clever hacks” rely on.
But the consequences are bigger than that.

#### It breaks reasoning
If a final field can change at runtime, code becomes harder to understand and debug—especially in concurrent systems.

#### It weakens the Java Memory Model
Final fields get special visibility guarantees. Mutating them after construction undermines those guarantees in subtle and dangerous ways.

#### It limits JVM optimizations
The JVM wants to trust that final values stay constant. If it cannot, it must be conservative, which means fewer optimizations and worse performance.

Simply put: the platform has been paying a high price for this loophole.

## Enter JEP 500
With JEP-500, final finally(!) becomes fina.
But it will not suddenly break your application.
Instead, it introduces something Java has been missing for a long time: honesty.

Starting with JDK 26, when code tries to mutate a final field via reflection:
- The mutation still happens
- But the JVM emits a warning (once per module)
That is it. No failures. No forced rewrites. Just visibility.

This approach gives teams time to understand what their code—and their dependencies—are really doing.

### Making Unsafe Behavior a Choice
JEP 500 also introduces new JVM options that put developers in control.

To enable final field mutation by any code on the classpath, use:
```--enable-final-field-mutation=ALL-UNNAMED```

To enable final field mutation by specific modules on the module path, pass a comma-seperated list of modules:
```--enable-final-field-mutation=Mod1,Mod2,Mod3```

Now, if an illegal final mutation is discovered, meaning some code that is not on the exempt list of the above command-line options tries to modify a final field, the action that the JVM takes differs, depending on the setting of another command-line option:

```--enable-final-field-mutation=```
- warn (default)
“Tell me when this happens.”

- deny
“Fail fast. This should never happen in my code.”

- allow
“I know this is unsafe, but I accept it.”

The key shift here is philosophical: unsafe behavior is no longer silent. **If you rely on it, you must opt in**.

### Legacy Code Is Not Ignored
Java knows the real world is messy.
Some libraries still rely on reflective final field mutation—for deserialization, proxying, or legacy injection patterns. JEP 500 allows teams to explicitly enable this behavior for:
- The classpath
- Specific modules

This makes technical debt explicit and localized, instead of invisible and global.

### Finally, Observability
One of the most practical improvements in JEP 500 is its integration with Java Flight Recorder.
Each time a final field is mutated reflectively, a JFR event can be recorded. That means you can:
- Identify which library is responsible
- See where it happens
- Measure how often it occurs

For the first time, this behavior is no longer a black box.

### What This Means for Frameworks
Framework authors will feel this change first.
Libraries that rely on mutating final fields after construction will need to adapt. In most cases, better alternatives already exist:
- Constructor injection instead of field injection
- Code generation instead of reflection
- Explicit APIs instead of “magic” initialization

The result will likely be simpler, more predictable frameworks—and fewer surprises for users.

### What You Should Do as a Developer
You do not need to panic. But you should prepare.
A sensible approach:
- Run your application on JDK 26
- Watch for warnings
- Enable stricter modes in tests
- Track updates from your dependencies
- Avoid writing new code that mutates final fields reflectively

These steps are small, but they future-proof your codebase.



