# 22 - Notification System

## 📋 Problem Statement

Design a notification system that:
- Supports multiple channels (Email, SMS, Push Notification)
- Handles priority-based delivery
- Supports templating and user preferences

---

## 📌 Requirements

### Functional Requirements
1. Send notifications via **Email, SMS, Push**
2. User can set **channel preferences** (opt-in/opt-out)
3. Support **notification templates** with variables
4. **Priority levels**: LOW, MEDIUM, HIGH, CRITICAL
5. **Retry** on delivery failure
6. **Notification history** per user

---

## 💻 Code Implementation

### Notification Channel — Strategy + Decorator

```java
public interface NotificationChannel {
    boolean send(String recipient, String subject, String body);
    String getChannelType();
}

public class EmailChannel implements NotificationChannel {
    @Override
    public boolean send(String recipient, String subject, String body) {
        System.out.println("📧 Email to " + recipient + ": " + subject);
        return true;
    }

    @Override
    public String getChannelType() { return "EMAIL"; }
}

public class SMSChannel implements NotificationChannel {
    @Override
    public boolean send(String recipient, String subject, String body) {
        System.out.println("📱 SMS to " + recipient + ": " + body);
        return true;
    }

    @Override
    public String getChannelType() { return "SMS"; }
}

public class PushChannel implements NotificationChannel {
    @Override
    public boolean send(String recipient, String subject, String body) {
        System.out.println("🔔 Push to " + recipient + ": " + subject);
        return true;
    }

    @Override
    public String getChannelType() { return "PUSH"; }
}
```

### Notification Template

```java
public class NotificationTemplate {
    private String templateId;
    private String subjectTemplate;
    private String bodyTemplate;

    public NotificationTemplate(String id, String subject, String body) {
        this.templateId = id;
        this.subjectTemplate = subject;
        this.bodyTemplate = body;
    }

    public String renderSubject(Map<String, String> variables) {
        return replaceVariables(subjectTemplate, variables);
    }

    public String renderBody(Map<String, String> variables) {
        return replaceVariables(bodyTemplate, variables);
    }

    private String replaceVariables(String template, Map<String, String> variables) {
        String result = template;
        for (Map.Entry<String, String> entry : variables.entrySet()) {
            result = result.replace("{{" + entry.getKey() + "}}", entry.getValue());
        }
        return result;
    }
}
```

### Notification Service

```java
public class NotificationService {
    private Map<String, NotificationChannel> channels;
    private Map<String, NotificationTemplate> templates;
    private Map<String, UserPreference> userPreferences;
    private List<Notification> history;

    public NotificationService() {
        channels = Map.of(
            "EMAIL", new EmailChannel(),
            "SMS", new SMSChannel(),
            "PUSH", new PushChannel()
        );
        templates = new HashMap<>();
        userPreferences = new HashMap<>();
        history = new ArrayList<>();
    }

    public void send(String userId, String templateId, 
                     Map<String, String> variables, Priority priority) {
        NotificationTemplate template = templates.get(templateId);
        if (template == null) throw new TemplateNotFoundException("Template not found");

        String subject = template.renderSubject(variables);
        String body = template.renderBody(variables);

        UserPreference pref = userPreferences.getOrDefault(userId, UserPreference.defaultPrefs());

        for (String channelType : pref.getEnabledChannels()) {
            NotificationChannel channel = channels.get(channelType);
            if (channel != null) {
                boolean sent = sendWithRetry(channel, userId, subject, body, 3);

                Notification notification = new Notification(userId, channelType, subject, body, priority, sent);
                history.add(notification);
            }
        }
    }

    private boolean sendWithRetry(NotificationChannel channel, String recipient,
                                   String subject, String body, int maxRetries) {
        for (int i = 0; i < maxRetries; i++) {
            if (channel.send(recipient, subject, body)) return true;
        }
        return false;
    }

    public void registerTemplate(NotificationTemplate template) {
        templates.put(template.getTemplateId(), template);
    }

    public void setUserPreference(String userId, UserPreference pref) {
        userPreferences.put(userId, pref);
    }
}
```

### User Preferences

```java
public class UserPreference {
    private Set<String> enabledChannels;
    private boolean doNotDisturb;

    public UserPreference(Set<String> enabledChannels) {
        this.enabledChannels = enabledChannels;
        this.doNotDisturb = false;
    }

    public static UserPreference defaultPrefs() {
        return new UserPreference(Set.of("EMAIL", "PUSH"));
    }

    public Set<String> getEnabledChannels() {
        return doNotDisturb ? Collections.emptySet() : enabledChannels;
    }
}
```

---

## 🧪 Usage Example

```java
NotificationService service = new NotificationService();

// Register template
service.registerTemplate(new NotificationTemplate(
    "ORDER_CONFIRMED",
    "Order {{orderId}} Confirmed",
    "Hi {{name}}, your order {{orderId}} has been confirmed. Total: ₹{{amount}}"
));

// Set user preference
service.setUserPreference("U1", new UserPreference(Set.of("EMAIL", "SMS")));

// Send notification
service.send("U1", "ORDER_CONFIRMED", Map.of(
    "name", "Ritesh",
    "orderId", "ORD-001",
    "amount", "999"
), Priority.HIGH);
// → 📧 Email to U1: Order ORD-001 Confirmed
// → 📱 SMS to U1: Hi Ritesh, your order ORD-001 has been confirmed. Total: ₹999
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Notification channels (Email, SMS, Push) |
| **Observer** | Event-driven notification triggering |
| **Decorator** | Can add logging/retry decorators around channels |
| **Template Method** | Notification template rendering |
| **Factory** | Channel creation based on type |

---

## 🔑 Key Takeaways

1. **Strategy** for notification channels — easily add new ones
2. **Template variables** with `{{placeholder}}` syntax
3. **User preferences** control which channels are active
4. **Retry logic** for transient delivery failures
5. **Priority** can drive routing (CRITICAL → all channels, LOW → email only)
