# Factory Method Pattern — Java Reference

> **Purpose of this file:** Java has inheritance and abstract classes, which map directly to the classical GoF Factory Method structure. Use this alongside `factory-method-go.md` to see how the pattern looks when the language supports the original formulation.

## Table of Contents

- [Classical GoF Structure in Java](#classical-gof-structure-in-java)
- [The Four Roles](#the-four-roles)
- [Worked Example — Notification System](#worked-example--notification-system)
- [Adding a New Variant](#adding-a-new-variant)
- [Parameterised Factory Method](#parameterised-factory-method)
- [Why Go Looks Different](#why-go-looks-different)
- [References](#references)

---

## Classical GoF Structure in Java

The GoF definition: *"Define an interface for creating an object, but let subclasses decide which class to instantiate. Factory Method lets a class defer instantiation to subclasses."*

The key word is **subclasses**. The classical formulation relies on inheritance — an abstract creator class declares an abstract factory method, and each concrete subclass overrides it to return a specific product type. This is exactly what abstract classes and method overriding in Java were designed for.

```
          <<abstract>>
           Creator
        ─────────────
        + someOperation()         ← uses the product, calls createNotifier()
        # createNotifier(): Notifier  ← abstract factory method

               △
               │  extends
    ┌──────────┴──────────┐
    │                     │
EmailCreator          SlackCreator
─────────────         ─────────────
# createNotifier()    # createNotifier()
  → EmailNotifier       → SlackNotifier


          <<interface>>
            Notifier
          ──────────
          + send(msg)

               △
               │  implements
    ┌──────────┴──────────┐
    │                     │
EmailNotifier         SlackNotifier
```

The creator's `someOperation()` method calls `createNotifier()` internally. It does not know — and does not care — which concrete `Notifier` it gets back. The subclass decides.

---

## The Four Roles

**Product (`Notifier` interface)**
The interface that all concrete products implement. The creator and all callers depend only on this.

**Concrete Product (`EmailNotifier`, `SlackNotifier`)**
The actual objects being created. Each implements `Notifier`.

**Creator (`NotificationService` abstract class)**
Declares the abstract factory method `createNotifier()`. Also contains business logic that uses the product — this is the key point: the creator has non-trivial behaviour, not just creation.

**Concrete Creator (`EmailNotificationService`, `SlackNotificationService`)**
Extends the abstract creator and overrides `createNotifier()` to return the appropriate concrete product.

---

## Worked Example — Notification System

### Product Interface

```java
// Product — the only type callers and the creator depend on
public interface Notifier {
    void send(String message);
}
```

### Concrete Products

```java
// Concrete Product A
public class EmailNotifier implements Notifier {
    private final String smtpHost;
    private final String from;

    public EmailNotifier(String smtpHost, String from) {
        this.smtpHost = smtpHost;
        this.from = from;
    }

    @Override
    public void send(String message) {
        System.out.printf("[Email via %s from %s] %s%n", smtpHost, from, message);
    }
}

// Concrete Product B
public class SlackNotifier implements Notifier {
    private final String webhookUrl;

    public SlackNotifier(String webhookUrl) {
        this.webhookUrl = webhookUrl;
    }

    @Override
    public void send(String message) {
        System.out.printf("[Slack -> %s] %s%n", webhookUrl, message);
    }
}

// Concrete Product C
public class SMSNotifier implements Notifier {
    private final String apiKey;

    public SMSNotifier(String apiKey) {
        this.apiKey = apiKey;
    }

    @Override
    public void send(String message) {
        System.out.printf("[SMS] %s%n", message);
    }
}
```

### Abstract Creator

```java
// Creator — abstract class with the factory method + business logic
public abstract class NotificationService {

    // The factory method — abstract, no implementation here
    // Each subclass overrides this to return the right product
    protected abstract Notifier createNotifier();

    // Business logic that USES the product
    // This method never knows which concrete Notifier it gets
    public void notifyUser(String message) {
        Notifier notifier = createNotifier();  // delegation to subclass
        System.out.println("Preparing notification...");
        notifier.send(message);
        System.out.println("Notification sent.");
    }
}
```

The critical point: `notifyUser()` lives in the abstract creator. It calls `createNotifier()` — which it does not implement — and uses the result. The subclass plugs in the product; the parent provides the surrounding logic. This is the **Template Method** collaboration baked into Factory Method.

### Concrete Creators

```java
// Concrete Creator A — decides to create an EmailNotifier
public class EmailNotificationService extends NotificationService {

    private final String smtpHost;
    private final String from;

    public EmailNotificationService(String smtpHost, String from) {
        this.smtpHost = smtpHost;
        this.from = from;
    }

    @Override
    protected Notifier createNotifier() {
        return new EmailNotifier(smtpHost, from);
    }
}

// Concrete Creator B
public class SlackNotificationService extends NotificationService {

    private final String webhookUrl;

    public SlackNotificationService(String webhookUrl) {
        this.webhookUrl = webhookUrl;
    }

    @Override
    protected Notifier createNotifier() {
        return new SlackNotifier(webhookUrl);
    }
}

// Concrete Creator C
public class SMSNotificationService extends NotificationService {

    private final String apiKey;

    public SMSNotificationService(String apiKey) {
        this.apiKey = apiKey;
    }

    @Override
    protected Notifier createNotifier() {
        return new SMSNotifier(apiKey);
    }
}
```

### Caller / Client

```java
public class Main {
    public static void main(String[] args) {
        // The caller works with NotificationService — the abstract creator
        // It never references EmailNotifier, SlackNotifier, or SMSNotifier directly
        NotificationService service = new EmailNotificationService("smtp.example.com", "noreply@example.com");
        service.notifyUser("Your order has shipped.");

        // Swap to Slack — caller code structure is identical
        NotificationService slackService = new SlackNotificationService("https://hooks.slack.com/...");
        slackService.notifyUser("Deployment complete.");
    }
}
```

Output:
```
Preparing notification...
[Email via smtp.example.com from noreply@example.com] Your order has shipped.
Notification sent.

Preparing notification...
[Slack -> https://hooks.slack.com/...] Deployment complete.
Notification sent.
```

---

## Adding a New Variant

To add a Push notification channel, you create two new classes — one product, one creator — and change nothing else:

```java
// New Concrete Product
public class PushNotifier implements Notifier {
    private final String deviceToken;

    public PushNotifier(String deviceToken) {
        this.deviceToken = deviceToken;
    }

    @Override
    public void send(String message) {
        System.out.printf("[Push -> %s] %s%n", deviceToken, message);
    }
}

// New Concrete Creator
public class PushNotificationService extends NotificationService {

    private final String deviceToken;

    public PushNotificationService(String deviceToken) {
        this.deviceToken = deviceToken;
    }

    @Override
    protected Notifier createNotifier() {
        return new PushNotifier(deviceToken);
    }
}
```

`NotificationService`, `EmailNotificationService`, `SlackNotificationService`, and all callers are untouched. This is the Open/Closed Principle in action — open for extension, closed for modification.

---

## Parameterised Factory Method

An alternative formulation: instead of one concrete creator per product type, a single concrete creator uses a parameter to decide what to create. This is closer to what Go's constructor function pattern does.

```java
public class NotificationServiceFactory extends NotificationService {

    private final String channel;
    private final Map<String, String> config;

    public NotificationServiceFactory(String channel, Map<String, String> config) {
        this.channel = channel;
        this.config = config;
    }

    @Override
    protected Notifier createNotifier() {
        return switch (channel) {
            case "email" -> new EmailNotifier(config.get("smtpHost"), config.get("from"));
            case "slack" -> new SlackNotifier(config.get("webhookUrl"));
            case "sms"   -> new SMSNotifier(config.get("apiKey"));
            default      -> throw new IllegalArgumentException("Unknown channel: " + channel);
        };
    }
}
```

Trade-off: simpler class hierarchy (one creator, not N), but adding a new variant requires editing `createNotifier()` — violating Open/Closed. The subclass-per-variant approach avoids this at the cost of more classes.

In practice, Java codebases often use the parameterised form for simple cases and the subclass form when creators carry meaningfully different behaviour beyond just construction.

---

## Why Go Looks Different

| Concept | Java | Go |
|---|---|---|
| Abstract creator | `abstract class` with `abstract` factory method | Not available — no abstract classes |
| Concrete creator | `extends` abstract creator, overrides method | Not needed — constructor function replaces it |
| Subclass decision | Override `createNotifier()` per subclass | `switch` in `NewNotifier(kind string)` |
| Open/Closed extension | New subclass, no edits to existing code | Registry map + `init()` registration |
| Product type | `interface Notifier` | `type Notifier interface` |
| Caller dependency | `NotificationService` (abstract creator) | `Notifier` (product interface directly) |

The essence is identical: **callers depend on an interface; the creation decision is centralised and hidden from callers.** Java expresses this through inheritance and method overriding. Go expresses it through constructor functions and interface satisfaction. The pattern is the same; the mechanical expression differs because the languages have different capabilities.

The Go registry + `init()` approach is actually closer to the Open/Closed spirit of the classical pattern than the Java parameterised switch is — worth noting if this comparison comes up in an interview.

---

## References

- [Refactoring Guru — Factory Method](https://refactoring.guru/design-patterns/factory-method)
- [GoF Design Patterns — Factory Method (Wikipedia)](https://en.wikipedia.org/wiki/Factory_method_pattern)
- [Head First Design Patterns, Chapter 4 — O'Reilly](https://www.oreilly.com/library/view/head-first-design/0596007124/)