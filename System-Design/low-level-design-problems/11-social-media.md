# 11 - Social Media System (Twitter/Instagram)

## 📋 Problem Statement

Design a social media platform that supports:
- User profiles and follow/unfollow
- Creating posts (text, image, video)
- News feed generation
- Like, comment, and share functionality

---

## 📌 Requirements

### Functional Requirements
1. **User registration** and profile management
2. **Follow / Unfollow** other users
3. **Create posts** — text, image, video
4. **News feed** — show posts from followed users (sorted by time)
5. **Like** and **Unlike** posts
6. **Comment** on posts
7. **Share / Repost**
8. **Search** users and posts by hashtag
9. **Notifications** for likes, comments, follows

### Non-Functional Requirements
- Feed generation should be efficient
- Handle large follower counts
- Real-time notifications

---

## 🧩 Core Entities

```
User, Post, Comment, Like, Feed, Follow, Notification,
Media, Hashtag, UserProfile
```

---

## 📐 Class Diagram

```
┌──────────────────────────────────────────────────────┐
│                       User                            │
│  - userId: String                                     │
│  - username: String                                   │
│  - email: String                                      │
│  - profile: UserProfile                               │
│  - followers: Set<User>                               │
│  - following: Set<User>                               │
│  - posts: List<Post>                                  │
│  + follow(user): void                                 │
│  + unfollow(user): void                               │
│  + createPost(content, media): Post                   │
│  + getFeed(): List<Post>                              │
└──────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────┐
│                       Post                            │
│  - postId: String                                     │
│  - author: User                                       │
│  - content: String                                    │
│  - media: List<Media>                                 │
│  - likes: Set<User>                                   │
│  - comments: List<Comment>                            │
│  - hashtags: Set<String>                              │
│  - createdAt: LocalDateTime                           │
│  + like(user): void                                   │
│  + unlike(user): void                                 │
│  + addComment(user, text): Comment                    │
│  + getLikeCount(): int                                │
└──────────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐
│    Comment        │    │     Media        │
│  - commentId      │    │  - mediaId       │
│  - user           │    │  - url           │
│  - text           │    │  - type          │
│  - createdAt      │    │  (IMAGE, VIDEO)  │
│  - likes          │    │                  │
└──────────────────┘    └──────────────────┘
```

---

## 💻 Code Implementation

### User Class

```java
public class User {
    private String userId;
    private String username;
    private String email;
    private UserProfile profile;
    private Set<String> followers;    // userIds
    private Set<String> following;    // userIds
    private List<Post> posts;

    public User(String userId, String username, String email) {
        this.userId = userId;
        this.username = username;
        this.email = email;
        this.profile = new UserProfile();
        this.followers = new HashSet<>();
        this.following = new HashSet<>();
        this.posts = new ArrayList<>();
    }

    public void follow(User other) {
        if (other.userId.equals(this.userId)) throw new IllegalArgumentException("Cannot follow yourself");
        this.following.add(other.userId);
        other.followers.add(this.userId);
    }

    public void unfollow(User other) {
        this.following.remove(other.userId);
        other.followers.remove(this.userId);
    }

    public Post createPost(String content, List<Media> media) {
        Post post = new Post(this, content, media);
        posts.add(post);
        return post;
    }

    public int getFollowerCount() { return followers.size(); }
    public int getFollowingCount() { return following.size(); }
    public String getUserId() { return userId; }
    public Set<String> getFollowing() { return following; }
    public List<Post> getPosts() { return posts; }
}
```

### Post Class

```java
public class Post {
    private String postId;
    private User author;
    private String content;
    private List<Media> media;
    private Set<String> likes;        // userIds who liked
    private List<Comment> comments;
    private Set<String> hashtags;
    private LocalDateTime createdAt;

    public Post(User author, String content, List<Media> media) {
        this.postId = UUID.randomUUID().toString();
        this.author = author;
        this.content = content;
        this.media = media != null ? media : new ArrayList<>();
        this.likes = new HashSet<>();
        this.comments = new ArrayList<>();
        this.hashtags = extractHashtags(content);
        this.createdAt = LocalDateTime.now();
    }

    public void like(User user) {
        likes.add(user.getUserId());
    }

    public void unlike(User user) {
        likes.remove(user.getUserId());
    }

    public Comment addComment(User user, String text) {
        Comment comment = new Comment(user, text);
        comments.add(comment);
        return comment;
    }

    private Set<String> extractHashtags(String content) {
        Set<String> tags = new HashSet<>();
        String[] words = content.split("\\s+");
        for (String word : words) {
            if (word.startsWith("#") && word.length() > 1) {
                tags.add(word.toLowerCase());
            }
        }
        return tags;
    }

    public int getLikeCount() { return likes.size(); }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public User getAuthor() { return author; }
    public String getPostId() { return postId; }
    public Set<String> getHashtags() { return hashtags; }
}
```

### Comment Class

```java
public class Comment {
    private String commentId;
    private User user;
    private String text;
    private LocalDateTime createdAt;
    private Set<String> likes;

    public Comment(User user, String text) {
        this.commentId = UUID.randomUUID().toString();
        this.user = user;
        this.text = text;
        this.createdAt = LocalDateTime.now();
        this.likes = new HashSet<>();
    }

    public void like(User user) { likes.add(user.getUserId()); }
    public void unlike(User user) { likes.remove(user.getUserId()); }
}
```

### Feed Service

```java
public class FeedService {
    private Map<String, User> users; // userId → User

    public FeedService(Map<String, User> users) {
        this.users = users;
    }

    /**
     * Pull-based feed: gather posts from followed users and sort by time.
     * Simple approach — suitable for small-medium scale.
     */
    public List<Post> generateFeed(User user, int limit) {
        List<Post> feed = new ArrayList<>();

        for (String followedId : user.getFollowing()) {
            User followedUser = users.get(followedId);
            if (followedUser != null) {
                feed.addAll(followedUser.getPosts());
            }
        }

        // Sort by time (newest first)
        feed.sort((a, b) -> b.getCreatedAt().compareTo(a.getCreatedAt()));

        // Return top N posts
        return feed.stream().limit(limit).collect(Collectors.toList());
    }

    /**
     * Push-based feed (Fan-out on write):
     * When a user creates a post, push it to all followers' feed cache.
     * Better for read-heavy scenarios with limited followers.
     */
    public void fanOutPost(Post post, User author) {
        // For each follower, add post to their cached feed
        // (In real system, this would be a Redis/cache operation)
        for (String followerId : author.getFollowing()) {
            // Add to follower's pre-computed feed
        }
    }
}
```

### Notification Service (Observer Pattern)

```java
public interface NotificationObserver {
    void onNotification(Notification notification);
}

public class Notification {
    private String notificationId;
    private User recipient;
    private User actor;
    private NotificationType type;
    private String message;
    private LocalDateTime createdAt;
    private boolean read;

    public Notification(User recipient, User actor, NotificationType type, String message) {
        this.notificationId = UUID.randomUUID().toString();
        this.recipient = recipient;
        this.actor = actor;
        this.type = type;
        this.message = message;
        this.createdAt = LocalDateTime.now();
        this.read = false;
    }
}

public enum NotificationType {
    LIKE, COMMENT, FOLLOW, MENTION, SHARE
}

public class NotificationService {
    private Map<String, List<Notification>> userNotifications; // userId → notifications

    public void notify(User recipient, User actor, NotificationType type, String entityId) {
        String message = buildMessage(actor, type);
        Notification notification = new Notification(recipient, actor, type, message);
        userNotifications.computeIfAbsent(recipient.getUserId(), k -> new ArrayList<>()).add(notification);
    }

    private String buildMessage(User actor, NotificationType type) {
        return switch (type) {
            case LIKE -> actor.getUsername() + " liked your post";
            case COMMENT -> actor.getUsername() + " commented on your post";
            case FOLLOW -> actor.getUsername() + " started following you";
            case MENTION -> actor.getUsername() + " mentioned you";
            case SHARE -> actor.getUsername() + " shared your post";
        };
    }

    public List<Notification> getNotifications(String userId) {
        return userNotifications.getOrDefault(userId, Collections.emptyList());
    }
}
```

### Search Service

```java
public class SearchService {
    private Map<String, User> usernameIndex;
    private Map<String, List<Post>> hashtagIndex;

    public void indexPost(Post post) {
        for (String tag : post.getHashtags()) {
            hashtagIndex.computeIfAbsent(tag, k -> new ArrayList<>()).add(post);
        }
    }

    public User searchUser(String username) {
        return usernameIndex.get(username.toLowerCase());
    }

    public List<Post> searchByHashtag(String hashtag) {
        String tag = hashtag.startsWith("#") ? hashtag.toLowerCase() : "#" + hashtag.toLowerCase();
        return hashtagIndex.getOrDefault(tag, Collections.emptyList());
    }
}
```

---

## 🧪 Usage Example

```java
// Create users
User alice = new User("U1", "alice", "alice@email.com");
User bob = new User("U2", "bob", "bob@email.com");
User charlie = new User("U3", "charlie", "charlie@email.com");

// Follow
alice.follow(bob);
alice.follow(charlie);

// Create posts
Post post1 = bob.createPost("Hello world! #first", null);
Post post2 = charlie.createPost("Beautiful sunset #nature #photography", 
    List.of(new Media("img1.jpg", MediaType.IMAGE)));

// Like and comment
post1.like(alice);
post1.addComment(alice, "Welcome!");

// Generate feed
FeedService feedService = new FeedService(Map.of("U2", bob, "U3", charlie));
List<Post> feed = feedService.generateFeed(alice, 10);
// Shows: charlie's post, then bob's post (newest first)
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Observer** | Notifications when someone likes/comments/follows |
| **Strategy** | Feed algorithm (chronological vs ranked) |
| **Decorator** | Post types (text, image, video, story) |
| **Factory** | Create different notification types |

---

## 📊 Feed Generation Approaches

| Approach | When to Use |
|----------|-------------|
| **Pull (Fan-out on read)** | Celebrity users (millions of followers) |
| **Push (Fan-out on write)** | Regular users (few hundred followers) |
| **Hybrid** | Push for regular users, Pull for celebrities |

---

## ⚠️ Edge Cases

- User follows/unfollows rapidly → debounce
- Self-follow → reject
- Delete post → remove from all feeds, update indexes
- Private account → only approved followers see posts
- Block user → hide from each other
- Celebrity with millions of followers → use pull-based feed

---

## 🔑 Key Takeaways

1. **Pull vs Push** feed generation — depends on follower count
2. **HashSet** for likes/followers — O(1) add/remove/check
3. **Hashtag extraction** from post content for searchability
4. **Observer pattern** for real-time notifications
5. **Separate Feed Service** from User for single responsibility
