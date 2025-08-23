# Question - Airport Security Scan Problem (In Progress)

```text
Design a security screening system for an airport that automatically sorts passengers into one of three priority lanes (US, EU, and Rest) based on their nationality to minimize delays.

Core Requirements
Input: The system must process passenger data, such as from a passport or boarding pass scan.

Processing: The system should be able to parse and validate nationality information.

Logic: A rule-based engine should categorize passengers into the correct lane:

US citizens get the highest priority.

EU citizens get medium priority.

All others are categorized as Rest.

Output: The system must communicate the assigned lane to the passenger in real-time.

Performance: The system must operate with high throughput and low latency to avoid creating bottlenecks.

Robustness: The system must handle errors gracefully, such as invalid data or scan failures, without a complete system failure.
```

## The Design Challenge

```text
The problem is a classic system design question. It requires me to think about the various components needed to build a real-world system and how they interact. Specifically, I need to outline a solution for

Input: How to get passenger data (e.g., from a passport or boarding pass).

Processing: How to analyze that data to determine nationality.

Classification: How to use business logic to categorize each passenger into one of the three priority lanes.

Output: How to communicate the assigned lane to the passenger (e.g., via a screen or a gate).

Performance: How to ensure the system is fast and reliable.

Error Handling: How to manage situations where data is missing or a scan fails, to prevent the system from stalling.
```

### 1\. System Goals and Requirements

* **Primary Goal:** Categorize passengers into one of three priority lanes:
  * **US Citizens:** Highest priority
  * **EU Citizens:** Medium priority
  * **Rest:** Lowest priority
* **Secondary Goal:** Minimize penalties for miscategorization. This implies a need for high accuracy and a mechanism for handling exceptions.
* **Performance:** The system must operate in real-time or near real-time to avoid creating bottlenecks at the security checkpoint.
* **Input:** The system's primary input will be passenger data, likely from a boarding pass scan, passport scan, or a mobile app.
* **Output:** The output is the assigned priority lane, which will then trigger a physical gate or display.
* **Scalability:** The system should be scalable to handle a high volume of passengers, especially during peak travel times.

### 2\. High-Level Design (Diagrammatic Format)

I'll use a diagrammatic approach to illustrate the data flow and system components.

```text
       +-------------------+
       |   Passenger Data  |
       |  (Boarding Pass,  |
       |  Passport Scan,   |
       |   Mobile App)     |
       +---------+---------+
                 |
                 v
       +-------------------+
       |  Data Acquisition |
       |   & Ingestion     |
       | (Scanners, APIs)  |
       +---------+---------+
                 |
                 v
       +-------------------+
       |     Passenger     |
       |   Data Processor  |
       |-------------------|
       | - Data Validation |
       | - Normalization   |
       | - Extract Nationality|
       | - Rule Engine     |
       +---------+---------+
                 |
                 +-------------------+
                 |
       +---------v---------+   +-------------------+
       |   Rule-Based      |   |   Exceptions      |
       |  Classification   |   |   Handler         |
       | (US, EU, Rest)    |   |-------------------|
       |-------------------|   | - Manual Override |
       |  - IF Nationality is US|   | - Fallback to 'Rest' |
       |  - IF Nationality is EU|   | - Logging for Audit |
       |  - ELSE -> Rest   |   +-------------------+
       +---------+---------+
                 |
                 v
       +-------------------+
       |   Lane Assignment |
       |     & Output      |
       |-------------------|
       | - Display Lane #  |
       | - Open Gate       |
       | - Real-time Queue |
       |   Management      |
       +-------------------+
```

### 3\. Component Breakdown

#### A. Data Acquisition & Ingestion

* **Function:** This is the entry point of the system. It collects passenger information from various sources.
* **Technology:**
  * **High-Speed Scanners:** For physical boarding passes and passports. These would use OCR (Optical Character Recognition) to extract key information like nationality, name, and flight details.
  * **APIs:** For integrating with airline check-in systems or mobile apps to receive pre-processed passenger data.
* **Key Consideration:** The system needs to handle different data formats and potential errors from scans.

#### B. Passenger Data Processor

* **Function:** Cleans, validates, and normalizes the incoming data. This is a critical step to ensure accuracy.
* **Sub-components:**
  * **Data Validation:** Checks if the extracted nationality code is valid (e.g., "US", "DE", "FR").
  * **Normalization:** Converts all nationality codes to a standard format (e.g., ISO 3166-1 alpha-2 codes).
  * **Extraction Engine:** The core logic to pull out the single most important piece of information: the passenger's nationality.

#### C. Rule-Based Classification Engine

* **Function:** This is the brain of the system. It applies business logic to categorize the passenger. A simple set of rules is the most effective and least error-prone approach for this problem.
* **Rules:**
    1. `IF nationality == 'US'` -\> Assign to `Priority Lane 1 (US)`
    2. `IF nationality IN ('DE', 'FR', 'IT', ...)` (All EU member states) -\> Assign to `Priority Lane 2 (EU)`
    3. `ELSE` (for all other nationalities) -\> Assign to `Priority Lane 3 (Rest)`
* **Benefits:** This is deterministic and highly accurate, which directly addresses the goal of minimizing penalties.

#### D. Exceptions Handler

* **Function:** Deals with all scenarios where the primary classification fails.
* **Scenarios:**
  * **Scan Failure:** The scanner couldn't read the nationality.
  * **Invalid Data:** The extracted nationality code is not recognized.
  * **Dual Citizenship:** A passenger may have both US and EU passports. The system would need a rule to prioritize, likely defaulting to the highest priority (US).
* **Actions:**
  * **Manual Override:** A security officer can manually input the correct lane.
  * **Fallback:** The system defaults the passenger to the "Rest" lane to ensure they are processed and don't block the system. This avoids a penalty for a system stall.
  * **Logging:** All exceptions should be logged for later analysis and system improvement.

#### E. Lane Assignment & Output

* **Function:** The final step, where the system communicates the decision to the physical world.
* **Outputs:**
  * **Display:** A digital screen shows the passenger's assigned lane number (e.g., "Go to Lane 1").
  * **Gate Control:** The system sends a signal to open the corresponding physical gate or barrier for the assigned lane.
* **Queue Management:** The system could also have a real-time display of the number of people in each lane to help passengers make their way to the correct area.

### 4\. Justification for Design Choices

* **Rule-Based System:** For a problem with clear, non-ambiguous rules (based on nationality), a simple rule engine is more reliable and faster than a complex machine learning model. An ML model would require extensive training data and could be prone to misclassification, which is the primary penalty we want to avoid.
* **Clear Data Flow:** The pipeline from data acquisition to lane assignment is straightforward, which makes the system easy to debug, maintain, and scale.
* **Explicit Exception Handling:** By building a separate component for handling errors, we ensure the system is robust and doesn't fail under pressure. The fallback to the "Rest" lane is a practical solution to keep the system moving and reduce bottlenecks.
* **Scalability:** The design is inherently scalable. The components can be deployed on a cloud infrastructure, allowing for horizontal scaling to handle increased passenger volume by simply adding more processing nodes.
