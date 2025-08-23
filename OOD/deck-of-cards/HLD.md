# High-Level Design Document

## 1. Document Information

### Document Metadata
- **Document Title**: Deck of Cards - Object-Oriented Design
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
This document provides a simple Object-Oriented Design (OOD) for a standard 52-card deck of playing cards. The design provides a foundational set of classes that can be used as the basis for various card games.

### 2.2 Scope
**In Scope:**
- Representing a standard playing card with a suit and a rank.
- Representing a standard 52-card deck.
- The ability to shuffle the deck.
- The ability to deal one or more cards from the deck.

**Out of Scope:**
- The logic for any specific card game (e.g., Poker, Blackjack).
- The concept of a "hand" of cards for a player.
- Jokers or any non-standard cards.

### 2.3 Key Classes/Objects
- **Card**: Represents a single playing card with a `Suit` and a `Rank`.
- **Deck**: Represents the collection of 52 `Card` objects.
- **Suit (Enum)**: An enumeration for the four suits (HEARTS, DIAMONDS, CLUBS, SPADES).
- **Rank (Enum)**: An enumeration for the 13 ranks (TWO, THREE, ..., KING, ACE).

### 2.4 High-Level Design Overview
The design is composed of a few simple, clear classes. The core of the design is the `Deck` class, which is initialized with a standard, ordered set of 52 `Card` objects. The `Deck` class provides essential behaviors, namely `shuffle()` to randomize the order of its cards and `deal()` to remove and return cards from the collection. The `Card` class itself is a simple, immutable data object composed of two enumerations, `Suit` and `Rank`, which ensures type safety and clarity. This design provides a clean, reusable, and easy-to-understand foundation for any application that requires a deck of cards.

---

## 3. System Overview & Use Cases

### 3.1 Core Use Cases
- **UC-01: Create a new Deck**
  1. A game requires a new deck of cards.
  2. The system instantiates a `Deck` object.
  3. The new deck contains 52 unique `Card` objects in a standard order.
- **UC-02: Shuffle the Deck**
  1. A game requires the deck to be randomized.
  2. The `shuffle()` method is called on the `Deck` object.
  3. The internal order of the cards in the deck is now random.
- **UC-03: Deal a Card**
  1. A game needs to deal a card to a player.
  2. The `deal()` method is called on the `Deck` object.
  3. The top card is removed from the deck and returned to the caller.
  4. The deck now contains one fewer card.

---

## 4. Class Diagram

```
+------------------+<>----1..52---+-----------------+
| Deck             |             | Card            |
|------------------|             |-----------------|
| - cards: List    |             | - suit: Suit    |
|------------------|             | - rank: Rank    |
| + shuffle()      |             |-----------------|
| + deal(): Card   |             | + toString()    |
| + deal(n): List  |             +-------+---------+
| + size(): int    |                     |      |
+------------------+                     |      |
                                         v      v
                               +----------+  +----------+
                               | Suit(Enum) |  | Rank(Enum) |
                               +----------+  +----------+
```

### 4.1 Key Classes
- **Deck**: Manages a list of `Card` objects. The constructor creates the 52 standard cards.
- **Card**: An immutable data object representing a single card.
- **Suit (Enum)**: Contains four constants: `HEARTS`, `DIAMONDS`, `CLUBS`, `SPADES`.
- **Rank (Enum)**: Contains thirteen constants: `TWO`, `THREE`, `FOUR`, `FIVE`, `SIX`, `SEVEN`, `EIGHT`, `NINE`, `TEN`, `JACK`, `QUEEN`, `KING`, `ACE`. Can also have a method `getValue()` to return the numeric value (e.g., for Blackjack).

---

## 5. API Design (Class Methods)

### 5.1 `Deck` Class
- `Deck()`: Constructor that creates a standard 52-card deck.
- `void shuffle()`: Randomizes the order of the cards in the deck.
- `Card deal()`: Removes and returns the top card from the deck. Returns `null` or throws an exception if the deck is empty.
- `List<Card> deal(int n)`: Deals `n` cards.
- `int size()`: Returns the number of cards remaining in the deck.

### 5.2 `Card` Class
- `Card(Suit suit, Rank rank)`: Constructor.
- `Suit getSuit()`: Returns the card's suit.
- `Rank getRank()`: Returns the card's rank.
- `String toString()`: Returns a human-readable representation, e.g., "Ace of Spades".

---

## 6. - 15. Remaining Sections

For this simple OOD problem, the remaining sections are not applicable in a traditional sense but can be summarized:
- **Testing**: Unit tests are essential.
  - Test that a new `Deck` has exactly 52 unique cards.
  - Test that `shuffle()` randomizes the order (e.g., by comparing the order before and after).
  - Test that `deal()` removes a card and reduces the deck size.
  - Test that dealing from an empty deck is handled correctly.
- **Risks**: The primary risk is an implementation bug, such as an off-by-one error in the `deal` method or an incorrect `shuffle` algorithm. Rigorous unit testing mitigates this.
