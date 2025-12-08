# Moneytree Integration - Step-by-Step Implementation Guide

## Prerequisites

- Ruby on Rails application
- PostgreSQL database
- Understanding of OAuth 2.0 flow
- Moneytree API credentials (Client ID, Client Secret, Redirect URI)

## Step 1: Understanding Moneytree API

### What is Moneytree?

Moneytree is a financial data aggregation platform (similar to Plaid, Yodlee) that provides:
- Secure access to financial accounts
- Credit card transaction data
- Account balance information
- OAuth 2.0 authentication

### API Endpoints Overview

1. **Authorization**: `GET /oauth/authorize` - User authorization
2. **Token Exchange**: `POST /oauth/token` - Get access/refresh tokens
3. **Accounts**: `GET /accounts` - Fetch credit card list
4. **Transactions**: `GET /accounts/{account_id}/transactions` - Fetch transactions

## Step 2: Database Schema Design

### 2.1 Create MoneytreeLogin Table

**Purpose**: Store OAuth tokens per company/organization

```ruby
# Migration: create_moneytree_logins.rb
create_table :moneytree_logins do |t|
  t.references :company, null: false, foreign_key: true
  t.boolean :token_is_active, default: true
  t.string :access_token
  t.string :refresh_token
  t.string :created_by
  t.string :updated_by
  t.timestamps
end

add_index :moneytree_logins, :company_id, unique: true
```

**Key Fields**:
- `access_token`: Short-lived token for API calls
- `refresh_token`: Long-lived token for refreshing access token
- `token_is_active`: Flag to track token validity

### 2.2 Create CreditCard Table

**Purpose**: Store credit cards synced from Moneytree

```ruby
# Migration: create_credit_cards.rb
create_table :credit_cards do |t|
  t.references :moneytree_login, null: false, foreign_key: true
  t.references :company, null: false, foreign_key: true
  t.string :account_id, null: false  # Moneytree account ID
  t.string :card_number
  t.string :account_type
  t.string :account_subtype
  t.string :owner
  t.string :card_source  # 'personal' or 'corporate'
  t.float :available_balance
  t.boolean :moneytree_exist, default: true
  t.datetime :moneytree_created_at
  t.datetime :moneytree_deleted_at
  t.datetime :last_aggregated_at
  t.boolean :is_deleted, default: false
  t.datetime :deleted_at
  t.timestamps
end

add_index :credit_cards, [:account_id, :company_id], unique: true
add_index :credit_cards, :company_id
```

**Key Fields**:
- `account_id`: Unique identifier from Moneytree
- `moneytree_exist`: Tracks if card still exists in Moneytree
- `moneytree_deleted_at`: Timestamp when card was deleted from Moneytree

### 2.3 Create CreditCardStatement Table

**Purpose**: Store individual credit card transactions

```ruby
# Migration: create_credit_card_statements.rb
create_table :credit_card_statements do |t|
  t.references :credit_card, null: false, foreign_key: true
  t.bigint :txn_id, null: false  # Moneytree transaction ID
  t.date :use_date
  t.decimal :amount, precision: 10, scale: 2
  t.text :description
  t.boolean :is_refund, default: false
  t.integer :tenant_id
  t.boolean :is_deleted, default: false
  t.datetime :deleted_at
  t.timestamps
end

add_index :credit_card_statements, [:credit_card_id, :txn_id], unique: true
add_index :credit_card_statements, :credit_card_id
```

**Key Fields**:
- `txn_id`: Unique transaction identifier from Moneytree
- `use_date`: Transaction date
- `amount`: Transaction amount (negative for charges)

### 2.4 Link to Expense Details

```ruby
# Migration: add_credit_card_statement_id_to_expense_details.rb
add_column :t_expense_details, :credit_card_statement_id, :integer
add_index :t_expense_details, :credit_card_statement_id
```

## Step 3: Model Implementation

### 3.1 MoneytreeLogin Model

**Responsibilities**:
- Store and manage OAuth tokens
- Refresh access tokens
- Fetch credit card data
- Fetch transaction data

**Key Methods**:
```ruby
class MoneytreeLogin < ActiveRecord::Base
  belongs_to :company
  has_many :credit_cards
  
  validates_uniqueness_of :company_id
  
  # Refresh access token using refresh token
  def fetch_new_refresh_token
    # Implementation: Call Moneytree API to refresh token
  end
  
  # Fetch all credit cards for this company
  def fetch_credit_card_list
    # Implementation: Sync cards from Moneytree
  end
  
  # Fetch transactions for a specific card
  def fetch_credit_card_statement(credit_card)
    # Implementation: Sync transactions for card
  end
end
```

### 3.2 CreditCard Model

```ruby
class CreditCard < ActiveRecord::Base
  belongs_to :moneytree_login
  belongs_to :company
  has_many :credit_card_statements
  has_one :credit_card_user  # Links card to user
  
  validates :card_number, presence: true
  validates :account_id, uniqueness: { scope: :company_id }
end
```

### 3.3 CreditCardStatement Model

```ruby
class CreditCardStatement < ActiveRecord::Base
  belongs_to :credit_card
  has_one :t_expense_detail  # Links to expense record
  
  validates :txn_id, uniqueness: { scope: :credit_card_id }
  
  # Class methods for tracking statement usage
  def self.fetch_in_progress_statements(statement_ids)
    # Find statements linked to in-progress expenses
  end
  
  def self.fetch_completed_statements(statement_ids)
    # Find statements linked to completed expenses
  end
end
```

## Step 4: API Connection Service

### 4.1 MoneytreeConnection Service

**Purpose**: Generic HTTP client for Moneytree API

**Implementation Pattern**:
```ruby
class MoneytreeConnection
  def initialize(api_url)
    @uri = URI.parse(api_url)
    @http = Net::HTTP.new(@uri.host, @uri.port)
    @http.use_ssl = @uri.scheme == 'https'
  end
  
  def get(access_token:)
    # Build GET request with Bearer token
  end
  
  def post(data)
    # Build POST request with JSON body
  end
  
  def call
    # Execute request and return response
  end
end
```

**Key Features**:
- Handles both GET and POST requests
- Manages SSL connections
- Error handling and logging
- Returns structured response objects

## Step 5: OAuth 2.0 Flow Implementation

### 5.1 Authorization Endpoint

**Controller Action**: `fetch_login_url`

**Purpose**: Generate OAuth authorization URL

**Flow**:
1. Build authorization URL with:
   - `client_id`
   - `redirect_uri`
   - `response_type=code`
   - `scope=accounts_read transactions_read`
   - `state` (for CSRF protection)
2. Return URL to frontend
3. User redirects to Moneytree for authorization

### 5.2 Token Exchange Endpoint

**Controller Action**: `update_access_token`

**Purpose**: Exchange authorization code for tokens

**Flow**:
1. Receive authorization code from callback
2. POST to `/oauth/token` with:
   - `code`
   - `grant_type=authorization_code`
   - `client_id`
   - `client_secret`
   - `redirect_uri`
3. Receive `access_token` and `refresh_token`
4. Store tokens in `MoneytreeLogin` record
5. Set `token_is_active = true`

**Error Handling**:
- Invalid code → Return error
- Token exchange failure → Log and notify
- Missing credentials → Return error

## Step 6: Data Synchronization Services

### 6.1 FetchCreditCard Service

**Purpose**: Sync credit card list from Moneytree

**Implementation Steps**:

1. **Initialize Service**
   - Determine card source (personal/corporate)
   - Set up API endpoint URL
   - Configure pagination (page=1, per_page=500)

2. **Fetch Cards in Loop**
   ```ruby
   loop do
     response = fetch_moneytree_response
     break if response.blank? || response.code != '200'
     
     process_credit_card_data(response)
     break if no_more_pages?
     @page += 1
   end
   ```

3. **Process Each Card**
   - Check if card exists (by `account_id`)
   - Update existing or create new
   - Track account IDs for deletion detection

4. **Handle Deletions**
   - Compare fetched cards with existing cards
   - Mark missing cards as `moneytree_exist = false`
   - Set `moneytree_deleted_at = Time.now`

**Key Logic**:
- Only process cards with `account_type == 'credit_card'`
- Format card numbers (mask all but last 4 digits)
- Store card metadata (balance, owner, type)

### 6.2 FetchCreditCardStatement Service

**Purpose**: Sync transactions for a specific credit card

**Implementation Steps**:

1. **Initialize Service**
   - Set up transaction endpoint URL
   - Configure pagination
   - Set date filter (current month only)

2. **Fetch Transactions**
   ```ruby
   loop do
     response = fetch_moneytree_response
     break if response.blank? || no_transactions?
     
     response.body['transactions'].each do |txn|
       create_statement_if_new(txn)
     end
     @page += 1
   end
   ```

3. **Create Statement Records**
   - Check if transaction exists (by `txn_id`)
   - Create new `CreditCardStatement` if not exists
   - Store transaction details

**Key Logic**:
- Filter by `since=current_month_start` for performance
- Deduplicate by `txn_id`
- Store amount, date, description
- Determine if refund (negative amount)

## Step 7: Background Job Implementation

### 7.1 Token Refresh Task

**Purpose**: Automatically refresh access tokens

**Implementation**:
```ruby
namespace :moneytree_api do
  task update_refresh_token: :environment do
    MoneytreeLogin.where(token_is_active: true).each do |login|
      login.fetch_new_refresh_token
    end
  end
end
```

**Scheduling**: Run daily (e.g., via whenever gem)

### 7.2 Data Synchronization Task

**Purpose**: Sync cards and transactions

**Implementation**:
```ruby
task update_refresh_token: :environment do
  # 1. Refresh tokens
  MoneytreeLogin.where(token_is_active: true).each do |login|
    login.fetch_new_refresh_token
  end
  
  # 2. Sync card list
  MoneytreeLogin.where(token_is_active: true).each do |login|
    login.fetch_credit_card_list
  end
  
  # 3. Fetch transactions for each card
  MoneytreeLogin.includes(:company).each do |login|
    next unless login.token_is_active
    
    CreditCard.where(company_id: login.company_id).each do |card|
      login.fetch_credit_card_statement(card)
    end
  end
end
```

### 7.3 Cleanup Task

**Purpose**: Remove cards deleted from Moneytree (after grace period)

**Implementation**:
```ruby
task remove_moneytree_deleted_card: :environment do
  MoneytreeLogin.includes(:company).each do |login|
    # Find cards deleted 15+ days ago
    deleted_cards = CreditCard.where(
      company_id: login.company_id,
      moneytree_exist: false
    ).where('moneytree_deleted_at < ?', 15.days.ago)
    
    deleted_cards.each do |card|
      # Only delete if statements not in use
      unused_statements = find_unused_statements(card)
      delete_card_and_statements(card, unused_statements)
    end
  end
end
```

**Safety Logic**:
- Check if statements are linked to expenses
- Only delete unused statements
- Preserve cards with active expense links

## Step 8: Email Notifications

### 8.1 Token Expiration Notification

**Trigger**: When `token_is_active = false`

**Recipients**: Company admins

**Content**:
- Token expiration notice
- Re-authorization link
- Instructions for re-linking

### 8.2 Pending Statement Notification

**Trigger**: New statements added for current month

**Recipients**: Card owners

**Content**:
- List of new pending statements
- Statement details (date, amount, description)
- Link to expense creation page

## Step 9: Integration with Expense System

### 9.1 Link Statement to Expense

**User Flow**:
1. User views credit card statements
2. Selects statement to create expense
3. System creates `TExpenseDetail` with `credit_card_statement_id`
4. Expense report workflow continues

**Implementation**:
```ruby
# In expense creation
expense_detail = TExpenseDetail.create(
  # ... other fields
  credit_card_statement_id: statement.id
)
```

### 9.2 Track Statement Usage

**Status Tracking**:
- **Pending**: Not linked to any expense
- **In Progress**: Linked to expense in draft/in-progress
- **Completed**: Linked to approved expense
- **Rejected**: Linked to rejected expense (can be reused)

**Query Methods**:
```ruby
# Find in-progress statements
def self.fetch_in_progress_statements(statement_ids)
  TExpenseDetail
    .joins(t_expense_report: [t_form_detail: :t_application_flow])
    .where(t_application_flow: { app_status: [0, 1, 6], is_wfs_end: false })
    .where(credit_card_statement_id: statement_ids)
    .pluck(:credit_card_statement_id)
end
```

## Step 10: Error Handling & Edge Cases

### 10.1 API Errors

**Handling**:
- Network timeouts → Retry with exponential backoff
- Invalid tokens → Mark as inactive, notify admin
- Rate limiting → Implement rate limit handling
- API changes → Version API calls

### 10.2 Data Consistency

**Issues**:
- Duplicate cards → Unique constraint on `account_id`
- Duplicate transactions → Unique constraint on `txn_id`
- Orphaned records → Cleanup tasks

### 10.3 Token Management

**Scenarios**:
- Token expired → Refresh automatically
- Refresh failed → Mark inactive, require re-auth
- Multiple refresh attempts → Prevent concurrent refreshes

## Step 11: Testing Strategy

### 11.1 Unit Tests

- Model validations
- Service method logic
- Token refresh logic
- Data processing methods

### 11.2 Integration Tests

- OAuth flow end-to-end
- API connection service
- Data synchronization
- Background job execution

### 11.3 Mocking

- Mock Moneytree API responses
- Test error scenarios
- Test pagination
- Test token refresh

## Step 12: Deployment Considerations

### 12.1 Environment Variables

Required variables:
- `MONEYTREE_API_URL`
- `MONEYTREE_API_CLIENT_ID`
- `MONEYTREE_API_CLIENT_SECRET`
- `MONEYTREE_API_REDIRECT_URI`
- `MONEYTREE_ADD_CARD_URL`
- `MONEYTREE_CARD_DETAILS_FETCH_URL`
- `MONEYTREE_PERSONAL_CORPORATE`
- `MONEYTREE_CORPORATE`

### 12.2 Database Migrations

- Run migrations in order
- Add indexes for performance
- Consider data migration for existing records

### 12.3 Background Jobs

- Set up job scheduler (whenever, sidekiq, etc.)
- Configure job frequency
- Monitor job execution
- Set up alerts for failures

## Step 13: Monitoring & Maintenance

### 13.1 Logging

- Log all API calls
- Log token refresh attempts
- Log synchronization results
- Log errors with context

### 13.2 Metrics

- Track sync success rate
- Monitor token expiration
- Track API response times
- Monitor background job performance

### 13.3 Alerts

- Token refresh failures
- Sync job failures
- API error rate spikes
- Data inconsistency issues

## Summary

This implementation provides:
- ✅ Secure OAuth 2.0 authentication
- ✅ Automated data synchronization
- ✅ Robust error handling
- ✅ Background job processing
- ✅ Email notifications
- ✅ Integration with expense system
- ✅ Data cleanup and maintenance

The system is production-ready with proper error handling, logging, and monitoring capabilities.

