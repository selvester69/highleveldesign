# High-Level Design Document

## 1. Document Information

### Document Metadata
- **Document Title**: Parking Lot Management System - Object-Oriented Design
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
This document provides an Object-Oriented Design (OOD) for a Parking Lot Management System. The design focuses on the classes required to model a parking lot, including different types of parking spots, vehicles, and the ticketing/payment process.

### 2.2 Scope
**In Scope:**
- Handling multiple types of parking spots (e.g., Compact, Large, Motorcycle).
- Handling different types of vehicles (e.g., Car, Truck, Motorcycle).
- A vehicle can only park in a spot that is large enough for it.
- Issuing a ticket upon entry, which records the vehicle details and entry time.
- Calculating a fee upon exit based on the duration of the stay.
- Displaying the number of available spots for each type.

**Out of Scope:**
- Handling payments via cash or credit card.
- A physical gate/barrier system.
- Reservations or pre-booked parking.
- Valet parking services.

### 2.3 Key Classes/Objects
- **ParkingLot**: The main class, representing the entire parking lot, which contains multiple levels and spots.
- **ParkingSpot**: An abstract class representing a single parking spot, with concrete subclasses like `CompactSpot`, `LargeSpot`, etc.
- **Vehicle**: An abstract class representing a vehicle, with concrete subclasses like `Car`, `Truck`, `Motorcycle`.
- **Ticket**: An object that stores information about a parking session, including the vehicle, spot, and entry time.
- **PaymentCalculator**: A strategy object for calculating parking fees.

### 2.4 High-Level Design Overview
The design is centered around the `ParkingLot` class, which acts as a singleton or a central facade. It manages a collection of `ParkingSpot`s of various types. When a `Vehicle` arrives, the system finds a suitable available spot, assigns the vehicle to it, and generates a `Ticket`. When the vehicle leaves, the `Ticket` is used to calculate the duration and fee. The system uses abstract classes for `Vehicle` and `ParkingSpot` to allow for easy extension with new types (e.g., adding an `ElectricSpot` with charging capabilities). The core logic involves managing the availability of spots and the lifecycle of a `Ticket`.

---

## 3. System Overview & Use Cases

### 3.1 Core Use Cases
- **UC-01: Park a Vehicle**
  1. A vehicle arrives at the parking lot entrance.
  2. The system checks for an available spot of a suitable size.
  3. If a spot is available, the system issues a ticket.
  4. The ticket contains the spot number, vehicle license plate, and entry time.
  5. The system marks the spot as occupied.
- **UC-02: Un-park a Vehicle**
  1. A driver presents their ticket at the exit.
  2. The system retrieves the entry time from the ticket.
  3. The system calculates the parking duration and the total fee.
  4. (Out of scope: User pays the fee).
  5. The system marks the parking spot as vacant.
- **UC-03: No Spot Available**
  1. A vehicle arrives at the entrance.
  2. The system checks for a suitable spot and finds none.
  3. The system displays a "Full" message on the entrance display.
- **UC-04: Display Vacancy**
  1. The system continuously updates a public display board with the number of available spots for each type (e.g., Compact: 5, Large: 2).

---

## 4. Requirements Analysis (OOD Focus)

### 4.1 Functional Requirements
- **FR-001**: The parking lot must have a fixed number of spots of different types.
- **FR-002**: The system must be able to assign a vehicle to a parking spot. A motorcycle can park in any spot, a car in a compact or large spot, and a truck only in a large spot.
- **FR-003**: The system must not assign a vehicle to a spot that is already occupied.
- **FR-004**: The system must generate a unique ticket for each parked vehicle.
- **FR-005**: The system must calculate the parking fee based on an hourly rate.

---

## 5. Class Diagram

```
+----------------+<>----1..*---+-----------------+      +-----------------+
| ParkingLot     |             | ParkingSpot     |----|>| Vehicle         |
|----------------|             | (Abstract)      |      | (Abstract)      |
| - spots: List  |             |-----------------|      |-----------------|
| - tickets: Map |             | - spotNumber    |      | - licensePlate  |
|----------------|             | - isOccupied    |      | - vehicleType   |
| + parkVehicle()|             | - vehicle       |      | - ticket        |
| + unparkVehicle()|           |-----------------|      +-----------------+
| + findSpot()   |             | + assignVehicle()|            ^
+----------------+             | + removeVehicle()|            |
                               +-----------------+            |
                                       ^                      |
                                       |                      |
            +----------------+---------+---------+  +---------+---------------+
            |                |                   |  |         |               |
            v                v                   v  v         v               v
+-----------------+ +-----------------+ +-----------------+ +---------+ +-----------+ +-------------+
| MotorcycleSpot  | | CompactSpot     | | LargeSpot       | |Motorcycle| | Car       | | Truck       |
+-----------------+ +-----------------+ +-----------------+ +----------+ +-----------+ +-------------+
```

### 5.1 Key Classes
- **ParkingLot**: The main class that manages the entire lot.
- **ParkingSpot (Abstract)**: Represents a generic spot. Has concrete subclasses `MotorcycleSpot`, `CompactSpot`, `LargeSpot`. Each spot knows if it's occupied and by which vehicle.
- **Vehicle (Abstract)**: Represents a generic vehicle. Has concrete subclasses `Motorcycle`, `Car`, `Truck`. Each vehicle type has a size that determines which spots it can fit in.
- **Ticket**: A data object holding a reference to the parked vehicle, the spot number, and the entry timestamp.
- **PaymentCalculator (Strategy Pattern)**: A separate class responsible for calculating fees. This decouples the fee calculation logic from the core parking logic, allowing for different fee strategies (e.g., hourly, daily, weekend).
- **DisplayBoard**: A class responsible for showing the number of vacant spots. It would observe the `ParkingLot` for changes.

---

## 6. Object Interaction / Sequence Diagram (Park Vehicle)

```
+----------+      +--------------+      +-----------------+      +-----------+
|  Driver  |      |  ParkingLot  |      |   ParkingSpot   |      |  Ticket   |
+----------+      +--------------+      +-----------------+      +-----------+
     |                  |                       |                      |
     |  parkVehicle(Car)|                       |                      |
     |----------------->|                       |                      |
     |                  | findSpot(Car)         |                      |
     |                  |---------------------->|                      |
     |                  |                       |                      |
     |                  |    spot_123           |                      |
     |                  |<----------------------|                      |
     |                  |                       |                      |
     |                  | assignVehicle(Car)    |                      |
     |                  |---------------------->|                      |
     |                  |                       |                      |
     |                  | createTicket(Car, spot_123)                  |
     |                  |--------------------------------------------->|
     |                  |                       |                      |
     |    ticket        |                       |                      |
     |<-----------------|                       |                      |
     |                  |                       |                      |
```

## 7. API Design (Class Methods)

This section details the public methods of the core classes.

### 7.1 `ParkingLot` Class
- `boolean isFull(VehicleType type)`: Checks if there are any available spots for a given vehicle type.
- `Ticket parkVehicle(Vehicle vehicle)`: Finds a spot, parks the vehicle, and returns a ticket. Throws an exception if the lot is full.
- `double unparkVehicle(Ticket ticket)`: Marks the spot as free and calculates the fee.
- `int getAvailableSpotCount(VehicleType type)`: Returns the number of available spots for a vehicle type.

### 7.2 `ParkingSpot` (Abstract Class)
- `boolean isOccupied()`: Returns true if a vehicle is parked in the spot.
- `boolean canFitVehicle(Vehicle vehicle)`: An abstract method to be implemented by subclasses. Checks if the vehicle can fit in the spot.
- `void assignVehicle(Vehicle vehicle)`: Assigns a vehicle to this spot.
- `void removeVehicle()`: Frees up the spot.

### 7.3 `Vehicle` (Abstract Class)
- `VehicleType getType()`: Returns the type of the vehicle.
- `String getLicensePlate()`: Returns the license plate.

---

## 8. - 15. Remaining Sections

For an OOD problem, the remaining sections are less critical than for a large distributed system. They can be summarized:
- **Security**: The system needs to prevent fraudulent tickets. Tickets should have a unique, unguessable ID.
- **Scalability**: For a single parking lot, scalability is not a major concern. If the system were extended to manage multiple lots, the `ParkingLot` class would no longer be a singleton, and a central service would be needed to manage all lots.
- **Deployment**: The code would be deployed on a server at the parking lot, connected to entry/exit terminals.
- **Testing**: Unit testing is critical for the parking/unparking logic and fee calculation. Test cases should include various vehicle types, spot types, and parking durations.
- **Risks**: The main risk is a bug in the spot allocation logic, leading to two vehicles being assigned to the same spot or a vehicle being assigned to a spot that is too small.
