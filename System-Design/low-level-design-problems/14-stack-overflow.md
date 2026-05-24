# 14 - Stack Overflow

## 📋 Problem Statement

Design a Q&A platform like Stack Overflow where users can:
- Post questions and answers
- Vote on questions and answers
- Comment, tag, and search
- Track reputation and badges

---

## 📌 Requirements

### Functional Requirements
1. Users can **post questions** with tags
2. Users can **post answers** to questions
3. **Vote** (upvote/downvote) on questions and answers
4. **Accept** an answer (by question author)
5. **Comment** on questions and answers
6. **Search** by title, tags, keywords
7. **Reputation system** — earn/lose points based on votes
8. **Badges** for achievements
9. **Close/Reopen** questions (moderators)

### Non-Functional Requirements
- Efficient search and filtering
- Reputation calculation accuracy
- Pagination for large result sets

---

## 🧩 Core Entities

```
User, Question, Answer, Comment, Vote, Tag, Badge, 
Reputation, SearchService, Notification
```

---

## 💻 Code Implementation

### Votable Interface

```java
public interface Votable {
    void upvote(User voter);
    void downvote(User voter);
    int getScore();
}
```

### Question Class

```java
public class Question implements Votable {
    private String questionId;
    private String title;
    private String body;
    private User author;
    private List<Answer> answers;
    private List<Comment> comments;
    private Set<Tag> tags;
    private Map<String, VoteType> votes; // userId → vote
    private Answer acceptedAnswer;
    private QuestionStatus status;
    private LocalDateTime createdAt;
    private int viewCount;

    public Question(User author, String title, String body, Set<Tag> tags) {
        this.questionId = UUID.randomUUID().toString();
        this.author = author;
        this.title = title;
        this.body = body;
        this.tags = tags;
        this.answers = new ArrayList<>();
        this.comments = new ArrayList<>();
        this.votes = new HashMap<>();
        this.status = QuestionStatus.OPEN;
        this.createdAt = LocalDateTime.now();
        this.viewCount = 0;
    }

    @Override
    public void upvote(User voter) {
        if (voter.equals(author)) throw new IllegalArgumentException("Cannot vote on own post");
        VoteType prev = votes.put(voter.getUserId(), VoteType.UPVOTE);
        author.updateReputation(prev == VoteType.DOWNVOTE ? 15 : 10); // reverse downvote + upvote
    }

    @Override
    public void downvote(User voter) {
        if (voter.equals(author)) throw new IllegalArgumentException("Cannot vote on own post");
        VoteType prev = votes.put(voter.getUserId(), VoteType.DOWNVOTE);
        author.updateReputation(prev == VoteType.UPVOTE ? -15 : -5);
        voter.updateReputation(-1); // downvoting costs 1 rep
    }

    @Override
    public int getScore() {
        return (int) votes.values().stream()
            .mapToInt(v -> v == VoteType.UPVOTE ? 1 : -1)
            .sum();
    }

    public void addAnswer(Answer answer) {
        answers.add(answer);
    }

    public void acceptAnswer(Answer answer, User acceptor) {
        if (!acceptor.equals(author)) throw new UnauthorizedException("Only author can accept");
        this.acceptedAnswer = answer;
        answer.getAuthor().updateReputation(15); // accepted answer bonus
        acceptor.updateReputation(2); // accepting gives 2 rep
    }

    public Comment addComment(User user, String text) {
        Comment comment = new Comment(user, text);
        comments.add(comment);
        return comment;
    }

    public void incrementViewCount() { viewCount++; }

    // Getters
    public String getQuestionId() { return questionId; }
    public User getAuthor() { return author; }
    public String getTitle() { return title; }
    public Set<Tag> getTags() { return tags; }
    public List<Answer> getAnswers() { return answers; }
    public Answer getAcceptedAnswer() { return acceptedAnswer; }
}
```

### Answer Class

```java
public class Answer implements Votable {
    private String answerId;
    private String body;
    private User author;
    private Question question;
    private List<Comment> comments;
    private Map<String, VoteType> votes;
    private boolean isAccepted;
    private LocalDateTime createdAt;

    public Answer(User author, Question question, String body) {
        this.answerId = UUID.randomUUID().toString();
        this.author = author;
        this.question = question;
        this.body = body;
        this.comments = new ArrayList<>();
        this.votes = new HashMap<>();
        this.isAccepted = false;
        this.createdAt = LocalDateTime.now();
    }

    @Override
    public void upvote(User voter) {
        if (voter.equals(author)) throw new IllegalArgumentException("Cannot vote on own post");
        votes.put(voter.getUserId(), VoteType.UPVOTE);
        author.updateReputation(10);
    }

    @Override
    public void downvote(User voter) {
        if (voter.equals(author)) throw new IllegalArgumentException("Cannot vote on own post");
        votes.put(voter.getUserId(), VoteType.DOWNVOTE);
        author.updateReputation(-5);
        voter.updateReputation(-1);
    }

    @Override
    public int getScore() {
        return (int) votes.values().stream()
            .mapToInt(v -> v == VoteType.UPVOTE ? 1 : -1).sum();
    }

    public User getAuthor() { return author; }
}
```

### User with Reputation

```java
public class User {
    private String userId;
    private String username;
    private String email;
    private int reputation;
    private List<Badge> badges;
    private List<Question> questions;
    private List<Answer> answers;

    public User(String userId, String username, String email) {
        this.userId = userId;
        this.username = username;
        this.email = email;
        this.reputation = 1; // starts at 1
        this.badges = new ArrayList<>();
        this.questions = new ArrayList<>();
        this.answers = new ArrayList<>();
    }

    public Question askQuestion(String title, String body, Set<Tag> tags) {
        Question q = new Question(this, title, body, tags);
        questions.add(q);
        return q;
    }

    public Answer answerQuestion(Question question, String body) {
        Answer a = new Answer(this, question, body);
        question.addAnswer(a);
        answers.add(a);
        return a;
    }

    public void updateReputation(int points) {
        this.reputation = Math.max(1, this.reputation + points); // min 1
        checkBadges();
    }

    private void checkBadges() {
        if (reputation >= 1000 && !hasBadge("Gold")) badges.add(Badge.GOLD);
        if (reputation >= 500 && !hasBadge("Silver")) badges.add(Badge.SILVER);
        if (reputation >= 100 && !hasBadge("Bronze")) badges.add(Badge.BRONZE);
    }

    private boolean hasBadge(String name) {
        return badges.stream().anyMatch(b -> b.getName().equals(name));
    }

    public String getUserId() { return userId; }
    public int getReputation() { return reputation; }
}
```

### Reputation Rules

```
+10  Question upvoted
-5   Question downvoted
+10  Answer upvoted
-5   Answer downvoted
+15  Answer accepted
+2   Accepting an answer
-1   Downvoting someone (cost to voter)
```

### Search Service

```java
public class SearchService {
    private Map<String, List<Question>> tagIndex;
    private List<Question> allQuestions;

    public List<Question> searchByKeyword(String keyword) {
        String lower = keyword.toLowerCase();
        return allQuestions.stream()
            .filter(q -> q.getTitle().toLowerCase().contains(lower)
                      || q.getBody().toLowerCase().contains(lower))
            .collect(Collectors.toList());
    }

    public List<Question> searchByTag(String tagName) {
        return tagIndex.getOrDefault(tagName.toLowerCase(), Collections.emptyList());
    }

    public List<Question> getTopQuestions(int limit) {
        return allQuestions.stream()
            .sorted(Comparator.comparingInt(Question::getScore).reversed())
            .limit(limit)
            .collect(Collectors.toList());
    }

    public List<Question> getUnanswered() {
        return allQuestions.stream()
            .filter(q -> q.getAnswers().isEmpty())
            .collect(Collectors.toList());
    }
}
```

---

## 🧪 Usage Example

```java
User alice = new User("U1", "alice", "alice@email.com");
User bob = new User("U2", "bob", "bob@email.com");

// Alice asks a question
Question q = alice.askQuestion(
    "How does HashMap work in Java?",
    "I want to understand the internal working...",
    Set.of(new Tag("java"), new Tag("hashmap"))
);

// Bob answers
Answer a = bob.answerQuestion(q, "HashMap uses an array of linked lists...");

// Alice upvotes Bob's answer
a.upvote(alice);        // Bob gets +10 rep

// Alice accepts Bob's answer
q.acceptAnswer(a, alice); // Bob gets +15, Alice gets +2

System.out.println(bob.getReputation());   // 26 (1 + 10 + 15)
System.out.println(alice.getReputation()); // 3 (1 + 2)
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Observer** | Notifications for votes, answers, comments |
| **Strategy** | Search and sort strategies |
| **Composite** | Comment threads (nested comments) |
| **Decorator** | Badges decorating user profiles |

---

## 🔑 Key Takeaways

1. **Votable interface** — shared by Question and Answer
2. **Reputation system** with defined point rules
3. **Vote tracking** per user prevents double voting
4. **Tag-based indexing** enables efficient search
5. Self-voting is **prohibited**
