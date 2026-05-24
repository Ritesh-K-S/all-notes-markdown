# 26 - Deck of Cards / Card Game (Blackjack)

## 📋 Problem Statement

Design a generic deck of cards system that can be extended for any card game, with Blackjack as a concrete implementation.

---

## 📌 Requirements

### Functional Requirements
1. Standard deck of **52 cards** (4 suits × 13 ranks)
2. **Shuffle** the deck
3. **Deal** cards to players
4. Support **multiple decks** (casino-style)
5. Implement a **Blackjack** game:
   - Player vs Dealer
   - Hit, Stand, Double Down
   - Bust if total > 21
   - Ace counts as 1 or 11
   - Dealer hits until 17

### Non-Functional Requirements
- Extensible for other card games (Poker, Rummy)
- Clean OOP with inheritance

---

## 🧩 Core Entities

```
Card, Suit, Rank, Deck, Hand, Player, Dealer, Game, GameAction
```

---

## 📐 Class Diagram

```
┌──────────────────┐     ┌──────────────────┐
│      Card         │     │    Suit (Enum)    │
│  - suit: Suit     │     │  HEARTS          │
│  - rank: Rank     │     │  DIAMONDS        │
│  - faceUp: bool   │     │  CLUBS           │
│  + getValue(): int│     │  SPADES          │
│  + flip(): void   │     └──────────────────┘
└──────────────────┘
                          ┌──────────────────┐
┌──────────────────┐      │   Rank (Enum)    │
│      Deck         │     │  ACE, TWO...TEN  │
│  - cards: List    │     │  JACK, QUEEN,    │
│  + shuffle(): void│     │  KING            │
│  + deal(): Card   │     └──────────────────┘
│  + isEmpty(): bool│
└──────────────────┘

┌──────────────────────────────────────────┐
│                  Hand                     │
│  - cards: List<Card>                      │
│  + addCard(card): void                    │
│  + getScore(): int  ← game-specific      │
│  + getCards(): List<Card>                 │
└──────────────────────────────────────────┘

┌──────────────────┐     ┌──────────────────┐
│    Player         │     │    Dealer         │
│  - name: String   │     │  (extends Player) │
│  - hand: Hand     │     │  + shouldHit()    │
│  - balance: double│     │    : boolean      │
│  + hit(): void    │     └──────────────────┘
│  + stand(): void  │
└──────────────────┘
```

---

## 🔧 Enums

```java
public enum Suit {
    HEARTS("♥"), DIAMONDS("♦"), CLUBS("♣"), SPADES("♠");

    private final String symbol;
    Suit(String symbol) { this.symbol = symbol; }
    public String getSymbol() { return symbol; }
}

public enum Rank {
    ACE(1), TWO(2), THREE(3), FOUR(4), FIVE(5), SIX(6), SEVEN(7),
    EIGHT(8), NINE(9), TEN(10), JACK(10), QUEEN(10), KING(10);

    private final int value;
    Rank(int value) { this.value = value; }
    public int getValue() { return value; }
}
```

---

## 💻 Code Implementation

### Card Class

```java
public class Card {
    private Suit suit;
    private Rank rank;
    private boolean faceUp;

    public Card(Suit suit, Rank rank) {
        this.suit = suit;
        this.rank = rank;
        this.faceUp = false;
    }

    public void flip() { this.faceUp = !this.faceUp; }

    public int getValue() { return rank.getValue(); }
    public Suit getSuit() { return suit; }
    public Rank getRank() { return rank; }
    public boolean isFaceUp() { return faceUp; }

    @Override
    public String toString() {
        return faceUp ? rank + " of " + suit : "[Hidden]";
    }
}
```

### Deck Class

```java
public class Deck {
    private List<Card> cards;

    public Deck() {
        cards = new ArrayList<>();
        initialize();
    }

    public Deck(int numDecks) {
        cards = new ArrayList<>();
        for (int i = 0; i < numDecks; i++) {
            initialize();
        }
    }

    private void initialize() {
        for (Suit suit : Suit.values()) {
            for (Rank rank : Rank.values()) {
                cards.add(new Card(suit, rank));
            }
        }
    }

    public void shuffle() {
        Collections.shuffle(cards);
    }

    public Card deal() {
        if (cards.isEmpty()) throw new EmptyDeckException("No cards left in deck");
        Card card = cards.remove(cards.size() - 1);
        card.flip(); // face up when dealt
        return card;
    }

    public boolean isEmpty() { return cards.isEmpty(); }
    public int remainingCards() { return cards.size(); }
}
```

### Hand Class (Blackjack-specific scoring)

```java
public class BlackjackHand {
    private List<Card> cards;

    public BlackjackHand() {
        this.cards = new ArrayList<>();
    }

    public void addCard(Card card) {
        cards.add(card);
    }

    public int getScore() {
        int score = 0;
        int aceCount = 0;

        for (Card card : cards) {
            score += card.getValue();
            if (card.getRank() == Rank.ACE) aceCount++;
        }

        // Ace can be 11 instead of 1 (if it doesn't bust)
        while (aceCount > 0 && score + 10 <= 21) {
            score += 10;
            aceCount--;
        }

        return score;
    }

    public boolean isBust() { return getScore() > 21; }

    public boolean isBlackjack() {
        return cards.size() == 2 && getScore() == 21;
    }

    public List<Card> getCards() { return cards; }

    public void clear() { cards.clear(); }

    @Override
    public String toString() {
        return cards.toString() + " (Score: " + getScore() + ")";
    }
}
```

### Player and Dealer

```java
public class Player {
    private String name;
    private BlackjackHand hand;
    private double balance;
    private double currentBet;

    public Player(String name, double balance) {
        this.name = name;
        this.balance = balance;
        this.hand = new BlackjackHand();
    }

    public void placeBet(double amount) {
        if (amount > balance) throw new InsufficientBalanceException("Not enough balance");
        this.currentBet = amount;
        this.balance -= amount;
    }

    public void win(double amount) { this.balance += amount; }

    public void hit(Card card) { hand.addCard(card); }

    public void resetHand() { hand.clear(); }

    // Getters
    public String getName() { return name; }
    public BlackjackHand getHand() { return hand; }
    public double getBalance() { return balance; }
    public double getCurrentBet() { return currentBet; }
}

public class Dealer extends Player {
    public Dealer() {
        super("Dealer", Double.MAX_VALUE);
    }

    public boolean shouldHit() {
        return getHand().getScore() < 17;
    }
}
```

### Blackjack Game

```java
public class BlackjackGame {
    private Deck deck;
    private Player player;
    private Dealer dealer;
    private GameStatus status;

    public BlackjackGame(Player player) {
        this.deck = new Deck(6); // 6-deck shoe (casino standard)
        this.deck.shuffle();
        this.player = player;
        this.dealer = new Dealer();
        this.status = GameStatus.WAITING_FOR_BET;
    }

    public void startRound(double bet) {
        player.resetHand();
        dealer.resetHand();
        player.placeBet(bet);

        // Deal 2 cards each
        player.hit(deck.deal());
        dealer.hit(deck.deal());
        player.hit(deck.deal());
        Card dealerHidden = deck.deal();
        dealerHidden.flip(); // face down
        dealer.hit(dealerHidden);

        status = GameStatus.PLAYER_TURN;

        // Check natural blackjack
        if (player.getHand().isBlackjack()) {
            status = GameStatus.DEALER_TURN;
            playDealerTurn();
        }

        System.out.println("Player: " + player.getHand());
        System.out.println("Dealer: " + dealer.getHand().getCards().get(0) + " [Hidden]");
    }

    public void playerHit() {
        if (status != GameStatus.PLAYER_TURN) return;

        player.hit(deck.deal());
        System.out.println("Player: " + player.getHand());

        if (player.getHand().isBust()) {
            System.out.println("Player BUSTS!");
            status = GameStatus.FINISHED;
            resolveGame();
        }
    }

    public void playerStand() {
        if (status != GameStatus.PLAYER_TURN) return;
        status = GameStatus.DEALER_TURN;
        playDealerTurn();
    }

    private void playDealerTurn() {
        // Reveal hidden card
        for (Card card : dealer.getHand().getCards()) {
            if (!card.isFaceUp()) card.flip();
        }

        // Dealer hits until 17
        while (dealer.shouldHit()) {
            dealer.hit(deck.deal());
        }

        System.out.println("Dealer: " + dealer.getHand());
        status = GameStatus.FINISHED;
        resolveGame();
    }

    private void resolveGame() {
        int playerScore = player.getHand().getScore();
        int dealerScore = dealer.getHand().getScore();

        if (player.getHand().isBust()) {
            System.out.println("Dealer wins! Player busted.");
        } else if (dealer.getHand().isBust()) {
            System.out.println("Player wins! Dealer busted.");
            player.win(player.getCurrentBet() * 2);
        } else if (player.getHand().isBlackjack() && !dealer.getHand().isBlackjack()) {
            System.out.println("BLACKJACK! Player wins 3:2.");
            player.win(player.getCurrentBet() * 2.5);
        } else if (playerScore > dealerScore) {
            System.out.println("Player wins!");
            player.win(player.getCurrentBet() * 2);
        } else if (playerScore < dealerScore) {
            System.out.println("Dealer wins!");
        } else {
            System.out.println("Push! (Tie)");
            player.win(player.getCurrentBet());
        }
    }
}

public enum GameStatus {
    WAITING_FOR_BET, PLAYER_TURN, DEALER_TURN, FINISHED
}
```

---

## 🧪 Usage Example

```java
Player player = new Player("Ritesh", 10000);
BlackjackGame game = new BlackjackGame(player);

game.startRound(100);
// Player: [ACE of HEARTS, SEVEN of CLUBS] (Score: 18)
// Dealer: KING of DIAMONDS [Hidden]

game.playerStand();
// Dealer: [KING of DIAMONDS, SIX of SPADES, THREE of HEARTS] (Score: 19)
// Dealer wins!
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Inheritance** | Dealer extends Player |
| **Polymorphism** | Hand scoring can vary per game |
| **Factory** | Deck creates cards |
| **Strategy** | Dealer strategy (hit below 17), extendable for player AI |
| **Template Method** | Game loop: deal → player turn → dealer turn → resolve |

---

## 🔑 Key Takeaways

1. **Ace handling** is the tricky part — can be 1 or 11
2. **Deck** is decoupled from game — reusable for Poker, Rummy, etc.
3. **Dealer logic** (`shouldHit()`) is simple but separate from Player
4. **Multiple decks** via constructor parameter (casino shoe)
5. Classic **OOP design** problem — focus on clean class hierarchy
