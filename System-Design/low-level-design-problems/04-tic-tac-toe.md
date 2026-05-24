# 04 - Tic-Tac-Toe Game

## 📋 Problem Statement

Design a Tic-Tac-Toe game that:
- Supports a configurable N×N board
- Handles two players with different symbols
- Detects win, draw, and invalid moves
- Can be extended for different game modes

---

## 📌 Requirements

### Functional Requirements
1. Two players take turns to place their symbol (X or O)
2. Board is **N×N** (default 3×3)
3. Detect **win** (row, column, diagonal) after each move
4. Detect **draw** when board is full
5. Validate moves (cell must be empty, within bounds)
6. Support **undo** last move (optional)

### Non-Functional Requirements
- Clean separation of concerns
- Extensible for AI player
- O(1) win detection (optimized approach)

---

## 🧩 Core Entities

```
Game, Board, Player, Cell, Symbol, GameStatus
```

---

## 📐 Class Diagram

```
┌────────────────────────────────────────────────┐
│                    Game                         │
│  - board: Board                                 │
│  - players: List<Player>                        │
│  - currentPlayerIndex: int                      │
│  - status: GameStatus                           │
│  + play(row, col): GameStatus                   │
│  + switchPlayer(): void                         │
│  + undo(): void                                 │
└────────────────┬───────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │                         │
┌───▼──────────────┐  ┌──────▼───────────────────┐
│     Board         │  │      Player              │
│  - grid: Cell[][] │  │  - name: String          │
│  - size: int      │  │  - symbol: Symbol        │
│  + makeMove()     │  │                          │
│  + checkWin()     │  │                          │
│  + isFull()       │  └──────────────────────────┘
│  + reset()        │
└──────────────────┘

┌──────────────────┐    ┌──────────────────┐
│     Cell          │    │   GameStatus     │
│  - row: int       │    │  ─ ─ ─ ─ ─ ─    │
│  - col: int       │    │  IN_PROGRESS     │
│  - symbol: Symbol │    │  WIN             │
│                   │    │  DRAW            │
└──────────────────┘    └──────────────────┘
```

---

## 🔧 Enums

```java
public enum Symbol {
    X, O, EMPTY
}

public enum GameStatus {
    IN_PROGRESS, WIN, DRAW
}
```

---

## 💻 Code Implementation

### Cell Class

```java
public class Cell {
    private int row;
    private int col;
    private Symbol symbol;

    public Cell(int row, int col) {
        this.row = row;
        this.col = col;
        this.symbol = Symbol.EMPTY;
    }

    public boolean isEmpty() {
        return symbol == Symbol.EMPTY;
    }

    public void setSymbol(Symbol symbol) {
        this.symbol = symbol;
    }

    public Symbol getSymbol() { return symbol; }
    public int getRow() { return row; }
    public int getCol() { return col; }
}
```

### Player Class

```java
public class Player {
    private String name;
    private Symbol symbol;

    public Player(String name, Symbol symbol) {
        this.name = name;
        this.symbol = symbol;
    }

    public String getName() { return name; }
    public Symbol getSymbol() { return symbol; }
}
```

### Board Class

```java
public class Board {
    private Cell[][] grid;
    private int size;
    private int movesCount;

    public Board(int size) {
        this.size = size;
        this.movesCount = 0;
        this.grid = new Cell[size][size];
        initializeBoard();
    }

    private void initializeBoard() {
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                grid[i][j] = new Cell(i, j);
            }
        }
    }

    public boolean makeMove(int row, int col, Symbol symbol) {
        if (row < 0 || row >= size || col < 0 || col >= size) {
            throw new InvalidMoveException("Move out of bounds");
        }
        if (!grid[row][col].isEmpty()) {
            throw new InvalidMoveException("Cell already occupied");
        }

        grid[row][col].setSymbol(symbol);
        movesCount++;
        return true;
    }

    public void undoMove(int row, int col) {
        grid[row][col].setSymbol(Symbol.EMPTY);
        movesCount--;
    }

    public boolean checkWin(int row, int col, Symbol symbol) {
        // Check row
        boolean rowWin = true;
        for (int j = 0; j < size; j++) {
            if (grid[row][j].getSymbol() != symbol) {
                rowWin = false;
                break;
            }
        }
        if (rowWin) return true;

        // Check column
        boolean colWin = true;
        for (int i = 0; i < size; i++) {
            if (grid[i][col].getSymbol() != symbol) {
                colWin = false;
                break;
            }
        }
        if (colWin) return true;

        // Check main diagonal
        if (row == col) {
            boolean diagWin = true;
            for (int i = 0; i < size; i++) {
                if (grid[i][i].getSymbol() != symbol) {
                    diagWin = false;
                    break;
                }
            }
            if (diagWin) return true;
        }

        // Check anti-diagonal
        if (row + col == size - 1) {
            boolean antiDiagWin = true;
            for (int i = 0; i < size; i++) {
                if (grid[i][size - 1 - i].getSymbol() != symbol) {
                    antiDiagWin = false;
                    break;
                }
            }
            if (antiDiagWin) return true;
        }

        return false;
    }

    public boolean isFull() {
        return movesCount == size * size;
    }

    public void printBoard() {
        for (int i = 0; i < size; i++) {
            for (int j = 0; j < size; j++) {
                Symbol s = grid[i][j].getSymbol();
                System.out.print(s == Symbol.EMPTY ? " . " : " " + s + " ");
                if (j < size - 1) System.out.print("|");
            }
            System.out.println();
            if (i < size - 1) {
                System.out.println("-".repeat(size * 4 - 1));
            }
        }
    }

    public int getSize() { return size; }
}
```

### Optimized Win Check (O(1) per move)

```java
public class OptimizedBoard extends Board {
    private int[] rowCounts;    // sum per row
    private int[] colCounts;    // sum per column
    private int diagCount;       // main diagonal sum
    private int antiDiagCount;   // anti-diagonal sum
    private int size;

    // Map X → +1, O → -1
    // If any counter reaches +N or -N → that player wins

    public OptimizedBoard(int size) {
        super(size);
        this.size = size;
        this.rowCounts = new int[size];
        this.colCounts = new int[size];
        this.diagCount = 0;
        this.antiDiagCount = 0;
    }

    @Override
    public boolean checkWin(int row, int col, Symbol symbol) {
        int value = (symbol == Symbol.X) ? 1 : -1;

        rowCounts[row] += value;
        colCounts[col] += value;

        if (row == col) diagCount += value;
        if (row + col == size - 1) antiDiagCount += value;

        return Math.abs(rowCounts[row]) == size
            || Math.abs(colCounts[col]) == size
            || Math.abs(diagCount) == size
            || Math.abs(antiDiagCount) == size;
    }
}
```

### Game Controller

```java
public class Game {
    private Board board;
    private List<Player> players;
    private int currentPlayerIndex;
    private GameStatus status;
    private Deque<int[]> moveHistory;

    public Game(int boardSize, Player player1, Player player2) {
        this.board = new Board(boardSize);
        this.players = List.of(player1, player2);
        this.currentPlayerIndex = 0;
        this.status = GameStatus.IN_PROGRESS;
        this.moveHistory = new ArrayDeque<>();
    }

    public GameStatus play(int row, int col) {
        if (status != GameStatus.IN_PROGRESS) {
            throw new GameOverException("Game is already over");
        }

        Player currentPlayer = players.get(currentPlayerIndex);
        board.makeMove(row, col, currentPlayer.getSymbol());
        moveHistory.push(new int[]{row, col});

        board.printBoard();

        if (board.checkWin(row, col, currentPlayer.getSymbol())) {
            status = GameStatus.WIN;
            System.out.println(currentPlayer.getName() + " wins!");
            return status;
        }

        if (board.isFull()) {
            status = GameStatus.DRAW;
            System.out.println("It's a draw!");
            return status;
        }

        switchPlayer();
        return GameStatus.IN_PROGRESS;
    }

    public void undo() {
        if (moveHistory.isEmpty()) return;
        int[] lastMove = moveHistory.pop();
        board.undoMove(lastMove[0], lastMove[1]);
        switchPlayer();
    }

    private void switchPlayer() {
        currentPlayerIndex = (currentPlayerIndex + 1) % players.size();
    }

    public Player getCurrentPlayer() {
        return players.get(currentPlayerIndex);
    }
}
```

---

## 🧪 Usage Example

```java
Player p1 = new Player("Alice", Symbol.X);
Player p2 = new Player("Bob", Symbol.O);
Game game = new Game(3, p1, p2);

game.play(0, 0); // Alice: X at (0,0)
game.play(1, 1); // Bob: O at (1,1)
game.play(0, 1); // Alice: X at (0,1)
game.play(1, 0); // Bob: O at (1,0)
game.play(0, 2); // Alice: X at (0,2) → Alice wins! (row 0)
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Strategy** | Can plug different win-check strategies or AI strategies |
| **Command** | Undo support via move history |
| **Factory** | Create players (Human vs AI) |

---

## ⚠️ Edge Cases

- Player tries occupied cell → throw InvalidMoveException
- Move out of bounds → throw InvalidMoveException
- Game already over, player tries to play → throw GameOverException
- Undo on empty history → no-op

---

## 🔑 Key Takeaways

1. **Optimized win check** using row/col counters gives O(1) per move vs O(N)
2. **Move history stack** enables undo
3. Board size is **configurable** (not hardcoded to 3)
4. Separation of Board logic from Game controller
