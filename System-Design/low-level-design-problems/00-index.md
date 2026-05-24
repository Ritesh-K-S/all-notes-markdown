# Low-Level Design Problems - Index

> A comprehensive collection of classic LLD/OOD problems with detailed design breakdowns.

---

## Problem List

| # | Problem | Key Concepts |
|---|---------|-------------|
| 01 | [Parking Lot System](./01-parking-lot-system.md) | Inheritance, Strategy Pattern, Singleton |
| 02 | [Elevator System](./02-elevator-system.md) | State Pattern, Strategy Pattern, Observer |
| 03 | [Library Management System](./03-library-management-system.md) | CRUD, Search, Observer Pattern |
| 04 | [Tic-Tac-Toe Game](./04-tic-tac-toe.md) | Factory Pattern, Strategy Pattern |
| 05 | [Chess Game](./05-chess-game.md) | Inheritance, Polymorphism, Command Pattern |
| 06 | [BookMyShow / Movie Ticket Booking](./06-bookmyshow.md) | Concurrency, Locking, Observer |
| 07 | [Snake and Ladder Game](./07-snake-and-ladder.md) | Composition, Random, State |
| 08 | [ATM Machine](./08-atm-machine.md) | State Pattern, Chain of Responsibility |
| 09 | [Hotel Management System](./09-hotel-management-system.md) | Strategy, Observer, CRUD |
| 10 | [Vending Machine](./10-vending-machine.md) | State Pattern, Singleton |
| 11 | [Social Media (Twitter/Instagram)](./11-social-media.md) | Observer, Feed Algorithm, Decorator |
| 12 | [Splitwise / Expense Sharing](./12-splitwise.md) | Strategy Pattern, Graph, Observer |
| 13 | [Online Shopping (Amazon)](./13-online-shopping-amazon.md) | Factory, Strategy, Observer, Cart |
| 14 | [Stack Overflow](./14-stack-overflow.md) | Composite, Observer, Decorator |
| 15 | [Cab Booking (Uber/Ola)](./15-cab-booking-uber.md) | Strategy, Observer, State |
| 16 | [Food Delivery (Zomato/Swiggy)](./16-food-delivery-system.md) | Strategy, Observer, State |
| 17 | [Logging Framework](./17-logging-framework.md) | Singleton, Observer, Strategy, Chain of Responsibility |
| 18 | [Rate Limiter](./18-rate-limiter.md) | Strategy, Sliding Window, Token Bucket |
| 19 | [Cache System (LRU/LFU)](./19-cache-system.md) | Strategy, Doubly Linked List, HashMap |
| 20 | [File System](./20-file-system.md) | Composite Pattern, Iterator |
| 21 | [Payment System](./21-payment-system.md) | Strategy, Factory, Observer |
| 22 | [Notification System](./22-notification-system.md) | Observer, Factory, Decorator |
| 23 | [Task Scheduler / Cron](./23-task-scheduler.md) | Priority Queue, Command Pattern |
| 24 | [Pub-Sub Messaging System](./24-pub-sub-messaging.md) | Observer, Queue, Strategy |
| 25 | [Design HashMap](./25-design-hashmap.md) | Hashing, Chaining, Resizing |
| 26 | [Deck of Cards / Blackjack](./26-deck-of-cards.md) | Inheritance, Polymorphism, Shuffle |
| 27 | [Online Chat System (WhatsApp)](./27-online-chat-system.md) | Observer, Mediator, Strategy |
| 28 | [Meeting / Calendar Scheduler](./28-meeting-scheduler.md) | Interval Overlap, Observer, Strategy |
| 29 | [Car Rental System](./29-car-rental-system.md) | State, Strategy, Factory |
| 30 | [Cricbuzz / Sports Scoreboard](./30-cricbuzz-scoreboard.md) | Composite, Observer, Strategy |
| 31 | [Airline / Flight Booking System](./31-airline-booking-system.md) | State, Strategy, Two-Phase Locking |

---

## How to Approach LLD Problems

```
1. Clarify Requirements → Functional + Non-Functional
2. Identify Core Entities → Nouns in the problem
3. Define Relationships → IS-A, HAS-A, USES
4. Draw Class Diagram → UML
5. Apply Design Patterns → Where applicable
6. Write Code → Skeleton with key methods
7. Handle Edge Cases → Concurrency, Validation
```

---

## Common Patterns in LLD

| Pattern | Used In |
|---------|---------|
| **Singleton** | Parking Lot, Vending Machine, Logger |
| **Strategy** | Payment, Pricing, Cab Assignment |
| **Observer** | Notifications, Booking, Social Media |
| **State** | Elevator, Vending Machine, ATM |
| **Factory** | Vehicle, Payment, User types |
| **Command** | Chess moves, Undo operations |
| **Chain of Responsibility** | ATM dispensing, Logging levels |
| **Composite** | File System, Menus |
| **Decorator** | Toppings, Notifications |
| **Builder** | Complex object creation |
