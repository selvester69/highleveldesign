# High-Level Design Document

## 1. Document Information

### Document Metadata

- **Document Title**: ATM System - Object-Oriented Design
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

This document provides an Object-Oriented Design (OOD) for an Automated Teller Machine (ATM). The design models the components of an ATM, user interactions, and its communication with a backend bank system to process transactions.

### 2.2 Scope

**In Scope:**

- User authentication via a card and PIN.
- The ability to check the account balance.
- The ability to deposit cash.
- The ability to withdraw cash.
- A state machine to manage the ATM's operational flow.
- Interaction with a remote bank server to validate and process transactions.

**Out of Scope:**

- The internal logic of the backend bank system. It will be treated as an external service.
- Dispensing or accepting checks.
- Transferring funds between accounts.
- The physical hardware components (card reader, cash dispenser).

### 2.3 Key Classes/Objects

- **ATM**: The main facade class that manages the machine's state and hardware components.
- **State**: An interface for the State design pattern, with concrete states like `IdleState`, `CardInsertedState`, `AuthenticatedState`.
- **Card**: A data object representing the user's ATM card.
- **Account**: A data object representing the user's bank account.
- **BankService**: A proxy or adapter for the external bank's server, used to authenticate users and execute transactions.
- **Transaction**: An abstract class for transactions, with concrete subclasses like `BalanceInquiry`, `Deposit`, `Withdrawal`.

### 2.4 High-Level Design Overview

The design is centered around the `ATM` class, which manages the machine's state using the **State design pattern**. The ATM transitions between states like `Idle`, `CardInserted`, `PIN_Validation`, `SelectOperation`, and `TransactionProcessing` based on user input. When a user initiates a transaction (e.g., `Withdrawal`), a `Transaction` object is created. The `ATM` class then uses the `BankService` interface to communicate with the backend bank system to authorize and execute this transaction. This design cleanly separates the ATM's own logic (state management, user interaction) from the external banking logic, and the use of the State pattern makes the complex user session flow manageable and robust.

---

## 3. System Overview & Use Cases

### 3.1 Core Use Cases

- **UC-01: Authenticate User**
  1. User inserts an ATM card.
  2. The system prompts for a PIN.
  3. User enters their PIN.
  4. The system validates the card and PIN with the bank server.
  5. On success, the system displays the main menu of operations.
- **UC-02: Withdraw Cash**
  1. User is authenticated.
  2. User selects "Withdraw Cash" from the menu.
  3. User enters the desired amount.
  4. The system validates the amount against the user's account balance (via the bank server) and the cash available in the machine.
  5. On success, the system dispenses the cash and ejects the card.
- **UC-03: Deposit Cash**
  1. User is authenticated.
  2. User selects "Deposit Cash".
  3. User inserts cash into the deposit slot.
  4. The system counts the cash and confirms the amount with the user.
  5. On confirmation, the system sends the deposit transaction to the bank server and ejects the card.
- **UC-04: Invalid PIN**
  1. User inserts a card and enters an incorrect PIN.
  2. The system displays an "Invalid PIN" message.
  3. If the PIN is entered incorrectly multiple times (e.g., 3 times), the system retains the card and ends the session.

---

## 4. Class Diagram

```text
+--------------+<>----1----+-----------------+      +-----------------+
| ATM          |          | State(Interface)|      | Card            |
|--------------|          |-----------------|      |-----------------|
| - atmState   |          | + insertCard()  |      | - cardNumber    |
| - cashAmount |          | + enterPIN()    |      | - expiration    |
|--------------|          | + selectTxn()   |      +-----------------+
| + setState() |          +-----------------+
+--------------+                  ^
                                  |
            +---------------------+---------------------+
            |                     |                     |
            v                     v                     v
+-----------------+   +--------------------+   +--------------------+
| IdleState       |   | CardInsertedState  |   | AuthenticatedState | ...
+-----------------+   +--------------------+   +--------------------+


+------------------+      +--------------------------+
| BankService      |----->| Account                  |
| (Interface/Proxy)|      |--------------------------|
|------------------|      | - accountNumber          |
| + authenticate() |      | - balance                |
| + executeTxn()   |      +--------------------------+
+------------------+
         ^
         |
+--------+---------+<>----1----+-----------------+
| Transaction      |          | ATM             |
| (Abstract)       |          +-----------------+
|------------------|
| - transactionID  |
|------------------|
| + execute()      |
+------------------+
         ^
+--------+---------+
| ...  Withdrawal  |
+------------------+
```

### 4.1 Key Classes

- **ATM**: The main context class, managing the state of the ATM.
- **State (Interface)**: Defines the actions a user can perform in a given state. Concrete subclasses (`IdleState`, `AuthenticatedState`, etc.) implement this interface. The State pattern is central to this design.
- **Card**: A simple data object representing the physical card.
- **BankService (Adapter/Proxy)**: An object that encapsulates all communication with the remote bank's servers. This is a critical separation of concerns.
- **Account**: A data object representing the user's account information, retrieved from the `BankService`.
- **Transaction (Command Pattern)**: An abstract class representing a transaction. Subclasses like `Withdrawal` or `Deposit` encapsulate the logic for that specific transaction type.

---

## 5. Object Interaction / Sequence Diagram (Withdraw Cash)

```text
+----------+      +--------------+      +--------------------+      +-------------+
|  User    |      |     ATM      |      | AuthenticatedState |      | BankService |
+----------+      +--------------+      +--------------------+      +-------------+
     |                  |                       |                       |
     | selectWithdraw() |                       |                       |
     |----------------->|                       |                       |
     |                  | selectTxn(WITHDRAW)   |                       |
     |                  |---------------------->|                       |
     |                  |                       |                       |
     | enterAmount(100) |                       |                       |
     |----------------->|                       |                       |
     |                  | createTxn(Withdrawal) |                       |
     |                  |---------------------->|                       |
     |                  |                       |                       |
     |                  |   executeTxn(Txn)     |                       |
     |                  |---------------------------------------------->|
     |                  |                       |                       |
     |                  |      SUCCESS          |                       |
     |                  |<----------------------------------------------|
     |                  |                       |                       |
     |                  | dispenseCash(100)     |                       |
     |                  |---------------------->|                       |
     |                  |                       |                       |
     |< - - - - - - - - |                       |                       |
     | (Dispense Cash)  |                       |                       |
     |                  |                       |                       |
     |< - - - - - - - - |                       |                       |
     | (Eject Card)     |                       |                       |
     |                  |                       |                       |
```

## 6. State Diagram

```text
                 +-----------+
                 |   Idle    |
                 +-----------+
                      |
                      | insertCard()
                      v
                 +-----------------+
                 | CardInserted    |
                 +-----------------+
                      |
                      | enterPIN()
                      v
+--------------->+-----------------+<-----------------+
| incorrectPIN   | PIN_Validation  | correctPIN       |
+----------------+-----------------+----------------->+
                      |
                      | maxAttemptsReached
                      v
                 +-----------+
                 | CardEaten |
                 +-----------+

+------------------+
| Authenticated    |
+------------------+
       |
       | selectOperation()
       v
+------------------+       +------------------+
|   Transaction    |------>|    CardEjected   |
|   Processing     |       +------------------+
+------------------+
```

## 7. API Design (Class Methods)

### 7.1 `ATM` Class

- `ATM(BankService bankService, double initialCash)`: Constructor.
- `void insertCard(Card card)`: Called by the hardware. Delegates to the current state.
- `void enterPIN(int pin)`: Delegates to the current state.
- `void selectTransactionType(TransactionType type)`: Delegates to the current state.
- `void executeTransaction(double amount)`: Delegates to the current state.
- `void ejectCard()`: A final action to return the card.

### 7.2 `State` Interface

- `void insertCard(ATM atm, Card card)`
- `void enterPIN(ATM atm, int pin)`
- `void selectTransaction(ATM atm, TransactionType type, double amount)`
- `void ejectCard(ATM atm)`

---

## 8. - 15. Remaining Sections

For this OOD problem, the remaining sections are summarized:

- **Security**: The connection to the bank must be secure (e.g., over a VPN or private network with TLS). PINs should never be logged. The physical security of the machine is also a major concern.
- **Scalability**: For the software model, scalability is not a concern. For the backend `BankService`, it would need to handle transactions from thousands of ATMs concurrently.
- **Deployment**: The ATM software runs on embedded hardware within the machine itself.
- **Testing**: Unit testing the State pattern is crucial. Each state must be tested to ensure it correctly handles all possible inputs, either by performing an action and transitioning state, or by rejecting the action. Integration testing with a mock `BankService` is also essential.
- **Risks**: The main risk is a discrepancy between a transaction processed by the bank and the action taken by the machine (e.g., cash dispensed but transaction fails). The system must use a robust two-phase commit or transactional outbox pattern when communicating with the bank to ensure atomicity.
