# Moneytree Integration - System Architecture

## System Overview

The Moneytree integration is a financial data aggregation system that connects a Rails expense management application with the Moneytree API to automatically sync credit card data and transactions.

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    User Interface Layer                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ Authorization│  │ Card Display │  │ Expense Link │     │
│  │   Flow UI    │  │     UI       │  │     UI       │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Application Layer (Rails)                   │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Controller Layer                         │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ Api::TeAccountMoneytreesController             │  │  │
│  │  │ - fetch_login_url                              │  │  │
│  │  │ - update_access_token                          │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ TeCreditCardsController                         │  │  │
│  │  │ - List cards, statements                       │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Service Layer                             │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ MoneytreeConnection                            │  │  │
│  │  │ - HTTP client for Moneytree API                │  │  │
│  │  │ - GET/POST request handling                    │  │  │
│  │  │ - SSL/TLS management                           │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ FetchCreditCard                                │  │  │
│  │  │ - Sync card list from Moneytree                │  │  │
│  │  │ - Handle pagination                            │  │  │
│  │  │ - Detect card deletions                        │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ FetchCreditCardStatement                       │  │  │
│  │  │ - Sync transactions for cards                  │  │  │
│  │  │ - Filter by date range                         │  │  │
│  │  │ - Deduplicate transactions                     │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Model Layer                              │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │  │
│  │  │MoneytreeLogin│  │ CreditCard  │  │ Statement  │ │  │
│  │  │              │  │              │  │            │ │  │
│  │  │- OAuth tokens│  │- Card data  │  │- Transaction│ │  │
│  │  │- Token mgmt  │  │- Metadata   │  │  data      │ │  │
│  │  └──────────────┘  └──────────────┘  └────────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │          Background Job Layer (Rake Tasks)            │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ moneytree_api:update_refresh_token             │  │  │
│  │  │ - Refresh OAuth tokens                         │  │  │
│  │  │ - Sync card list                               │  │  │
│  │  │ - Fetch transactions                           │  │  │
│  │  │ - Send notifications                           │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  │  ┌────────────────────────────────────────────────┐  │  │
│  │  │ moneytree_api:remove_moneytree_deleted_card    │  │  │
│  │  │ - Cleanup deleted cards                        │  │  │
│  │  │ - Remove unused statements                     │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    Data Layer (PostgreSQL)                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │moneytree_    │  │ credit_      │  │ credit_card_ │     │
│  │  logins      │  │   cards      │  │  statements  │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│  ┌──────────────┐  ┌──────────────┐                       │
│  │ companies    │  │ t_expense_   │                       │
│  │              │  │   details    │                       │
│  └──────────────┘  └──────────────┘                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  External Services                           │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Moneytree API                            │  │
│  │  - OAuth endpoints                                   │  │
│  │  - Account data endpoints                            │  │
│  │  - Transaction endpoints                             │  │
│  └──────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              Email Service (ActionMailer)            │  │
│  │  - Token expiration notifications                    │  │
│  │  - Pending statement notifications                   │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

**Api::TeAccountMoneytreesController**

Responsibilities:
- Handle OAuth authorization flow
- Manage token exchange
- Provide authorization URLs

Key Actions:
- `fetch_login_url`: Generate OAuth authorization URL
- `update_access_token`: Exchange code for tokens

### 2. Service Layer

#### MoneytreeConnection

**Purpose**: Generic HTTP client for Moneytree API

**Responsibilities**:
- Build HTTP requests (GET/POST)
- Handle SSL/TLS connections
- Manage request headers
- Parse JSON responses
- Error handling and logging

**Design Pattern**: Service Object

#### FetchCreditCard

**Purpose**: Synchronize credit card list from Moneytree

**Responsibilities**:
- Fetch cards via paginated API calls
- Create/update card records
- Detect deleted cards
- Handle personal and corporate cards separately

**Key Features**:
- Pagination support (500 per page)
- Deduplication by account_id
- Soft delete tracking

#### FetchCreditCardStatement

**Purpose**: Synchronize transactions for credit cards

**Responsibilities**:
- Fetch transactions for specific card
- Filter by date range (current month)
- Create statement records
- Deduplicate by transaction ID

**Key Features**:
- Date filtering for performance
- Pagination support
- Transaction deduplication

### 3. Model Layer

#### MoneytreeLogin

**Relationships**:
- `belongs_to :company`
- `has_many :credit_cards`

**Key Attributes**:
- `access_token`: Short-lived API token
- `refresh_token`: Long-lived refresh token
- `token_is_active`: Validity flag

**Key Methods**:
- `fetch_new_refresh_token`: Refresh access token
- `fetch_credit_card_list`: Sync cards
- `fetch_credit_card_statement(cc)`: Sync transactions

#### CreditCard

**Relationships**:
- `belongs_to :moneytree_login`
- `belongs_to :company`
- `has_many :credit_card_statements`
- `has_one :credit_card_user`

**Key Attributes**:
- `account_id`: Moneytree account identifier
- `card_number`: Masked card number
- `moneytree_exist`: Existence flag
- `moneytree_deleted_at`: Deletion timestamp

#### CreditCardStatement

**Relationships**:
- `belongs_to :credit_card`
- `has_one :t_expense_detail`

**Key Attributes**:
- `txn_id`: Moneytree transaction identifier
- `use_date`: Transaction date
- `amount`: Transaction amount
- `description`: Transaction description

**Key Methods**:
- `fetch_in_progress_statements`: Find statements in use
- `fetch_completed_statements`: Find completed statements
- `fetch_used_statements`: Find all used statements

### 4. Background Job Layer

#### Token Refresh & Data Sync Task

**Schedule**: Daily (configurable)

**Steps**:
1. Refresh tokens for active logins
2. Sync credit card list
3. Fetch transactions for all cards
4. Send email notifications

**Error Handling**:
- Token refresh failures → Mark inactive, notify admin
- API errors → Log and continue with next company
- Network errors → Retry with exponential backoff

#### Cleanup Task

**Schedule**: Daily (configurable)

**Purpose**: Remove cards deleted from Moneytree

**Logic**:
- Find cards marked as deleted 15+ days ago
- Check if statements are in use
- Delete only unused statements
- Soft delete cards

**Safety**:
- Never delete cards with active expense links
- Preserve statements linked to expenses
- Grace period prevents accidental deletion

## Data Flow Architecture

### OAuth Flow

```
User → Frontend → Controller → Moneytree API
                              ↓
                         Authorization
                              ↓
User ← Frontend ← Controller ← Access Token
```

### Data Synchronization Flow

```
Scheduled Job → MoneytreeLogin → FetchCreditCard → Moneytree API
                                                      ↓
                                              Credit Card Data
                                                      ↓
Scheduled Job ← Database ← Service ← Process & Store
```

### Transaction Sync Flow

```
Scheduled Job → CreditCard → FetchCreditCardStatement → Moneytree API
                                                          ↓
                                                    Transaction Data
                                                          ↓
Scheduled Job ← Database ← Service ← Process & Store
```

## Security Architecture

### Authentication Flow

1. **User Authorization**
   - User clicks "Connect Moneytree"
   - Redirected to Moneytree OAuth page
   - User authorizes application
   - Redirected back with authorization code

2. **Token Exchange**
   - Application exchanges code for tokens
   - Tokens stored in database (encrypted recommended)
   - Access token used for API calls
   - Refresh token used to renew access token

3. **Token Management**
   - Access tokens expire (short-lived)
   - Refresh tokens used to get new access tokens
   - Automatic refresh via background jobs
   - Manual re-authorization if refresh fails

### Data Security

- **Token Storage**: Encrypted in database
- **API Communication**: HTTPS only
- **Error Logging**: No sensitive data in logs
- **Access Control**: Company-scoped data access

## Scalability Considerations

### Database Optimization

- **Indexes**: On foreign keys and lookup fields
- **Unique Constraints**: Prevent duplicates
- **Soft Deletes**: Preserve data for audit

### API Rate Limiting

- **Pagination**: Limit records per request
- **Date Filtering**: Only fetch current month
- **Batch Processing**: Process companies sequentially
- **Error Handling**: Graceful degradation

### Background Jobs

- **Scheduling**: Configurable frequency
- **Error Recovery**: Retry logic
- **Monitoring**: Job execution tracking
- **Notifications**: Alert on failures

## Integration Points

### Expense System Integration

```
CreditCardStatement → TExpenseDetail
                          ↓
                    Expense Report
                          ↓
                    Approval Workflow
```

### Notification System

```
Token Expiration → Email Service → Admin Users
New Statements → Email Service → Card Owners
```

## Error Handling Architecture

### Error Types

1. **API Errors**
   - Network failures → Retry with backoff
   - Invalid tokens → Refresh or re-authorize
   - Rate limiting → Wait and retry
   - API changes → Version handling

2. **Data Errors**
   - Duplicate records → Unique constraints
   - Missing data → Validation checks
   - Orphaned records → Cleanup tasks

3. **Business Logic Errors**
   - Token expiration → Notify admin
   - Sync failures → Log and alert
   - Data inconsistencies → Validation

### Error Recovery

- **Automatic**: Token refresh, retry logic
- **Manual**: Admin notifications, re-authorization
- **Monitoring**: Logging, alerts, metrics

## Performance Architecture

### Optimization Strategies

1. **Data Fetching**
   - Pagination (500 records per page)
   - Date filtering (current month only)
   - Selective field fetching

2. **Database Queries**
   - Eager loading (includes)
   - Indexed lookups
   - Batch operations

3. **Background Processing**
   - Asynchronous jobs
   - Scheduled execution
   - Parallel processing (where safe)

### Caching Strategy

- **Token Caching**: Store in memory (with expiration)
- **Card List Caching**: Cache for short duration
- **Statement Caching**: Cache pending statements

## Monitoring Architecture

### Metrics to Track

1. **API Performance**
   - Response times
   - Success/failure rates
   - Rate limit hits

2. **Data Sync**
   - Cards synced per run
   - Transactions synced per run
   - Sync duration

3. **Token Management**
   - Token refresh success rate
   - Token expiration events
   - Re-authorization frequency

### Logging Strategy

- **API Calls**: Log all requests/responses
- **Errors**: Log with full context
- **Business Events**: Token refresh, sync completion
- **Performance**: Track slow operations

## Deployment Architecture

### Environment Configuration

- **Development**: Local Moneytree sandbox
- **Staging**: Moneytree test environment
- **Production**: Moneytree production API

### Job Scheduling

- **Tool**: whenever gem or cron
- **Frequency**: Configurable per environment
- **Monitoring**: Job execution logs

### Database Migrations

- **Order**: Sequential execution
- **Rollback**: Safe rollback strategy
- **Data Migration**: Handle existing data

This architecture provides a robust, scalable, and maintainable integration with the Moneytree API while ensuring data security and system reliability.

