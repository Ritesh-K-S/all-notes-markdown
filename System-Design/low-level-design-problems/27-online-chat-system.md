# 27 - Online Chat System (WhatsApp)

## 📋 Problem Statement

Design a messaging system like WhatsApp that supports:
- One-to-one and group messaging
- Online/offline status
- Message delivery status (sent, delivered, read)
- Media sharing

---

## 📌 Requirements

### Functional Requirements
1. **One-to-one** private messaging
2. **Group chat** — create, add/remove members, admin controls
3. **Message status**: SENT → DELIVERED → READ
4. **Online/Offline/Last Seen** status
5. **Media messages** — image, video, document
6. **Message history** / conversation thread
7. **Typing indicator**
8. **Block/Unblock** users

### Non-Functional Requirements
- Messages delivered in order
- Handle offline users (deliver when online)
- Real-time delivery

---

## 🧩 Core Entities

```
User, Chat, PrivateChat, GroupChat, Message, 
MessageStatus, Media, UserStatus, ChatService
```

---

## 📐 Class Diagram

```
┌──────────────────────────────────────────────────────┐
│                       User                            │
│  - userId: String                                     │
│  - name: String                                       │
│  - phone: String                                      │
│  - status: UserStatus                                 │
│  - lastSeen: LocalDateTime                            │
│  - chats: Map<String, Chat>                           │
│  - blockedUsers: Set<String>                          │
│  + sendMessage(chatId, content): Message              │
│  + createGroup(name, members): GroupChat               │
│  + block(userId): void                                │
└──────────────────────────────────────────────────────┘

                  ┌──────────────┐
                  │  Chat (abs)  │
                  │  - chatId    │
                  │  - messages  │
                  │  + send()    │
                  └──────┬───────┘
                         │
          ┌──────────────┼──────────────┐
          │                             │
┌─────────▼──────────┐      ┌──────────▼───────────┐
│   PrivateChat       │      │     GroupChat          │
│  - user1            │      │  - name                │
│  - user2            │      │  - members: List<User> │
│                     │      │  - admins: Set<User>   │
│                     │      │  + addMember()          │
│                     │      │  + removeMember()       │
└────────────────────┘      └──────────────────────┘

┌──────────────────────────────────────────────────────┐
│                     Message                           │
│  - messageId: String                                  │
│  - sender: User                                       │
│  - content: String                                    │
│  - media: Media                                       │
│  - type: MessageType                                  │
│  - timestamp: LocalDateTime                           │
│  - statusMap: Map<String, MessageStatus>              │
│    (per recipient: SENT → DELIVERED → READ)           │
└──────────────────────────────────────────────────────┘
```

---

## 🔧 Enums

```java
public enum MessageType {
    TEXT, IMAGE, VIDEO, DOCUMENT, AUDIO, LOCATION
}

public enum MessageStatus {
    SENT, DELIVERED, READ
}

public enum UserStatus {
    ONLINE, OFFLINE, BUSY, AWAY
}
```

---

## 💻 Code Implementation

### Message Class

```java
public class Message {
    private String messageId;
    private User sender;
    private String content;
    private Media media;
    private MessageType type;
    private LocalDateTime timestamp;
    private Map<String, MessageStatus> statusMap; // recipientId → status

    public Message(User sender, String content, MessageType type) {
        this.messageId = UUID.randomUUID().toString();
        this.sender = sender;
        this.content = content;
        this.type = type;
        this.timestamp = LocalDateTime.now();
        this.statusMap = new HashMap<>();
    }

    public Message(User sender, Media media) {
        this(sender, "", media.getType());
        this.media = media;
    }

    public void markDelivered(String recipientId) {
        statusMap.put(recipientId, MessageStatus.DELIVERED);
    }

    public void markRead(String recipientId) {
        statusMap.put(recipientId, MessageStatus.READ);
    }

    public void addRecipient(String recipientId) {
        statusMap.put(recipientId, MessageStatus.SENT);
    }

    // Getters
    public String getMessageId() { return messageId; }
    public User getSender() { return sender; }
    public String getContent() { return content; }
    public LocalDateTime getTimestamp() { return timestamp; }
    public MessageStatus getStatus(String recipientId) { return statusMap.get(recipientId); }
}
```

### Chat (Abstract), PrivateChat, GroupChat

```java
public abstract class Chat {
    protected String chatId;
    protected List<Message> messages;
    protected LocalDateTime createdAt;

    public Chat() {
        this.chatId = UUID.randomUUID().toString();
        this.messages = new ArrayList<>();
        this.createdAt = LocalDateTime.now();
    }

    public abstract void sendMessage(Message message);
    public abstract List<User> getParticipants();

    public List<Message> getMessages(int limit) {
        int start = Math.max(0, messages.size() - limit);
        return messages.subList(start, messages.size());
    }

    public String getChatId() { return chatId; }
}

public class PrivateChat extends Chat {
    private User user1;
    private User user2;

    public PrivateChat(User user1, User user2) {
        super();
        this.user1 = user1;
        this.user2 = user2;
    }

    @Override
    public void sendMessage(Message message) {
        User recipient = message.getSender().equals(user1) ? user2 : user1;

        // Check if blocked
        if (recipient.isBlocked(message.getSender().getUserId())) {
            throw new UserBlockedException("You are blocked by this user");
        }

        message.addRecipient(recipient.getUserId());
        messages.add(message);

        // Deliver if online
        if (recipient.getStatus() == UserStatus.ONLINE) {
            message.markDelivered(recipient.getUserId());
        }

        System.out.println(message.getSender().getName() + " → " + recipient.getName()
            + ": " + message.getContent());
    }

    @Override
    public List<User> getParticipants() {
        return List.of(user1, user2);
    }
}

public class GroupChat extends Chat {
    private String name;
    private List<User> members;
    private Set<String> adminIds;
    private int maxMembers;

    public GroupChat(String name, User creator, List<User> initialMembers) {
        super();
        this.name = name;
        this.members = new ArrayList<>();
        this.adminIds = new HashSet<>();
        this.maxMembers = 256;

        // Creator is auto-admin
        members.add(creator);
        adminIds.add(creator.getUserId());
        members.addAll(initialMembers);
    }

    @Override
    public void sendMessage(Message message) {
        if (!members.contains(message.getSender())) {
            throw new NotAMemberException("You are not a member of this group");
        }

        for (User member : members) {
            if (!member.equals(message.getSender())) {
                message.addRecipient(member.getUserId());
                if (member.getStatus() == UserStatus.ONLINE) {
                    message.markDelivered(member.getUserId());
                }
            }
        }

        messages.add(message);
        System.out.println("[" + name + "] " + message.getSender().getName()
            + ": " + message.getContent());
    }

    public void addMember(User admin, User newMember) {
        if (!adminIds.contains(admin.getUserId())) {
            throw new UnauthorizedException("Only admins can add members");
        }
        if (members.size() >= maxMembers) {
            throw new GroupFullException("Group is full");
        }
        if (!members.contains(newMember)) {
            members.add(newMember);
        }
    }

    public void removeMember(User admin, User member) {
        if (!adminIds.contains(admin.getUserId())) {
            throw new UnauthorizedException("Only admins can remove members");
        }
        members.remove(member);
        adminIds.remove(member.getUserId());
    }

    public void makeAdmin(User admin, User member) {
        if (!adminIds.contains(admin.getUserId())) {
            throw new UnauthorizedException("Only admins can promote");
        }
        adminIds.add(member.getUserId());
    }

    @Override
    public List<User> getParticipants() { return members; }
    public String getName() { return name; }
}
```

### User Class

```java
public class User {
    private String userId;
    private String name;
    private String phone;
    private UserStatus status;
    private LocalDateTime lastSeen;
    private Map<String, Chat> chats; // chatId → Chat
    private Set<String> blockedUsers;

    public User(String userId, String name, String phone) {
        this.userId = userId;
        this.name = name;
        this.phone = phone;
        this.status = UserStatus.OFFLINE;
        this.chats = new HashMap<>();
        this.blockedUsers = new HashSet<>();
    }

    public Message sendMessage(String chatId, String content) {
        Chat chat = chats.get(chatId);
        if (chat == null) throw new ChatNotFoundException("Chat not found");

        Message message = new Message(this, content, MessageType.TEXT);
        chat.sendMessage(message);
        return message;
    }

    public void goOnline() {
        this.status = UserStatus.ONLINE;
        this.lastSeen = LocalDateTime.now();
    }

    public void goOffline() {
        this.status = UserStatus.OFFLINE;
        this.lastSeen = LocalDateTime.now();
    }

    public void block(String userId) { blockedUsers.add(userId); }
    public void unblock(String userId) { blockedUsers.remove(userId); }
    public boolean isBlocked(String userId) { return blockedUsers.contains(userId); }

    public void addChat(Chat chat) { chats.put(chat.getChatId(), chat); }

    // Getters
    public String getUserId() { return userId; }
    public String getName() { return name; }
    public UserStatus getStatus() { return status; }
    public LocalDateTime getLastSeen() { return lastSeen; }
}
```

### Chat Service (Controller)

```java
public class ChatService {
    private Map<String, User> users;
    private Map<String, Chat> chats;

    public ChatService() {
        this.users = new ConcurrentHashMap<>();
        this.chats = new ConcurrentHashMap<>();
    }

    public void registerUser(User user) {
        users.put(user.getUserId(), user);
    }

    public PrivateChat startPrivateChat(User user1, User user2) {
        PrivateChat chat = new PrivateChat(user1, user2);
        chats.put(chat.getChatId(), chat);
        user1.addChat(chat);
        user2.addChat(chat);
        return chat;
    }

    public GroupChat createGroup(String name, User creator, List<User> members) {
        GroupChat group = new GroupChat(name, creator, members);
        chats.put(group.getChatId(), group);
        creator.addChat(group);
        for (User member : members) {
            member.addChat(group);
        }
        return group;
    }

    // Deliver pending messages when user comes online
    public void onUserOnline(User user) {
        user.goOnline();
        for (Chat chat : user.getChats().values()) {
            for (Message msg : chat.getMessages(100)) {
                if (msg.getStatus(user.getUserId()) == MessageStatus.SENT) {
                    msg.markDelivered(user.getUserId());
                }
            }
        }
    }
}
```

---

## 🧪 Usage Example

```java
ChatService service = new ChatService();

User alice = new User("U1", "Alice", "9876543210");
User bob = new User("U2", "Bob", "9876543211");
User charlie = new User("U3", "Charlie", "9876543212");

service.registerUser(alice);
service.registerUser(bob);
service.registerUser(charlie);

alice.goOnline();
bob.goOnline();

// Private chat
PrivateChat pvtChat = service.startPrivateChat(alice, bob);
alice.sendMessage(pvtChat.getChatId(), "Hey Bob!");
bob.sendMessage(pvtChat.getChatId(), "Hi Alice!");

// Group chat
GroupChat group = service.createGroup("College Friends", alice, List.of(bob, charlie));
alice.sendMessage(group.getChatId(), "Hello group!");
// → [College Friends] Alice: Hello group!

// Block
alice.block(bob.getUserId());
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Observer** | Message delivery notification when user comes online |
| **Strategy** | Message delivery (online vs offline queuing) |
| **Factory** | Creating PrivateChat vs GroupChat |
| **Mediator** | ChatService mediates between users |

---

## ⚠️ Edge Cases

- Send message to blocked user → reject
- User offline → queue message, deliver on login
- Group member removed → cannot send messages
- Read receipts in group → track per-member
- Concurrent messages → maintain message ordering by timestamp
- Max group size → enforce limit

---

## 🔑 Key Takeaways

1. **PrivateChat vs GroupChat** — polymorphic `sendMessage()`
2. **Message status** tracked per-recipient (`Map<userId, Status>`)
3. **Offline delivery** — messages with SENT status get marked DELIVERED when user comes online
4. **Block** is checked before message delivery
5. **Group admin controls** — add/remove/promote enforced
