# Moneytree API Integration - Financial Data Aggregation System

## 📋 Overview

This project demonstrates the implementation of a **Moneytree API integration** for a Rails-based expense management system. Moneytree is a financial data aggregation service that allows applications to securely access users' financial account information, including credit card transactions.

## 🎯 Project Objectives

- **Automated Financial Data Sync**: Integrate with Moneytree API to automatically fetch credit card data and transactions
- **OAuth 2.0 Authentication**: Implement secure token-based authentication flow
- **Real-time Data Synchronization**: Sync credit card statements and transactions in real-time
- **Expense Management Integration**: Link credit card transactions to expense reports for automated reimbursement processing
- **Background Job Processing**: Implement scheduled tasks for data synchronization and cleanup

## 🏗️ Architecture Overview

```
┌─────────────────┐
│   Frontend UI   │
│  (User Actions) │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│      Rails Application              │
│  ┌──────────────────────────────┐  │
│  │  OAuth Controller            │  │
│  │  - Authorization Flow        │  │
│  │  - Token Management         │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │  Service Layer              │  │
│  │  - MoneytreeConnection      │  │
│  │  - FetchCreditCard          │  │
│  │  - FetchCreditCardStatement │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │  Background Jobs (Rake)      │  │
│  │  - Token Refresh            │  │
│  │  - Data Sync                │  │
│  │  - Cleanup Tasks            │  │
│  └──────────────────────────────┘  │
└────────┬───────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│      Moneytree API                  │
│  - OAuth Endpoints                  │
│  - Account Data                     │
│  - Transaction Data                 │
└─────────────────────────────────────┘
```

## 🔑 Key Features

### 1. OAuth 2.0 Authentication Flow
- Secure authorization code flow
- Access token and refresh token management
- Automatic token refresh mechanism
- Token expiration handling

### 2. Credit Card Data Synchronization
- Fetch personal and corporate credit cards
- Real-time card list updates
- Handle card additions and deletions
- Track card lifecycle

### 3. Transaction Data Management
- Fetch credit card statements/transactions
- Pagination support for large datasets
- Current month transaction filtering
- Deduplication by transaction ID

### 4. Expense Integration
- Link credit card statements to expense details
- Track statement usage status (pending/in-progress/completed)
- Prevent deletion of statements in use
- Automated expense report generation

### 5. Background Processing
- Scheduled token refresh
- Automated data synchronization
- Cleanup of deleted cards (15-day grace period)
- Email notifications for admins and users

## 📊 Database Schema

### Core Models

**MoneytreeLogin**
- Stores OAuth tokens per company
- Tracks token validity status
- Manages refresh token lifecycle

**CreditCard**
- Represents credit cards from Moneytree
- Links to company and moneytree login
- Tracks card existence in Moneytree
- Stores card metadata (number, type, balance)

**CreditCardStatement**
- Individual credit card transactions
- Links to credit card and expense details
- Stores transaction data (date, amount, description)
- Tracks usage status

**TExpenseDetail**
- Expense records that can link to credit card statements
- Part of expense reimbursement workflow

## 🔄 Data Flow

1. **Initial Setup**: User authorizes application → OAuth tokens stored
2. **Scheduled Sync**: Background job fetches credit cards → Creates/updates records
3. **Transaction Sync**: Background job fetches transactions → Creates statement records
4. **User Action**: User creates expense → Links to credit card statement
5. **Cleanup**: Background job removes deleted cards after grace period

## 🛠️ Technology Stack

- **Backend**: Ruby on Rails
- **API Integration**: RESTful API with OAuth 2.0
- **Background Jobs**: Rake tasks with scheduling (e.g., whenever gem)
- **Database**: PostgreSQL (assumed)
- **Authentication**: OAuth 2.0 Authorization Code Flow

## 📁 Project Structure

```
app/
├── controllers/
│   └── api/
│       └── te_account_moneytrees_controller.rb  # OAuth endpoints
├── models/
│   ├── moneytree_login.rb                       # Token management
│   ├── credit_card.rb                           # Card model
│   └── credit_card_statement.rb                 # Transaction model
├── services/
│   ├── moneytree_connection.rb                  # API client
│   ├── fetch_credit_card.rb                     # Card sync service
│   └── fetch_credit_card_statement.rb           # Transaction sync service
└── workers/
    └── (background job processing)

lib/
└── tasks/
    └── moneytree_api.rake                       # Scheduled tasks

config/
├── routes.rb                                     # API routes
└── schedule.rb                                   # Job scheduling
```

## 🔐 Security Considerations

- OAuth 2.0 secure token exchange
- HTTPS-only API communication
- Token storage in database (consider encryption)
- Environment variable management for credentials
- Token refresh to minimize exposure window
- Graceful error handling and logging

## 📈 Performance Optimizations

- Pagination for large datasets (500 records per page)
- Current month transaction filtering
- Batch processing in background jobs
- Efficient database queries with proper indexing
- Deduplication to prevent duplicate records

## 🚀 Getting Started

See [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md) for detailed step-by-step implementation instructions.

## 📚 Documentation

- [Architecture Details](./ARCHITECTURE.md) - Detailed system architecture
- [Implementation Guide](./IMPLEMENTATION_GUIDE.md) - Step-by-step implementation
- [API Flow Diagrams](./FLOW_DIAGRAMS.md) - Visual flow representations
- [Technical Specifications](./TECHNICAL_SPECS.md) - Technical details

## 🎓 Learning Outcomes

This project demonstrates:
- OAuth 2.0 implementation
- Third-party API integration
- Background job processing
- Data synchronization patterns
- Error handling and retry logic
- Database design for financial data
- Scheduled task management
- Email notification systems

## 📝 License

This documentation is for portfolio/resume purposes only. Implementation details are based on real-world experience with financial API integrations.

