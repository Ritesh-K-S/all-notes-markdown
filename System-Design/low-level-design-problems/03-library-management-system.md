# 03 - Library Management System

## 📋 Problem Statement

Design a Library Management System that allows:
- Members to search, borrow, return, and reserve books
- Librarians to manage books and members
- Automated fine calculation for late returns

---

## 📌 Requirements

### Functional Requirements
1. **Search** books by title, author, ISBN, category
2. Members can **borrow** books (max 5 at a time, for 14 days)
3. Members can **return** books
4. Members can **reserve** a book if currently borrowed by someone
5. System sends **notifications** for due dates and reservations
6. **Fine calculation** for overdue books
7. Librarian can **add/remove books** and **manage members**
8. Each book can have **multiple copies**

### Non-Functional Requirements
- Handle concurrent borrow/return
- Maintain audit trail of transactions
- Notification system (email/SMS)

---

## 🧩 Core Entities

```
Library, Book, BookItem (physical copy), Member, Librarian, 
BookLending, BookReservation, Fine, Notification, Rack, Account
```

---

## 📐 Class Diagram

```
                      ┌──────────────┐
                      │   Account    │
                      │  - username  │
                      │  - password  │
                      │  - status    │
                      └──────┬───────┘
                             │
              ┌──────────────┼──────────────┐
              │                             │
     ┌────────▼────────┐          ┌─────────▼────────┐
     │     Member       │          │    Librarian      │
     │  - totalBooks    │          │  + addBook()      │
     │  - borrowedBooks │          │  + removeBook()   │
     │  + borrowBook()  │          │  + blockMember()  │
     │  + returnBook()  │          │  + addMember()    │
     │  + reserveBook() │          │                   │
     └─────────────────┘          └───────────────────┘

┌──────────────────────────────────────────────────────┐
│                        Book                           │
│  - isbn: String                                       │
│  - title: String                                      │
│  - author: String                                     │
│  - category: BookCategory                             │
│  - copies: List<BookItem>                             │
└──────────────────┬───────────────────────────────────┘
                   │ 1..*
┌──────────────────▼───────────────────────────────────┐
│                     BookItem                          │
│  - barcode: String                                    │
│  - status: BookStatus                                 │
│  - rack: Rack                                         │
│  - dueDate: LocalDate                                 │
│  - borrowedBy: Member                                 │
│  + checkout(member): void                             │
│  + checkin(): void                                    │
└──────────────────────────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐
│  BookLending      │    │  BookReservation  │
│  - lendingId      │    │  - reservationId  │
│  - bookItem       │    │  - bookItem       │
│  - member         │    │  - member         │
│  - lendDate       │    │  - reserveDate    │
│  - dueDate        │    │  - status         │
│  - returnDate     │    │                   │
└──────────────────┘    └──────────────────┘
```

---

## 🔧 Enums

```java
public enum BookStatus {
    AVAILABLE, BORROWED, RESERVED, LOST
}

public enum BookCategory {
    FICTION, NON_FICTION, SCIENCE, HISTORY, TECHNOLOGY, BIOGRAPHY
}

public enum AccountStatus {
    ACTIVE, BLOCKED, CLOSED
}

public enum ReservationStatus {
    PENDING, FULFILLED, CANCELLED
}
```

---

## 💻 Code Implementation

### Book and BookItem

```java
public class Book {
    private String isbn;
    private String title;
    private String author;
    private BookCategory category;
    private List<BookItem> copies;

    public Book(String isbn, String title, String author, BookCategory category) {
        this.isbn = isbn;
        this.title = title;
        this.author = author;
        this.category = category;
        this.copies = new ArrayList<>();
    }

    public void addCopy(BookItem item) {
        copies.add(item);
    }

    public BookItem getAvailableCopy() {
        return copies.stream()
                     .filter(item -> item.getStatus() == BookStatus.AVAILABLE)
                     .findFirst()
                     .orElse(null);
    }

    // Getters
    public String getIsbn() { return isbn; }
    public String getTitle() { return title; }
    public String getAuthor() { return author; }
}

public class BookItem {
    private String barcode;
    private BookStatus status;
    private Rack rack;
    private LocalDate dueDate;
    private Member borrowedBy;
    private Book parentBook;

    public BookItem(String barcode, Book parentBook, Rack rack) {
        this.barcode = barcode;
        this.parentBook = parentBook;
        this.rack = rack;
        this.status = BookStatus.AVAILABLE;
    }

    public synchronized boolean checkout(Member member) {
        if (status != BookStatus.AVAILABLE) return false;
        this.status = BookStatus.BORROWED;
        this.borrowedBy = member;
        this.dueDate = LocalDate.now().plusDays(14);
        return true;
    }

    public synchronized void checkin() {
        this.status = BookStatus.AVAILABLE;
        this.borrowedBy = null;
        this.dueDate = null;
    }

    // Getters
    public BookStatus getStatus() { return status; }
    public LocalDate getDueDate() { return dueDate; }
    public Member getBorrowedBy() { return borrowedBy; }
}
```

### Member Class

```java
public class Member {
    private String memberId;
    private String name;
    private String email;
    private AccountStatus status;
    private List<BookLending> borrowedBooks;
    private List<BookReservation> reservations;

    private static final int MAX_BOOKS = 5;
    private static final int LOAN_DAYS = 14;

    public Member(String memberId, String name, String email) {
        this.memberId = memberId;
        this.name = name;
        this.email = email;
        this.status = AccountStatus.ACTIVE;
        this.borrowedBooks = new ArrayList<>();
        this.reservations = new ArrayList<>();
    }

    public BookLending borrowBook(BookItem bookItem) {
        if (status != AccountStatus.ACTIVE) {
            throw new AccountInactiveException("Account is not active");
        }
        if (borrowedBooks.size() >= MAX_BOOKS) {
            throw new BookLimitExceededException("Cannot borrow more than " + MAX_BOOKS + " books");
        }
        if (!bookItem.checkout(this)) {
            throw new BookNotAvailableException("Book is not available");
        }

        BookLending lending = new BookLending(bookItem, this);
        borrowedBooks.add(lending);
        return lending;
    }

    public Fine returnBook(BookItem bookItem) {
        BookLending lending = borrowedBooks.stream()
            .filter(l -> l.getBookItem().equals(bookItem))
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Book not borrowed by this member"));

        lending.setReturnDate(LocalDate.now());
        bookItem.checkin();
        borrowedBooks.remove(lending);

        // Check for reservations
        checkAndNotifyReservation(bookItem);

        // Calculate fine if overdue
        if (LocalDate.now().isAfter(lending.getDueDate())) {
            long overdueDays = ChronoUnit.DAYS.between(lending.getDueDate(), LocalDate.now());
            return new Fine(overdueDays * 2.0); // ₹2 per day
        }
        return null;
    }

    public BookReservation reserveBook(BookItem bookItem) {
        if (bookItem.getStatus() == BookStatus.AVAILABLE) {
            throw new IllegalStateException("Book is available, borrow it directly");
        }
        BookReservation reservation = new BookReservation(bookItem, this);
        reservations.add(reservation);
        return reservation;
    }

    private void checkAndNotifyReservation(BookItem bookItem) {
        // Notify the first person who reserved this book
        // Observer pattern in action
    }
}
```

### BookLending and BookReservation

```java
public class BookLending {
    private String lendingId;
    private BookItem bookItem;
    private Member member;
    private LocalDate lendDate;
    private LocalDate dueDate;
    private LocalDate returnDate;

    public BookLending(BookItem bookItem, Member member) {
        this.lendingId = UUID.randomUUID().toString();
        this.bookItem = bookItem;
        this.member = member;
        this.lendDate = LocalDate.now();
        this.dueDate = LocalDate.now().plusDays(14);
    }

    // Getters and setters
    public LocalDate getDueDate() { return dueDate; }
    public BookItem getBookItem() { return bookItem; }
    public void setReturnDate(LocalDate date) { this.returnDate = date; }
}

public class BookReservation {
    private String reservationId;
    private BookItem bookItem;
    private Member member;
    private LocalDate reservationDate;
    private ReservationStatus status;

    public BookReservation(BookItem bookItem, Member member) {
        this.reservationId = UUID.randomUUID().toString();
        this.bookItem = bookItem;
        this.member = member;
        this.reservationDate = LocalDate.now();
        this.status = ReservationStatus.PENDING;
    }
}
```

### Search System

```java
public class SearchService {
    private Map<String, List<Book>> titleIndex;
    private Map<String, List<Book>> authorIndex;
    private Map<String, Book> isbnIndex;
    private Map<BookCategory, List<Book>> categoryIndex;

    public SearchService() {
        titleIndex = new HashMap<>();
        authorIndex = new HashMap<>();
        isbnIndex = new HashMap<>();
        categoryIndex = new HashMap<>();
    }

    public void addBook(Book book) {
        isbnIndex.put(book.getIsbn(), book);
        titleIndex.computeIfAbsent(book.getTitle().toLowerCase(), k -> new ArrayList<>()).add(book);
        authorIndex.computeIfAbsent(book.getAuthor().toLowerCase(), k -> new ArrayList<>()).add(book);
        categoryIndex.computeIfAbsent(book.getCategory(), k -> new ArrayList<>()).add(book);
    }

    public List<Book> searchByTitle(String title) {
        return titleIndex.getOrDefault(title.toLowerCase(), Collections.emptyList());
    }

    public List<Book> searchByAuthor(String author) {
        return authorIndex.getOrDefault(author.toLowerCase(), Collections.emptyList());
    }

    public Book searchByIsbn(String isbn) {
        return isbnIndex.get(isbn);
    }

    public List<Book> searchByCategory(BookCategory category) {
        return categoryIndex.getOrDefault(category, Collections.emptyList());
    }
}
```

### Library (Main Controller)

```java
public class Library {
    private String name;
    private List<Book> books;
    private List<Member> members;
    private SearchService searchService;
    private NotificationService notificationService;

    public Library(String name) {
        this.name = name;
        this.books = new ArrayList<>();
        this.members = new ArrayList<>();
        this.searchService = new SearchService();
        this.notificationService = new NotificationService();
    }

    public void addBook(Book book) {
        books.add(book);
        searchService.addBook(book);
    }

    public void registerMember(Member member) {
        members.add(member);
    }

    public BookLending issueBook(Member member, String isbn) {
        Book book = searchService.searchByIsbn(isbn);
        if (book == null) throw new BookNotFoundException("Book not found");

        BookItem copy = book.getAvailableCopy();
        if (copy == null) throw new BookNotAvailableException("No copies available");

        BookLending lending = member.borrowBook(copy);
        notificationService.sendNotification(member,
            "Book issued: " + book.getTitle() + ", Due: " + lending.getDueDate());
        return lending;
    }

    public Fine returnBook(Member member, BookItem bookItem) {
        Fine fine = member.returnBook(bookItem);
        if (fine != null) {
            notificationService.sendNotification(member,
                "Overdue fine: ₹" + fine.getAmount());
        }
        return fine;
    }
}
```

---

## 🧪 Usage Example

```java
Library library = new Library("City Central Library");

// Add books
Book book = new Book("978-0-13-468599-1", "Clean Code", "Robert Martin", BookCategory.TECHNOLOGY);
book.addCopy(new BookItem("BC001", book, new Rack(1, "A")));
book.addCopy(new BookItem("BC002", book, new Rack(1, "A")));
library.addBook(book);

// Register member
Member member = new Member("M001", "Ritesh", "ritesh@email.com");
library.registerMember(member);

// Borrow book
BookLending lending = library.issueBook(member, "978-0-13-468599-1");

// Return book
Fine fine = library.returnBook(member, lending.getBookItem());
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Observer** | Notifications on due dates, reservation availability |
| **Strategy** | Search strategies, Fine calculation strategies |
| **Singleton** | Library instance (optional) |
| **Repository** | SearchService acts as a repository with indexes |

---

## ⚠️ Edge Cases

- Member tries to borrow 6th book → reject
- Book already borrowed → offer reservation
- Return after due date → calculate fine
- Member account blocked → cannot borrow
- All copies lost → update book status
- Concurrent checkout of last copy → synchronize

---

## 🔑 Key Takeaways

1. **Book vs BookItem** separation — Book is metadata, BookItem is physical copy
2. Search indexes (HashMap) make lookups O(1)
3. **Fine calculation** based on overdue days
4. **Reservation queue** notifies next member when book is returned
5. **Max borrow limit** enforced at Member level
