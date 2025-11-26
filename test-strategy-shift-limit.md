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
    13h 59m 59s = COMPLIANT (50,399 seconds)
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

### ✔ User risk
### *If this fature fails the user will get a ticket they will need to  pay to the governing body and OOS ( out of service ) that affects the user (driver ) and the carrier ( ELD provider ).*
___

### 1. Choose the Right Testing Levels (Pyramid Thinking)
Shift is a crucial part of the HOS feature. All Driving and non driving events happen durring the shift. The module is covered by unit tests but to validate we need to  validate or recreate the logic. 
### Testing Pyramid Distribution:
```
         /\
        /UI\ ← 10% - Mobile/Web critical path validation
       /----\
      / API  \ ← 60% - Core business logic validation
     /--------\
    /   DB     \ ← 30% - Data integrity & state validation
   /------------\
```
##### Why This Distribution?

    API Layer (60% of effort):

    Business logic is in the API
    Fastest feedback loop
    Most stable tests
    Easy to validate calculations
    Can test all edge cases efficiently

    Database Layer (30% of effort):

    Validates data persistence
    Checks referential integrity
    Confirms violation logging
    Verifies timer state across tables

    UI Layer (10% of effort):

    Validates display accuracy
    Confirms mobile/web parity
    Tests critical user workflows only
    Reserved for paths that MUST use interface


### 1. Define Environment & Test Data Strategy
✔ Test Environments

    We start to create the tests on dev enviroment and when succefull move on to prod. 

✔ Test Data

    we create fresh data , on a new company that would be created on the test server. 
    we need to  be able to  implement cleanup of old data if needed . Some test will requre inserting 7 days of data worth ( to validate cycle )
___
### Define the Testing Techniques
#### Happy Path Scenarios:

* Exact 14-hour shift - ON DUTY for exactly 14h 00m 00s → OFF DUTY
* Under-limit shift - ON DUTY for 13h 45m 00s → OFF DUTY (no violation)
* Mixed duty statuses - ON DUTY (2h) → DRIVING (8h) → ON DUTY (4h) = 14h total
* 10-hour reset - After 14h shift → OFF DUTY for 10h → New 14h shift available


#### Negative Scenarios:

* One second over - ON DUTY for 14h 00m 01s (50,401s) → VIOLATION
* One minute over - ON DUTY for 14h 01m 00s (50,460s) → VIOLATION
* Insufficient reset - 14h shift → OFF DUTY 9h 59m → Try new shift → VIOLATION
* Continuing after 14h - 14h shift completed → Continue DRIVING without rest → VIOLATION

#### Edge Cases:

* Midnight crossing - Shift starts 22:00 Day 1, ends 12:00 Day 2 (exactly 14h)
* DST spring forward - Shift starts 01:00, DST occurs at 02:00, verify timer calculation
* Cycle limit interaction - 14h available in shift timer, but only 6h in cycle → Timer shows 6h
* Retroactive edit - Event edited after shift end, recalculation must check for violations
 
### API testing
Using Python + pytest + requests + API ( swagger )

#### Database validation
  
    SQL queries matching business logic
    Mongo queries for document states
#### Security testing
    Auth flows, token expiry, role-based access
#### Define the Test Execution Plan
    API
    Integration
    DB validation 
### Quality Gates
    API tests must pass 100% before UI automation runs
    Test data reset between test runs
    Review of results ( logs ) for anomalies 
___
### API Test Pseudocode Example
```python
pythondef test_14h_shift_exact_limit():
    """
    Test: Shift timer at exactly 14 hours (no violation)
    
    Business Rule: 14h 00m 00s is compliant, 14h 00m 01s is violation
    """
    # 1. Setup: Authenticate and get clean driver
    token = authenticate(test_driver)
    reset_driver_logs(test_driver.id)
    
    # 2. Verify initial state
    timers = get_timers(token)
    assert timers["shift_remaining"] == 50400  # 14 hours in seconds
    assert timers["drive_remaining"] == 39600  # 11 hours in seconds
    
    # 3. Insert ON DUTY event for exactly 14 hours
    start_time = datetime.now(timezone.utc)
    end_time = start_time + timedelta(hours=14)  # Exactly 14h 00m 00s
    
    response = post_event(
        token=token,
        event_type="ON",
        start_time=start_time.isoformat(),
        end_time=end_time.isoformat()
    )
    assert response.status_code == 201
    
    # 4. Validate timers after event
    timers = get_timers(token)
    assert timers["shift_remaining"] == 0  # Exactly at limit
    assert timers["drive_remaining"] == 39600  # Unchanged (no driving)
    
    # 5. Validate NO VIOLATION logged
    violations = get_violations(token, start_time=start_time)
    shift_violations = [v for v in violations if v["type"] == "SHIFT_LIMIT"]
    assert len(shift_violations) == 0, "Should be no violation at exactly 14h"
    
    # 6. Validate database state
    db_timers = query_db(f"SELECT * FROM timers WHERE user_id = {test_driver.id}")
    assert db_timers[0]["shift_remaining"] == 0
    
    db_violations = query_db(f"""
        SELECT * FROM violations 
        WHERE user_id = {test_driver.id} 
        AND violation_type = 'SHIFT_LIMIT'
        AND created_at >= '{start_time}'
    """)
    assert len(db_violations) == 0


def test_14h_shift_one_second_over_violation():
    """
    Test: Shift timer at 14h 00m 01s triggers violation
    
    Business Rule: Any time over 14h 00m 00s is a violation
    """
    # Setup
    token = authenticate(test_driver)
    reset_driver_logs(test_driver.id)
    
    # Insert ON DUTY for 14 hours and 1 second
    start_time = datetime.now(timezone.utc)
    end_time = start_time + timedelta(hours=14, seconds=1)  # 14h 00m 01s
    
    response = post_event(
        token=token,
        event_type="ON",
        start_time=start_time.isoformat(),
        end_time=end_time.isoformat()
    )
    assert response.status_code == 201
    
    # Validate timers show negative (over limit)
    timers = get_timers(token)
    assert timers["shift_remaining"] == -1  # 1 second over limit
    
    # Validate VIOLATION logged
    violations = get_violations(token, start_time=start_time)
    shift_violations = [v for v in violations if v["type"] == "SHIFT_LIMIT"]
    
    assert len(shift_violations) == 1, "Should have exactly 1 shift violation"
    assert shift_violations[0]["severity"] == "CRITICAL"
    assert shift_violations[0]["seconds_over"] == 1
    
    # Validate database logged violation
    db_violations = query_db(f"""
        SELECT * FROM violations 
        WHERE user_id = {test_driver.id} 
        AND violation_type = 'SHIFT_LIMIT'
        AND created_at >= '{start_time}'
    """)
    assert len(db_violations) == 1
    assert db_violations[0]["seconds_over"] == 1


def test_14h_shift_cycle_interaction():
    """
    Test: Shift timer respects cycle timer limit
    
    Business Rule: If cycle has 6h left, shift timer shows 6h (not 14h)
    """
    # Setup: Driver with only 6 hours left in cycle
    token = authenticate(test_driver)
    setup_driver_with_cycle_remaining(test_driver.id, hours=6)
    
    # Validate timers show cycle-limited shift
    timers = get_timers(token)
    assert timers["cycle_remaining"] == 21600  # 6 hours in seconds
    assert timers["shift_remaining"] == 21600  # Also 6 hours (limited by cycle)
    assert timers["drive_remaining"] == 21600  # Also 6 hours (limited by cycle)
    
    # Insert ON DUTY for 6 hours (at cycle limit)
    start_time = datetime.now(timezone.utc)
    end_time = start_time + timedelta(hours=6)
    
    post_event(token, "ON", start_time, end_time)
    
    # Validate all timers at zero
    timers = get_timers(token)
    assert timers["cycle_remaining"] == 0
    assert timers["shift_remaining"] == 0
    assert timers["drive_remaining"] == 0
    
    # No violation (didn't exceed cycle)
    violations = get_violations(token, start_time=start_time)
    assert len(violations) == 0
```

---