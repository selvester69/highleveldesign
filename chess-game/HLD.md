# High-Level Design Document

## 1. Document Information

### Document Metadata
- **Document Title**: Chess Game - Object-Oriented Design
- **Version**: 1.0
- **Date**: 2025-08-22
- **Author(s)**: Jules (AI Agent)
- **Document Status**: Draft

### Revision History
| Version | Date       | Author          | Changes         |
|---------|------------|-----------------|-----------------|
| 1.0     | 2025-08-22 | Jules (AI Agent)| Initial version |

---

## 2. Executive Summary

### 2.1 Purpose
This document provides an Object-Oriented Design (OOD) for a game of Chess. The design focuses on modeling the game's state, rules, pieces, and player interactions in a clean, extensible, and maintainable way.

### 2.2 Scope
**In Scope:**
- A standard 8x8 chessboard.
- All standard chess pieces (Pawn, Rook, Knight, Bishop, Queen, King).
- The basic movement rules for each piece.
- A two-player system, alternating turns.
- Tracking the game's state (e.g., active turn, check, checkmate, stalemate).
- A log of all moves made in the game.

**Out of Scope:**
- A graphical user interface (GUI).
- An AI opponent.
- Special rules like en passant, castling, and pawn promotion (though the design should allow for their addition).
- Timed games or clocks.

### 2.3 Key Classes/Objects
- **Game**: The main class that manages the overall game flow, including players, the board, and the current state.
- **Player**: Represents one of the two players (White or Black).
- **Board**: Represents the 8x8 chessboard and the placement of pieces on it.
- **Spot**: Represents a single square on the board.
- **Piece**: An abstract class for a chess piece, with concrete subclasses for `Pawn`, `Rook`, etc.
- **Move**: A data object representing a move from a starting spot to an ending spot.

### 2.4 High-Level Design Overview
The design is centered around the `Game` class, which acts as the main controller. It contains a `Board` object, two `Player` objects, and keeps track of whose turn it is. The `Board` is a collection of 64 `Spot` objects, each of which can contain a `Piece`. The `Piece` class is abstract, with each specific piece type (e.g., `Knight`, `Queen`) inheriting from it and implementing its own `canMove()` method to define its unique movement logic. When a player makes a `Move`, the `Game` class validates it against the current game state and the piece's rules before updating the `Board`. This approach encapsulates the rules within the relevant classes, making the system logical and easy to extend with more complex rules later.

---

## 3. System Overview & Use Cases

### 3.1 Core Use Cases
- **UC-01: Start a New Game**
  1. Two players start a new game.
  2. The system creates a `Game` object.
  3. The `Board` is initialized with all 32 pieces in their standard starting positions.
  4. The current turn is set to the White player.
- **UC-02: Player Makes a Valid Move**
  1. It is the current player's turn.
  2. The player chooses a piece to move from a start spot to a valid end spot.
  3. The system validates the move against the piece's movement rules and the game's rules (e.g., you cannot move into check).
  4. The system updates the board by moving the piece.
  5. If the move was a capture, the captured piece is removed from the board.
  6. The turn switches to the other player.
- **UC-03: Player Makes an Invalid Move**
  1. It is the current player's turn.
  2. The player attempts to move a piece to an invalid spot.
  3. The system rejects the move and informs the player.
  4. The game state does not change; it remains the current player's turn.
- **UC-04: Game Ends in Checkmate**
  1. A player makes a valid move that puts the opponent's King in check.
  2. The system determines that the opponent has no legal moves to get out of check.
  3. The system updates the game state to `CHECKMATE`.
  4. The current player is declared the winner.

---

## 4. Class Diagram

```
+-----------+       +-------------+       +------------------+
| Game      |<>---->| Player (2)  |       | Board            |<>----(64)----+--------------+
|-----------|       |-------------|       |------------------|              | Spot         |
| - board   |       | - isWhite   |       | - spots[8][8]    |              |--------------|
| - players |       +-------------+       |------------------|              | - x, y       |
| - current |                             | + getSpot(x,y)   |              | - piece      |
| - state   |                             | + setPiece(p,s)  |              +-------+------+
|-----------|                             +------------------+                      |
| + playerMove()|                                                                   |
| + changeTurn()|                                                                   |
+-----------+                                                                       |
                                                                                    |
                                            +------------------+                    |
                                            | Piece (Abstract) |<-------------------+
                                            |------------------|
                                            | - isKilled       |
                                            | - isWhite        |
                                            |------------------|
                                            | + canMove(b,s,e) |
                                            +------------------+
                                                    ^
                           +------------------------+------------------------+
                           |           |            |           |            |
                           v           v            v           v            v
                       +------+    +-------+    +--------+    +------+    +-------+
                       | Pawn |    | Rook  |    | Knight |    |Bishop|    | Queen | ...
                       +------+    +-------+    +--------+    +------+    +-------+
```

### 4.1 Key Classes
- **Game**: The main facade that manages the game logic, players, and board.
- **Player**: Represents a player. Simply holds their color (White/Black).
- **Board**: Represents the 8x8 grid. It is a collection of `Spot` objects and is responsible for managing the placement of pieces.
- **Spot**: Represents a single square on the board, defined by its x and y coordinates. It may or may not contain a `Piece`.
- **Piece (Abstract)**: The base class for all chess pieces. It defines common properties like color and a common abstract method `canMove(Board, StartSpot, EndSpot)`.
- **Pawn, Rook, etc. (Concrete Classes)**: Each subclass of `Piece` implements the specific movement logic for that piece type in its `canMove` method. For example, the `Knight`'s implementation would allow L-shaped moves.
- **Move**: A simple data class to represent a player's move, containing the start and end `Spot`.

---

## 5. Object Interaction / Sequence Diagram (Player makes a move)

```
+----------+      +------------+      +------------------+      +-------------+
|  Player  |      |    Game    |      |      Board       |      |    Piece    |
+----------+      +------------+      +------------------+      +-------------+
     |                  |                       |                      |
     | playerMove(move) |                       |                      |
     |----------------->|                       |                      |
     |                  | getPiece(move.start)  |                      |
     |                  |---------------------->|                      |
     |                  |                       |                      |
     |                  |       piece           |                      |
     |                  |<----------------------|                      |
     |                  |                       |                      |
     |                  | piece.canMove(board,  |                      |
     |                  |   move.start, move.end)                      |
     |                  |--------------------------------------------->|
     |                  |                       |                      |
     |                  |        true           |                      |
     |                  |<---------------------------------------------|
     |                  |                       |                      |
     |                  | updateBoard(move)     |                      |
     |                  |---------------------->|                      |
     |                  |                       |                      |
     |                  | changeTurn()          |                      |
     |                  |---------------->|     |                      |
     |                  |                 |     |                      |
     |                  |<----------------|     |                      |
     |                  |                       |                      |
```

## 6. API Design (Class Methods)

This section details the public methods of the core classes.

### 6.1 `Game` Class
- `Game()`: Constructor that initializes the board, pieces, and players.
- `boolean playerMove(Player player, int startX, int startY, int endX, int endY)`: The main method called by an external controller. It checks if it's the player's turn, validates the move, updates the board, and changes the game state. Returns true if the move was successful.
- `GameStatus getStatus()`: Returns the current status of the game (e.g., ACTIVE, CHECKMATE, STALEMATE).

### 6.2 `Piece` (Abstract Class)
- `Color getColor()`: Returns the piece's color.
- `abstract boolean canMove(Board board, Spot start, Spot end)`: The most important method. Each subclass will implement this to define its legal moves. For example, it checks if the path is clear for Rooks/Bishops/Queens.

### 6.3 `Board` Class
- `Spot getSpot(int x, int y)`: Returns the spot at the given coordinates.
- `void updateSpot(int x, int y, Piece piece)`: Places a piece on a spot (used for setup and moves).

---

## 7. - 15. Remaining Sections

For this OOD problem, the remaining sections are summarized:
- **Security**: Not applicable for a local, two-player game model.
- **Scalability**: Not applicable.
- **Deployment**: This design represents a library or a game engine's core logic. It would be packaged and included in a larger application that provides a UI.
- **Testing**: Unit testing is critical.
  - Each piece's `canMove` method must be tested exhaustively with valid and invalid moves on various board layouts.
  - The `Game` class's state management (check, checkmate, stalemate detection) must be tested for all edge cases.
- **Risks**: The main risk is the complexity of the rules. Chess has many special rules (castling, en passant) that are easy to implement incorrectly. The design should allow for these rules to be added and tested in isolation. A bug in the checkmate/stalemate detection logic is also a significant risk.
