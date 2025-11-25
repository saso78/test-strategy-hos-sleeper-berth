# TEST STRATEGY: Hours of Service (HoS) - 14h shift

## Feature Owner: *Sasho Trpovski*
Version: 1.0

Last Updated: [11/25/2025]

### 1. EXECUTIVE SUMMARY
## *Feature:*
    Hours of Service (HoS) 14h shift

## *Criticality:*  
    HIGH - Regulatory compliance feature (FMCSA ELD mandate)

## Scope: 
    API validation, database integrity, UI verification
## Test Strategy Approach:
    Focus on API-level validation (70%) with database verification and minimal UI coverage (10%) for critical user journeys. This aligns with testing pyramid best practices and provides fast, reliable feedback on regulatory compliance logic.

## 2. FEATURE OVERVIEW
*How is Shift and driving defined in Hours of service* 
### Shift is defined in Hours of service as the maximum amount of time a driver can be ON DUTY working. During this time period the driver can DRIVE , rest between drivings, perform inspection or fill up gas. After the 14h have passed the driver must change his status to OFF DUTY or SLEEPER BERTH for a total amount of 10h before the shift resets.
___

### 1. Define the Feature in Terms of Risk and Value
### ✔ Business impact

    This feature affects all users and is considered a critical feature of the whole platform. Unproper handeling may result in violation and if or when the driver is stoped the DOT officer will issue a citation and Out of Service notice. This will efectivly make all of the drivers to change carriesrs 

### ✔ Technical Risk: HIGH

Timer Precision Requirements:

    Timers must be calculated in seconds, not minutes
    14h 00m 00s = COMPLIANT (50,400 seconds)
    14h 00m 01s = VIOLATION (50,401 seconds)
    Single-second errors result in regulatory violations

### System Dependencies:

    Timer calculation engine (real-time and historical)
    Event log timestamp accuracy
    Database persistence (SQL timers table + MongoDB state documents)
    Timezone handling across state lines
    Clock synchronization between device and server

### Edge Cases:

    Midnight boundary crossings (shift starts 23:00, ends 13:00 next day)
    Daylight Saving Time transitions
    Retroactive event corrections
    Network latency causing delayed event submission
✔ User risk
If this fature fails the user will get a ticket they will need to  pay to the governing body and OOS ( out of service ) that affects the user (driver ) and the carrier ( ELD provider ).
1. Choose the Right Testing Levels (Pyramid Thinking)
Shift is a crucial part of the HOS feature. All Driving and non driving events happen durring the shift. The module is covered by unit tests but to validate we need to  validate or recreate the logic. 
API testing will be implemented. I need AUth token ,then we move to 
API call to validate the timers , then insert events. In order to do basic shift testing we can insert 1 on duty that is 14h long followed by an off duty . Validation of timers. We validate timers and violation via API and Database query.
Negative cases ON DUTY 14h00m01s and 14h01m00s , expected violation
UI / mobile test coverage only to validate same state of times as in DB as well as shown violations
1. Define Environment & Test Data Strategy
✔ Test Environments
We start to create the tests on dev enviroment and when succefull move on to prod. 
✔ Test Data
we create fresh data , on a new company that would be created on the test server. 
we need to  be able to  implement cleanup of old data if needed . Some test will requre inserting 7 days of data worth ( to validate cycle )
1. Define the Testing Techniques
Happy Path Scenarios:

Exact 14-hour shift - ON DUTY for exactly 14h 00m 00s → OFF DUTY
Under-limit shift - ON DUTY for 13h 45m 00s → OFF DUTY (no violation)
Mixed duty statuses - ON DUTY (2h) → DRIVING (8h) → ON DUTY (4h) = 14h total
10-hour reset - After 14h shift → OFF DUTY for 10h → New 14h shift available
Negative Scenarios:

One second over - ON DUTY for 14h 00m 01s (50,401s) → VIOLATION
One minute over - ON DUTY for 14h 01m 00s (50,460s) → VIOLATION
Insufficient reset - 14h shift → OFF DUTY 9h 59m → Try new shift → VIOLATION
Continuing after 14h - 14h shift completed → Continue DRIVING without rest → VIOLATION
Edge Cases:

Midnight crossing - Shift starts 22:00 Day 1, ends 12:00 Day 2 (exactly 14h)
DST spring forward - Shift starts 01:00, DST occurs at 02:00, verify timer calculation
Cycle limit interaction - 14h available in shift timer, but only 6h in cycle → Timer shows 6h
Retroactive edit - Event edited after shift end, recalculation must check for violations 
• API testing
Using Python + pytest + requests + API ( swagger )
• Database validation
SQL queries matching business logic
Mongo queries for document states
• Security testing
Auth flows, token expiry, role-based access
5. Define the Test Execution Plan
API
Integration
DB validation 
6. Define Your Quality Gates
API tests must pass 100% before UI automation runs
Test data reset between test runs
Review of results ( logs ) for anomalies 