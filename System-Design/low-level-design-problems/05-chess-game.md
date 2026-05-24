# 05 - Chess Game

## 📋 Problem Statement

Design a two-player Chess game that:
- Supports all standard chess pieces and their movements
- Validates legal moves including special moves (castling, en passant, pawn promotion)
- Detects check, checkmate, and stalemate

---

## 📌 Requirements

### Functional Requirements
1. Standard **8×8 board** with 16 pieces per player
2. All piece types: **King, Queen, Rook, Bishop, Knight, Pawn**
3. Validate **legal moves** for each piece
4. Detect **check, checkmate, stalemate**
5. Support **castling, en passant, pawn promotion**
6. Two players alternate turns (White starts)
7. **Move history** and undo

### Non-Functional Requirements
- Extensible for new game modes (e.g., Chess960)
- Clean OOP with polymorphism for piece movement

---

## 🧩 Core Entities

```
Game, Board, Cell, Piece (abstract), King, Queen, Rook, Bishop, 
Knight, Pawn, Player, Move, Color, GameStatus
```

---

## 📐 Class Diagram

```
┌────────────────────────────────────────────────────────────┐
│                         Game                               │
│  - board: Board                                            │
│  - players: Player[2]                                      │
│  - currentTurn: Color                                      │
│  - status: GameStatus                                      │
│  - moveHistory: List<Move>                                 │
│  + makeMove(start, end): boolean                           │
│  + isCheckmate(): boolean                                  │
│  + isStalemate(): boolean                                  │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                        Board                               │
│  - cells: Cell[8][8]                                       │
│  + getPiece(row, col): Piece                               │
│  + movePiece(start, end): void                             │
│  + isValidPosition(row, col): boolean                      │
└────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────┐
│                     Piece (abstract)                       │
│  - color: Color                                            │
│  - position: Cell                                          │
│  - killed: boolean                                         │
│  + canMove(board, start, end): boolean  ← abstract         │
│  + getPossibleMoves(board): List<Cell>  ← abstract         │
└───────────────┬────────────────────────────────────────────┘
                │
   ┌────────┬───┴────┬─────────┬──────────┬─────────┐
   │        │        │         │          │         │
 King    Queen    Rook     Bishop     Knight     Pawn
```

---

## 🔧 Enums

```java
public enum Color {
    WHITE, BLACK
}

public enum GameStatus {
    ACTIVE, WHITE_WIN, BLACK_WIN, STALEMATE, DRAW, RESIGNED
}
```

---

## 💻 Code Implementation

### Cell Class

```java
public class Cell {
    private int row;
    private int col;
    private Piece piece;

    public Cell(int row, int col) {
        this.row = row;
        this.col = col;
        this.piece = null;
    }

    public boolean isOccupied() {
        return piece != null;
    }

    // Getters and setters
    public Piece getPiece() { return piece; }
    public void setPiece(Piece piece) { this.piece = piece; }
    public int getRow() { return row; }
    public int getCol() { return col; }
}
```

### Abstract Piece and Concrete Pieces

```java
public abstract class Piece {
    protected Color color;
    protected boolean killed;

    public Piece(Color color) {
        this.color = color;
        this.killed = false;
    }

    public abstract boolean canMove(Board board, Cell start, Cell end);
    public abstract List<Cell> getPossibleMoves(Board board, Cell current);

    public Color getColor() { return color; }
    public boolean isKilled() { return killed; }
    public void setKilled(boolean killed) { this.killed = killed; }
}

// ─── KING ───────────────────────────────────
public class King extends Piece {
    private boolean hasMoved = false;

    public King(Color color) { super(color); }

    @Override
    public boolean canMove(Board board, Cell start, Cell end) {
        int rowDiff = Math.abs(start.getRow() - end.getRow());
        int colDiff = Math.abs(start.getCol() - end.getCol());

        // King moves one square in any direction
        if (rowDiff > 1 || colDiff > 1) return false;
        if (rowDiff == 0 && colDiff == 0) return false;

        // Cannot capture own piece
        if (end.isOccupied() && end.getPiece().getColor() == this.color) return false;

        return true;
    }

    @Override
    public List<Cell> getPossibleMoves(Board board, Cell current) {
        List<Cell> moves = new ArrayList<>();
        int[][] directions = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}};

        for (int[] dir : directions) {
            int newRow = current.getRow() + dir[0];
            int newCol = current.getCol() + dir[1];
            if (board.isValidPosition(newRow, newCol)) {
                Cell target = board.getCell(newRow, newCol);
                if (!target.isOccupied() || target.getPiece().getColor() != this.color) {
                    moves.add(target);
                }
            }
        }
        return moves;
    }
}

// ─── QUEEN ──────────────────────────────────
public class Queen extends Piece {
    public Queen(Color color) { super(color); }

    @Override
    public boolean canMove(Board board, Cell start, Cell end) {
        // Queen = Rook + Bishop (straight + diagonal)
        Rook rook = new Rook(this.color);
        Bishop bishop = new Bishop(this.color);
        return rook.canMove(board, start, end) || bishop.canMove(board, start, end);
    }
}

// ─── ROOK ───────────────────────────────────
public class Rook extends Piece {
    private boolean hasMoved = false;

    public Rook(Color color) { super(color); }

    @Override
    public boolean canMove(Board board, Cell start, Cell end) {
        int rowDiff = end.getRow() - start.getRow();
        int colDiff = end.getCol() - start.getCol();

        // Must move in straight line
        if (rowDiff != 0 && colDiff != 0) return false;

        // Check path is clear
        return isPathClear(board, start, end);
    }

    private boolean isPathClear(Board board, Cell start, Cell end) {
        int rowDir = Integer.signum(end.getRow() - start.getRow());
        int colDir = Integer.signum(end.getCol() - start.getCol());

        int r = start.getRow() + rowDir;
        int c = start.getCol() + colDir;

        while (r != end.getRow() || c != end.getCol()) {
            if (board.getCell(r, c).isOccupied()) return false;
            r += rowDir;
            c += colDir;
        }

        // Destination: empty or enemy piece
        return !end.isOccupied() || end.getPiece().getColor() != this.color;
    }
}

// ─── BISHOP ─────────────────────────────────
public class Bishop extends Piece {
    public Bishop(Color color) { super(color); }

    @Override
    public boolean canMove(Board board, Cell start, Cell end) {
        int rowDiff = Math.abs(end.getRow() - start.getRow());
        int colDiff = Math.abs(end.getCol() - start.getCol());

        // Must move diagonally
        if (rowDiff != colDiff || rowDiff == 0) return false;

        return isPathClear(board, start, end);
    }

    private boolean isPathClear(Board board, Cell start, Cell end) {
        int rowDir = Integer.signum(end.getRow() - start.getRow());
        int colDir = Integer.signum(end.getCol() - start.getCol());

        int r = start.getRow() + rowDir;
        int c = start.getCol() + colDir;

        while (r != end.getRow()) {
            if (board.getCell(r, c).isOccupied()) return false;
            r += rowDir;
            c += colDir;
        }

        return !end.isOccupied() || end.getPiece().getColor() != this.color;
    }
}

// ─── KNIGHT ─────────────────────────────────
public class Knight extends Piece {
    public Knight(Color color) { super(color); }

    @Override
    public boolean canMove(Board board, Cell start, Cell end) {
        int rowDiff = Math.abs(end.getRow() - start.getRow());
        int colDiff = Math.abs(end.getCol() - start.getCol());

        // L-shape: (2,1) or (1,2)
        if (!((rowDiff == 2 && colDiff == 1) || (rowDiff == 1 && colDiff == 2))) {
            return false;
        }

        return !end.isOccupied() || end.getPiece().getColor() != this.color;
    }
}

// ─── PAWN ───────────────────────────────────
public class Pawn extends Piece {
    private boolean hasMoved = false;

    public Pawn(Color color) { super(color); }

    @Override
    public boolean canMove(Board board, Cell start, Cell end) {
        int direction = (color == Color.WHITE) ? -1 : 1; // White moves up, Black moves down
        int rowDiff = end.getRow() - start.getRow();
        int colDiff = Math.abs(end.getCol() - start.getCol());

        // Forward one step
        if (colDiff == 0 && rowDiff == direction && !end.isOccupied()) {
            return true;
        }

        // Forward two steps from starting position
        if (colDiff == 0 && rowDiff == 2 * direction && !hasMoved && !end.isOccupied()) {
            Cell between = board.getCell(start.getRow() + direction, start.getCol());
            return !between.isOccupied();
        }

        // Diagonal capture
        if (colDiff == 1 && rowDiff == direction && end.isOccupied()
            && end.getPiece().getColor() != this.color) {
            return true;
        }

        return false;
    }
}
```

### Board Class

```java
public class Board {
    private Cell[][] cells;

    public Board() {
        cells = new Cell[8][8];
        initializeBoard();
        setupPieces();
    }

    private void initializeBoard() {
        for (int i = 0; i < 8; i++)
            for (int j = 0; j < 8; j++)
                cells[i][j] = new Cell(i, j);
    }

    private void setupPieces() {
        // Black pieces (row 0-1)
        cells[0][0].setPiece(new Rook(Color.BLACK));
        cells[0][1].setPiece(new Knight(Color.BLACK));
        cells[0][2].setPiece(new Bishop(Color.BLACK));
        cells[0][3].setPiece(new Queen(Color.BLACK));
        cells[0][4].setPiece(new King(Color.BLACK));
        cells[0][5].setPiece(new Bishop(Color.BLACK));
        cells[0][6].setPiece(new Knight(Color.BLACK));
        cells[0][7].setPiece(new Rook(Color.BLACK));
        for (int j = 0; j < 8; j++) cells[1][j].setPiece(new Pawn(Color.BLACK));

        // White pieces (row 6-7)
        cells[7][0].setPiece(new Rook(Color.WHITE));
        cells[7][1].setPiece(new Knight(Color.WHITE));
        cells[7][2].setPiece(new Bishop(Color.WHITE));
        cells[7][3].setPiece(new Queen(Color.WHITE));
        cells[7][4].setPiece(new King(Color.WHITE));
        cells[7][5].setPiece(new Bishop(Color.WHITE));
        cells[7][6].setPiece(new Knight(Color.WHITE));
        cells[7][7].setPiece(new Rook(Color.WHITE));
        for (int j = 0; j < 8; j++) cells[6][j].setPiece(new Pawn(Color.WHITE));
    }

    public void movePiece(Cell start, Cell end) {
        end.setPiece(start.getPiece());
        start.setPiece(null);
    }

    public boolean isValidPosition(int row, int col) {
        return row >= 0 && row < 8 && col >= 0 && col < 8;
    }

    public Cell getCell(int row, int col) { return cells[row][col]; }
}
```

### Move Class

```java
public class Move {
    private Cell start;
    private Cell end;
    private Piece movedPiece;
    private Piece capturedPiece;
    private Player player;
    private LocalDateTime timestamp;

    public Move(Cell start, Cell end, Player player) {
        this.start = start;
        this.end = end;
        this.movedPiece = start.getPiece();
        this.capturedPiece = end.getPiece();
        this.player = player;
        this.timestamp = LocalDateTime.now();
    }
}
```

### Game Controller

```java
public class Game {
    private Board board;
    private Player[] players;
    private int currentTurnIndex;
    private GameStatus status;
    private List<Move> moveHistory;

    public Game(Player whitePlayer, Player blackPlayer) {
        this.board = new Board();
        this.players = new Player[]{whitePlayer, blackPlayer};
        this.currentTurnIndex = 0; // White starts
        this.status = GameStatus.ACTIVE;
        this.moveHistory = new ArrayList<>();
    }

    public boolean makeMove(int startRow, int startCol, int endRow, int endCol) {
        if (status != GameStatus.ACTIVE) return false;

        Player currentPlayer = players[currentTurnIndex];
        Cell start = board.getCell(startRow, startCol);
        Cell end = board.getCell(endRow, endCol);

        // Validate: piece exists and belongs to current player
        if (!start.isOccupied()) return false;
        Piece piece = start.getPiece();
        if (piece.getColor() != currentPlayer.getColor()) return false;

        // Validate: piece can make this move
        if (!piece.canMove(board, start, end)) return false;

        // Execute move
        Move move = new Move(start, end, currentPlayer);
        if (end.isOccupied()) {
            end.getPiece().setKilled(true);
        }
        board.movePiece(start, end);
        moveHistory.add(move);

        // Check game status
        Color opponentColor = (currentPlayer.getColor() == Color.WHITE) ? Color.BLACK : Color.WHITE;
        if (isCheckmate(opponentColor)) {
            status = (currentPlayer.getColor() == Color.WHITE) ? GameStatus.WHITE_WIN : GameStatus.BLACK_WIN;
        } else if (isStalemate(opponentColor)) {
            status = GameStatus.STALEMATE;
        }

        // Switch turn
        currentTurnIndex = (currentTurnIndex + 1) % 2;
        return true;
    }

    private boolean isKingInCheck(Color color) {
        Cell kingCell = findKing(color);
        // Check if any opponent piece can move to king's position
        Color opponent = (color == Color.WHITE) ? Color.BLACK : Color.WHITE;
        for (int i = 0; i < 8; i++) {
            for (int j = 0; j < 8; j++) {
                Cell cell = board.getCell(i, j);
                if (cell.isOccupied() && cell.getPiece().getColor() == opponent) {
                    if (cell.getPiece().canMove(board, cell, kingCell)) {
                        return true;
                    }
                }
            }
        }
        return false;
    }

    private boolean isCheckmate(Color color) {
        if (!isKingInCheck(color)) return false;
        // Check if any piece of this color can make a legal move to escape check
        return !hasLegalMove(color);
    }

    private boolean isStalemate(Color color) {
        if (isKingInCheck(color)) return false;
        return !hasLegalMove(color);
    }

    private boolean hasLegalMove(Color color) {
        for (int i = 0; i < 8; i++) {
            for (int j = 0; j < 8; j++) {
                Cell cell = board.getCell(i, j);
                if (cell.isOccupied() && cell.getPiece().getColor() == color) {
                    List<Cell> moves = cell.getPiece().getPossibleMoves(board, cell);
                    if (!moves.isEmpty()) return true;
                }
            }
        }
        return false;
    }

    private Cell findKing(Color color) {
        for (int i = 0; i < 8; i++)
            for (int j = 0; j < 8; j++) {
                Cell cell = board.getCell(i, j);
                if (cell.isOccupied() && cell.getPiece() instanceof King
                    && cell.getPiece().getColor() == color) {
                    return cell;
                }
            }
        return null;
    }
}
```

---

## 🎯 Design Patterns Used

| Pattern | Where |
|---------|-------|
| **Polymorphism** | Each piece overrides `canMove()` and `getPossibleMoves()` |
| **Command** | `Move` objects enable undo/redo |
| **Factory** | Can create pieces during pawn promotion |
| **Template Method** | Common path-checking logic shared by Rook/Bishop/Queen |

---

## ⚠️ Edge Cases

- Castling (King + Rook special move)
- En passant (pawn special capture)
- Pawn promotion (reach last rank)
- King cannot move into check
- Pin: piece cannot move if it exposes King to check
- Stalemate vs Checkmate distinction

---

## 🔑 Key Takeaways

1. **Polymorphism** is the core — each piece implements its own movement logic
2. Queen = Rook + Bishop (composition of move logic)
3. Knight is unique — jumps over pieces
4. **Move validation** must also check if it exposes your own king
5. Separate `canMove()` from `getPossibleMoves()` for efficiency
