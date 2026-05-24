# 07 - Snake and Ladder Game

## 📋 Problem Statement

Design a Snake and Ladder board game that:
- Supports multiple players
- Has configurable board size, snakes, and ladders
- Handles dice rolling and player movement

---

## 📌 Requirements

### Functional Requirements
1. Board of **N×N** cells (default 100 cells: 10×10)
2. Configurable **snakes** (head → tail, moves DOWN)
3. Configurable **ladders** (bottom → top, moves UP)
4. **Multiple players** take turns rolling a dice
5. Dice roll: **1 to 6** (can support multiple dice)
6. Player reaching **exactly 100** wins
7. If roll exceeds 100, player stays in position
8. No snake head and ladder bottom on same cell

### Non-Functional Requirements
- Extensible for special dice, power-ups, etc.
- Fair randomization

---

## 🧩 Core Entities

```
Game, Board, Cell, Snake, Ladder, Player, Dice
```

---

## 📐 Class Diagram

```
┌────────────────────────────────────────┐
│                 Game                    │
│  - board: Board                         │
│  - players: Queue<Player>               │
│  - dice: Dice                           │
│  - winner: Player                       │
│  + start(): void                        │
│  + playTurn(): boolean                  │
└───────────────┬────────────────────────┘
                │
       ┌────────┼────────┐
       │                 │
┌──────▼──────┐  ┌───────▼──────┐
│    Board     │  │    Dice       │
│  - size: int │  │  - count: int │
│  - snakes[]  │  │  + roll(): int│
│  - ladders[] │  │               │
│  + getNewPos │  └──────────────┘
└──────┬───────┘
       │
┌──────▼──────────────────────────┐
│            Cell                  │
│  - position: int                 │
│  - jump: int (0 if normal cell)  │
│  → positive = ladder             │
│  → negative = snake              │
└─────────────────────────────────┘

┌──────────────────┐    ┌──────────────────┐
│     Snake          │    │     Ladder         │
│  - head: int       │    │  - bottom: int     │
│  - tail: int       │    │  - top: int        │
│  (head > tail)     │    │  (top > bottom)    │
└──────────────────┘    └──────────────────┘
```

---

## 💻 Code Implementation

### Dice Class

```java
public class Dice {
    private int count; // number of dice
    private Random random;

    public Dice(int count) {
        this.count = count;
        this.random = new Random();
    }

    public int roll() {
        int total = 0;
        for (int i = 0; i < count; i++) {
            total += random.nextInt(6) + 1;
        }
        return total;
    }
}
```

### Snake and Ladder

```java
public class Snake {
    private int head;
    private int tail;

    public Snake(int head, int tail) {
        if (head <= tail) throw new IllegalArgumentException("Snake head must be above tail");
        this.head = head;
        this.tail = tail;
    }

    public int getHead() { return head; }
    public int getTail() { return tail; }
}

public class Ladder {
    private int bottom;
    private int top;

    public Ladder(int bottom, int top) {
        if (top <= bottom) throw new IllegalArgumentException("Ladder top must be above bottom");
        this.bottom = bottom;
        this.top = top;
    }

    public int getBottom() { return bottom; }
    public int getTop() { return top; }
}
```

### Player Class

```java
public class Player {
    private String name;
    private int position;

    public Player(String name) {
        this.name = name;
        this.position = 0; // starts outside the board
    }

    public String getName() { return name; }
    public int getPosition() { return position; }
    public void setPosition(int position) { this.position = position; }
}
```

### Board Class

```java
public class Board {
    private int size;
    private Map<Integer, Integer> snakeMap;   // head → tail
    private Map<Integer, Integer> ladderMap;  // bottom → top

    public Board(int size, List<Snake> snakes, List<Ladder> ladders) {
        this.size = size;

        this.snakeMap = new HashMap<>();
        for (Snake s : snakes) {
            snakeMap.put(s.getHead(), s.getTail());
        }

        this.ladderMap = new HashMap<>();
        for (Ladder l : ladders) {
            if (snakeMap.containsKey(l.getBottom())) {
                throw new IllegalArgumentException("Snake and Ladder cannot start at same cell");
            }
            ladderMap.put(l.getBottom(), l.getTop());
        }
    }

    public int getFinalPosition(int position) {
        if (snakeMap.containsKey(position)) {
            System.out.println("  🐍 Snake! " + position + " → " + snakeMap.get(position));
            return snakeMap.get(position);
        }
        if (ladderMap.containsKey(position)) {
            System.out.println("  🪜 Ladder! " + position + " → " + ladderMap.get(position));
            return ladderMap.get(position);
        }
        return position;
    }

    public int getSize() { return size; }
}
```

### Game Controller

```java
public class Game {
    private Board board;
    private Queue<Player> players;
    private Dice dice;
    private Player winner;

    public Game(Board board, List<Player> playerList, int diceCount) {
        this.board = board;
        this.dice = new Dice(diceCount);
        this.players = new LinkedList<>(playerList);
        this.winner = null;
    }

    public void start() {
        System.out.println("=== Game Started ===");
        while (winner == null) {
            playTurn();
        }
        System.out.println("🏆 " + winner.getName() + " wins!");
    }

    private void playTurn() {
        Player current = players.poll();
        int diceValue = dice.roll();
        int newPosition = current.getPosition() + diceValue;

        System.out.println(current.getName() + " rolled " + diceValue
            + " | Position: " + current.getPosition() + " → " + newPosition);

        if (newPosition > board.getSize()) {
            // Cannot move, stay in current position
            System.out.println("  Exceeds board! Staying at " + current.getPosition());
        } else if (newPosition == board.getSize()) {
            // Winner!
            current.setPosition(newPosition);
            winner = current;
            return;
        } else {
            // Check for snake or ladder
            newPosition = board.getFinalPosition(newPosition);
            current.setPosition(newPosition);
        }

        players.offer(current); // back to queue
    }

    public Player getWinner() { return winner; }
}
```

---

## 🧪 Usage Example

```java
// Setup snakes
List<Snake> snakes = List.of(
    new Snake(99, 10),
    new Snake(65, 25),
    new Snake(45, 6),
    new Snake(52, 11)
);

// Setup ladders
List<Ladder> ladders = List.of(
    new Ladder(2, 38),
    new Ladder(7, 14),
    new Ladder(28, 84),
    new Ladder(51, 67)
);

Board board = new Board(100, snakes, ladders);

List<Player> players = List.of(
    new Player("Alice"),
    new Player("Bob"),
    new Player("Charlie")
);

Game game = new Game(board, players, 1);
game.start();
```

**Sample Output:**
```
=== Game Started ===
Alice rolled 4 | Position: 0 → 4
Bob rolled 2 | Position: 0 → 2
  🪜 Ladder! 2 → 38
Charlie rolled 6 | Position: 0 → 6
Alice rolled 3 | Position: 4 → 7
  🪜 Ladder! 7 → 14
...
🏆 Bob wins!
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Composition** | Board HAS snakes and ladders |
| **Strategy** | Can swap dice strategy (normal, weighted, etc.) |
| **Iterator** | Queue-based turn management |

---

## ⚠️ Edge Cases

- Dice roll exceeds board → stay in place
- Snake at position 100 → not allowed (100 is winning)
- Ladder at position 1 → allowed
- Multiple snakes/ladders in chain → apply only once per turn
- All players stuck → game continues until someone wins

---

## 🔑 Key Takeaways

1. **HashMap** for snake/ladder mapping gives O(1) lookups
2. **Queue** for turn management — natural round-robin
3. Board is **configurable** — snakes, ladders, size all injected
4. Simple but effective — great entry-level LLD problem
5. Can extend with: multiple dice, special power-ups, blocked cells
