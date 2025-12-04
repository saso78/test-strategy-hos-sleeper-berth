# Hours of Service (HoS) Compliance Test Strategies

Comprehensive test strategies for FMCSA Hours of Service regulations in ELD platforms.

## Overview

This repository contains strategic test planning documents for critical Hours of Service (HoS) features in Electronic Logging Device (ELD) platforms. These strategies demonstrate risk-based testing, testing pyramid methodology, and shift-left quality practices for regulatory compliance.

## Test Strategies

### 1. [14-Hour Shift Limit](test-strategy-shift-limit.md)
Validates the 14-hour on-duty shift window, timer precision requirements, and violation detection.

### 2. [11-Hour Drive Limit](test-strategy-driving.md)
Covers driving time limits, 30-minute break rules, and interaction with shift/cycle timers.

### 3. [Sleeper Berth Provisioning](test-strategy-hos-sleeper-berth.md)
Tests split-sleeper rules (7/3, 8/2 splits), timer pause/reset logic, and edge cases.

## Methodology

### Testing Pyramid Approach
- **60% API Testing:** Core business logic validation
- **30% Database Validation:** Data integrity and state management
- **10% UI Testing:** Critical user journey verification

### Risk Assessment
All strategies include:
- Business impact analysis (DOT citations, OOS orders)
- Technical risk factors (timer precision, edge cases)
- Regulatory compliance requirements (FMCSA rules)

### Quality Gates
- 100% API test pass rate before UI automation
- Database validation for all timer calculations
- Zero-tolerance for regulatory violations

## Business Impact

These test strategies prevent:
- DOT citations ($1,000+ fines per violation)
- Out of Service (OOS) orders
- Regulatory non-compliance
- Driver license suspensions
- Platform decertification

## Tech Stack

- **Languages:** Python
- **Frameworks:** pytest, requests
- **Tools:** Swagger/OpenAPI, Postman
- **Databases:** SQL, MongoDB
- **CI/CD:** GitHub Actions, Docker

## Author

**Saso Trpovski**  
QA Automation Engineer | Python + Playwright + API Testing  
📧 sasotrpovski@gmail.com | 🔗 [LinkedIn](https://www.linkedin.com/in/sasotrpovski/)
