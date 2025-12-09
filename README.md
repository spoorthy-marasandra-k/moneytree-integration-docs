# Moneytree API Integration - Documentation Index

This repository contains documentation for a **Moneytree API integration** project.
The goal of the project is to synchronize financial data, such as credit card accounts and transactions, into an expense management system.

This documentation focuses on system design, architecture, and technical decisions.
No proprietary code is included.

### The Main Docs
- **[MONEYTREE_INTEGRATION_README.md](./MONEYTREE_INTEGRATION_README.md)**
High-level project overview, system behavior, and feature summary.

- **[IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)**
Step-by-step implementation notes and development workflow.

- **[ARCHITECTURE.md](./ARCHITECTURE.md)**
Detailed explanation of the architecture, components, and design decisions.

- **[FLOW_DIAGRAMS.md](./FLOW_DIAGRAMS.md)** 
Process flow diagrams, including the OAuth flow, synchronization flow, and cleanup processes.

- **[TECHNICAL_SPECS.md](./TECHNICAL_SPECS.md)**
API specifications, data structures, and database schema details.

### Technical Skills I Learned/Demonstrated
 - System Design and Architecture
 - OAuth 2.0 authorization flow
 - REST API integration
 - Background processing
 - Token lifecycle management
 - Scheduled synchronization
 - Error handling and retry logic
 - Data cleanup and maintenance

### Implementation Highlights
 - Automatic credit card and transaction synchronization
 - Statement-to-expense linking
 - Secure storage of OAuth tokens
 - Daily scheduled jobs for refresh and sync
 - Grace-period handling for deleted or outdated data
 - Notifications for expiring tokens or new statements

### Architecture Overview (Simplified)

```
User → OAuth Flow → Token Storage → API Client → Moneytree API
                                                      ↓
Background Jobs ← Database ← Data Sync ← API Response
```

## What This Project Does

This integration connects a Rails expense management app with the Moneytree financial data aggregation API. Here's what it accomplishes:

- **Automatically syncs credit card data** from Moneytree
- **Imports transactions** for expense management
- **Links transactions to expenses** for automated reimbursement workflow
- **Handles OAuth 2.0 authentication** securely
- **Processes data in background jobs** for better performance
- **Sends notifications** when important stuff happens


## Tech Stack

- **Backend**: Ruby on Rails (what I was working with)
- **Database**: PostgreSQL
- **API**: RESTful API, OAuth 2.0
- **Background Jobs**: Rake tasks (scheduled with whenever gem)
- **HTTP Client**: Net::HTTP (Ruby's built-in library)

## What I Learned

This project really helped me understand:
- Third-party API integration (the good, the bad, and the ugly)
- OAuth 2.0 security implementation (tokens, refresh, all that jazz)
- Background job processing (scheduling, error handling)
- Database design for financial data (relationships, constraints, indexes)
- Error handling and recovery (retry logic, graceful degradation)
- Performance optimization (pagination, filtering, batching)
- System architecture design (service layer, separation of concerns)
