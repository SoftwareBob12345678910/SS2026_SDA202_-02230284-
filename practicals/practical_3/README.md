# Practical 3 — Plagiarism Checker System: Class & Object Diagram Analysis

---

## Overview

This practical presents Object-Oriented Design analysis of an automated Plagiarism Checker and Grading System through UML class and object diagrams. The system manages the complete assignment lifecycle: submission, automated testing, plagiarism detection, grading, and LMS integration.

---

## Part 1: Class Diagram Analysis

### System Components (10 Classes)

**Actors:**

`Student` - Submit assignments, view grades
 
 `Professor` - Create assignments, review reports, approve grades

**Task Management:**

`Assignment` - Assignment details, deadline, test suite

`Submission` - Student work, submission status tracking

`Grade` - Automated and manual grading with approve workflow

**Quality Assurance:**

`TestResult` - Automated test execution metrics

`PlagiarismReport` - Aggregates plagiarism analysis from peer and external sources

**External Integration:**

`TurnItInService` - Interfaces with external plagiarism detection

`TurnItInData` - Plagiarism detection results

`LMSIntegration` - Grades exported to university system

`AuditTrail` - Compliance logging for all actions

### Key Relationships

```
Student --|> Submission: submits
Professor --|> Assignment: creates
Assignment --|> Submission: has
Submission --|> TestResult: produces
Submission --|> PlagiarismReport: generates
Submission --|> Grade: evaluated by
PlagiarismReport --|> TurnItInData: uses
Grade --|> LMSIntegration: exported to
LMSIntegration --|> AuditTrail: logs
```

### Design Patterns Applied


**Repository Pattern**: Submission aggregates related work data

**Service Pattern**: TurnItInService and LMSIntegration abstract external systems

**Observer Pattern**: AuditTrail logs all state changes

**Strategy Pattern**: Multiple grading strategies (automated vs. manual override)

---

## Part 2: Object Diagram Analysis

### Scenario: Two Student Submissions

**Context:**

Course: SDA 202

Assignment: Library System Programming

Deadline: April 15, 2026

Actors: Karma N (STU001), Tashi D (STU002), Dr. Pema (PROF001)

### Karma N's Submission (SUB001)

| Component | Result |
|-----------|--------|
| **Submission Time** | 2026-04-10 (on-time) |
| **Tests Passed** | 18/20 (90%) |
| **Self-Plagiarism** | 5% |
| **External Plagiarism** | 12% |
| **Automated Grade** | 90.0 |
| **Final Grade** | 90.0 (no override) |
| **Status** | Approved |

### Tashi's Submission (SUB002)

| Component | Result |
|-----------|--------|
| **Submission Time** | 2026-04-11 (on-time) |
| **Tests Passed** | 20/20 (100%) |
| **Self-Plagiarism** | 2% |
| **External Plagiarism** | 8% |
| **Automated Grade** | 100.0 |
| **Manual Override** | 95.0 (documentation penalty) |
| **Final Grade** | 95.0 |
| **Status** | Approved |

### Processing Pipeline

```
Submission → Parallel Processing (Tests + Plagiarism Check) → 
Professor Review → Grade Decision → LMS Export → Audit Logging
```

**Key Observations:**

Parallel processing reduces evaluation time

Professor retains override authority despite high automated grade

Both grades exported with complete audit trail

System maintains individual pipelines while centralizing oversight

---

## Part 3: System Capabilities

### Automated Testing

Executes predefined test suites

Captures pass/fail metrics and execution time

Provides objective grading foundation

### Multi-Stage Plagiarism Detection

**Stage 1**: Peer comparison within class

**Stage 2**: External TurnItIn verification

Combined analysis prevents false positives/negatives

### Intelligent Grading

Automatic calculation from test results

Professor override for plagiarism penalties or special circumstances

Maintains both automated and final grades for transparency

### Compliance & Integration

LMS integration for grade synchronization

Immutable audit trail for regulatory review

Complete action logging for dispute resolution

---

## Part 4: Design Quality

### SOLID Principles

| Principle | Implementation |
|-----------|-----------------|
| **Single Responsibility** | Each class has one clear purpose |
| **Open/Closed** | Extensible for new detection algorithms |
| **Liskov Substitution** | Services interchangeable with minimal impact |
| **Interface Segregation** | Classes implement only required operations |
| **Dependency Inversion** | High-level modules depend on abstractions |

### Scalability & Security


**Scalability**: Parallel processing, asynchronous operations, indexed data queries

**Security**: Encrypted API credentials, role-based access control, immutable audit logs

---

## Part 5: Implementation Notes

### Technology Recommendations

**Backend**: Java/Spring Boot

**Database**: PostgreSQL (ACID compliance)

**Processing**: RabbitMQ (asynchronous queuing)

**Caching**: Redis

### Key Metrics

Average processing time: <5 hours per submission

System availability: 99.9% uptime

Plagiarism detection accuracy

LMS sync success rate

---

## Conclusion

The Plagiarism Checker System demonstrates a well-balanced object-oriented architecture:

Clear class responsibilities with defined relationships

Realistic runtime scenarios validating the design

Extensible through design patterns and abstraction

Compliant with regulatory audit requirements

Scalable for institutional deployment

The class diagram provides implementation blueprint while the object diagram validates real-world workflow efficiency and effectiveness.

---

**Practical**: 3 | **Topic**: Class & Object Diagram Design | **Date**: April 6, 2026

