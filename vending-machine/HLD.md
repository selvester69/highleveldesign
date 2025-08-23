# High-Level Design Document

## 1. Document Information

### Document Metadata
- **Document Title**: Vending Machine - Object-Oriented Design
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
This document provides an Object-Oriented Design (OOD) for a Vending Machine. The design focuses on identifying the key objects, their states, and their interactions to create a robust and extensible software model for the machine's logic.

### 2.2 Scope
**In Scope:**
- Handling a variety of items with different prices and quantities.
- Accepting different types of coins (e.g., Nickels, Dimes, Quarters).
- Selecting an item and dispensing it.
- Calculating and returning change.
- Handling inventory management (tracking item stock).
- Basic error handling (e.g., item out of stock, insufficient payment).

**Out of Scope:**
- Accepting bills or credit cards.
- A physical hardware interface.
- A networked API for remote management.
- Multi-currency support.

### 2.3 Key Classes/Objects
- **VendingMachine**: The main facade class that orchestrates the entire operation.
- **Item**: Represents a product in the machine (e.g., Soda, Chips) with a name and price.
- **Inventory**: A collection that holds `Item`s and their current stock count.
- **Coin**: An enumeration representing the different valid coin types and their values.
- **State**: A State design pattern will be used to manage the machine's different states (e.g., `NoCoinInsertedState`, `CoinInsertedState`, `DispensingState`).

### 2.4 High-Level Design Overview
The design is centered around a `VendingMachine` class that acts as a facade and holds the machine's state. We will use the **State design pattern** to manage the complex flow of operations. The machine transitions between states like `Idle`, `AcceptingMoney`, `Dispensing`, and `SoldOut` based on user actions (inserting coins, selecting items). The core logic involves managing an `Inventory` of `Item`s and handling monetary calculations for purchases and change-making. This approach encapsulates the logic for each state, making the system easier to understand and maintain.

---

## 3. System Overview & Use Cases

### 3.1 Core Use Cases
- **UC-01: Purchase an Item Successfully**
  1. User inserts sufficient coins.
  2. User selects a valid, in-stock item.
  3. Machine validates payment and selection.
  4. Machine dispenses the item.
  5. Machine dispenses correct change.
- **UC-02: Item is Out of Stock**
  1. User inserts coins.
  2. User selects an item that is out of stock.
  3. Machine displays an "Out of Stock" message.
  4. User can make another selection or request a refund.
- **UC-03: Insufficient Funds**
  1. User inserts some coins.
  2. User selects an item whose price is greater than the amount inserted.
  3. Machine displays the remaining amount needed.
  4. User can insert more coins or request a refund.
- **UC-04: Request Refund**
  1. User has inserted coins but has not completed a purchase.
  2. User presses the "Cancel" or "Refund" button.
  3. Machine returns all coins that were inserted.

---

## 4. Requirements Analysis (OOD Focus)

### 4.1 Functional Requirements
- **FR-001**: The machine must maintain an inventory of items and their prices.
- **FR-002**: The machine must accept coins of specific denominations (e.g., 5, 10, 25 cents).
- **FR-003**: The machine must keep track of the current balance inserted by the user.
- **FR-004**: The machine must dispense an item when a valid selection is made with sufficient funds.
- **FR-005**: The machine must return the correct change after a purchase.
- **FR-006**: The machine must not dispense an item if it is out of stock.
- **FR-007**: The machine must allow the user to cancel the transaction and receive a full refund of the current balance.

### 4.2 Non-Functional Requirements
- **Reliability**: The machine's state transitions and calculations must be correct and robust.
- **Extensibility**: The design should make it easy to add new item types or new coin denominations.
- **Maintainability**: The logic for each state of the machine should be isolated and easy to understand.

---

## 5. Class Diagram

```
+------------------+           +------------------+         +-----------------+
| VendingMachine   |<>-------->| State (Interface)|<>------>| Inventory       |
|------------------|           |------------------|         |-----------------|
| - currentState   |           | + insertCoin()   |         | - items: Map    |
| - inventory      |           | + selectItem()   |         | + addItem()     |
| - currentBalance |           | + dispenseItem() |         | + removeItem()  |
|------------------|           | + refund()       |         | + getCount()    |
| + insertCoin()   |           +------------------+         | + getPrice()    |
| + selectItem()   |                   ^                     +-------+---------+
| + cancel()       |                   |                             |
| + setState()     |                   |                             | 1..*
+------------------+                   |                             v
                                       |                     +-----------------+
        +------------------------------+-----------------+   | Item            |
        |                              |                 |   |-----------------|
        v                              v                 v   | - name: String  |
+------------------+      +--------------------+ +-----------------+ | - price: double |
| NoCoinState      |      | HasCoinState       | | SoldOutState    | +-----------------+
|------------------|      |--------------------| |-----------------|
| + insertCoin()   |      | + insertCoin()     | | ...             |
| + selectItem()   |      | + selectItem()     | +-----------------+
+------------------+      | + refund()         |
                          +--------------------+
```

### 5.1 Key Classes
- **VendingMachine**: The context class that holds the state and inventory. It delegates actions to the current state object.
- **State (Interface)**: Defines the common interface for all states. Each concrete state will implement the actions possible in that state.
- **NoCoinState**: The state when no money has been inserted. Only accepts `insertCoin()`.
- **HasCoinState**: The state after money has been inserted. Accepts more coins, a selection, or a refund request.
- **SoldOutState**: A state for when the machine has no items left.
- **DispensingState**: A transient state while the machine is dispensing an item.
- **Inventory**: A class that manages the stock of items. It's essentially a map from an `Item` to its quantity.
- **Item**: A simple data class representing a product with a name and price.
- **Coin (Enum)**: An enumeration for coin types (NICKEL, DIME, QUARTER) with their associated values.

---

## 6. Object Interaction / Sequence Diagram (Successful Purchase)

```
+----------+      +----------------+      +----------------+      +----------------+
|  User    |      | VendingMachine |      |  HasCoinState  |      |   Inventory    |
+----------+      +----------------+      +----------------+      +----------------+
     |                  |                       |                       |
     | insertCoin(0.25) |                       |                       |
     |----------------->|                       |                       |
     |                  | insertCoin(0.25)      |                       |
     |                  |---------------------->|                       |
     |                  |                       |                       |
     | selectItem("Soda")|                      |                       |
     |----------------->|                       |                       |
     |                  | selectItem("Soda")    |                       |
     |                  |---------------------->|                       |
     |                  |                       | getPrice("Soda")      |
     |                  |                       |---------------------->|
     |                  |                       |                       | 0.50
     |                  |                       |<----------------------|
     |                  |                       |                       |
     |                  |                       | getCount("Soda") > 0  |
     |                  |                       |---------------------->|
     |                  |                       |                       | true
     |                  |                       |<----------------------|
     |                  |                       |                       |
     |                  | dispenseItem()        |                       |
     |                  |---------------------->|                       |
     |                  |                       |                       |
     |                  | setState(Dispensing)  |                       |
     |                  |---------------------->|                       |
     |                  |                       |                       |
     |                  |                       | removeItem("Soda")    |
     |                  |                       |---------------------->|
     |                  |                       |                       |
     |< - - - - - - - - |                       |                       |
     |   (Dispense Item)|                       |                       |
     |                  |                       |                       |
     |< - - - - - - - - |                       |                       |
     |   (Return Change)|                       |                       |
     |                  |                       |                       |
```

## 7. State Diagram

```
                 +-----------------+
                 | NoCoinInserted  |
                 +-----------------+
                        |
                        | insertCoin()
                        v
                 +-----------------+
+--------------->| HasCoinInserted |<--------------+
|                +-----------------+               |
| refund()              |                          | insertCoin()
|                       | selectItem() [valid]     |
|                       v                          |
|                +-----------------+               |
|                |   Dispensing    |---------------+
|                +-----------------+   dispenseComplete()
|                       |
|                       | outOfStock()
|                       v
|                +-----------------+
+--------------->|    SoldOut      |
                 +-----------------+
```

## 8. API Design (Class Methods)

This section details the public methods of the core classes.

### 8.1 `VendingMachine` Class
- `VendingMachine(Inventory initialInventory)`: Constructor.
- `void insertCoin(Coin coin)`: Adds a coin to the current balance and informs the current state.
- `void selectItem(String itemName)`: Selects an item and informs the current state.
- `List<Coin> cancelTransaction()`: Cancels the current transaction and returns the inserted coins.
- `void setState(State newState)`: Internal method for states to transition the machine.
- `double getCurrentBalance()`: Returns the current balance.
- `Inventory getInventory()`: Returns a reference to the inventory.

### 8.2 `State` Interface
- `void insertCoin(VendingMachine machine, Coin coin)`
- `void selectItem(VendingMachine machine, String itemName)`
- `void dispenseItem(VendingMachine machine)`
- `List<Coin> refund(VendingMachine machine)`

---

## 9. - 15. Remaining Sections

For an OOD problem, the remaining sections are less critical than for a large distributed system. They can be summarized:
- **Security**: Not applicable for this offline model.
- **Scalability**: Not applicable.
- **Deployment**: The code would be compiled and deployed onto the vending machine's embedded hardware.
- **Testing**: Unit testing is critical. Each state's logic must be tested thoroughly. A test suite would simulate user interactions (inserting coins, making selections) and assert that the machine is in the correct state with the correct balance after each action.
- **Risks**: The main risk is a bug in the state machine logic leading to an incorrect state (e.g., dispensing an item without sufficient payment). Rigorous unit testing of each state transition is the primary mitigation.
