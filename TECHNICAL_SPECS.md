# Moneytree Integration - Technical Specifications

## API Specifications

### Moneytree API Endpoints

#### 1. OAuth Authorization
```
GET /oauth/authorize
Query Parameters:
  - client_id: Application client ID
  - redirect_uri: Callback URL
  - response_type: "code"
  - scope: "accounts_read transactions_read"
  - state: CSRF protection token
  - country: "JP"
```

#### 2. Token Exchange
```
POST /oauth/token
Headers:
  - Content-Type: application/json
Body:
  {
    "code": "authorization_code",
    "grant_type": "authorization_code",
    "client_id": "client_id",
    "client_secret": "client_secret",
    "redirect_uri": "redirect_uri"
  }
Response:
  {
    "access_token": "token_string",
    "refresh_token": "refresh_token_string",
    "expires_in": 3600,
    "token_type": "Bearer"
  }
```

#### 3. Token Refresh
```
POST /oauth/token
Headers:
  - Content-Type: application/json
Body:
  {
    "grant_type": "refresh_token",
    "client_id": "client_id",
    "client_secret": "client_secret",
    "refresh_token": "refresh_token_string"
  }
Response:
  {
    "access_token": "new_token_string",
    "refresh_token": "new_refresh_token_string",
    "expires_in": 3600
  }
```

#### 4. Fetch Credit Cards
```
GET /accounts.json
Query Parameters:
  - page: Page number (default: 1)
  - per_page: Records per page (max: 500)
Headers:
  - Authorization: Bearer {access_token}
  - Accept: application/json
Response:
  {
    "accounts": [
      {
        "id": "account_id",
        "account_type": "credit_card",
        "account_subtype": "credit_card",
        "institution_account_number": "****-****-****-1234",
        "nickname": "Card Name",
        "current_balance": 1000.00,
        "created_at": "2024-01-01T00:00:00Z",
        "last_aggregated_at": "2024-01-15T10:30:00Z"
      }
    ],
    "pagination": {
      "page": 1,
      "per_page": 500,
      "total_pages": 1
    }
  }
```

#### 5. Fetch Transactions
```
GET /accounts/{account_id}/transactions.json
Query Parameters:
  - page: Page number (default: 1)
  - per_page: Records per page (max: 500)
  - sort_key: "id"
  - sort_by: "asc"
  - since: "2024-01-01" (YYYY-MM-DD format)
Headers:
  - Authorization: Bearer {access_token}
  - Accept: application/json
Response:
  {
    "transactions": [
      {
        "id": 123456789,
        "date": "2024-01-15",
        "amount": -5000.00,
        "description": "Merchant Name",
        "description_pretty": "MERCHANT NAME - TOKYO"
      }
    ],
    "pagination": {
      "page": 1,
      "per_page": 500,
      "total_pages": 1
    }
  }
```

## Database Schema Specifications

### moneytree_logins

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | bigint | PRIMARY KEY | Auto-increment ID |
| company_id | bigint | NOT NULL, UNIQUE, FOREIGN KEY | Reference to companies table |
| token_is_active | boolean | DEFAULT true | Token validity flag |
| access_token | string | | OAuth access token |
| refresh_token | string | | OAuth refresh token |
| created_by | string | | User ID who created |
| updated_by | string | | User ID who last updated |
| created_at | timestamp | NOT NULL | Creation timestamp |
| updated_at | timestamp | NOT NULL | Update timestamp |

**Indexes:**
- `index_moneytree_logins_on_company_id` (UNIQUE)

### credit_cards

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | bigint | PRIMARY KEY | Auto-increment ID |
| moneytree_login_id | bigint | NOT NULL, FOREIGN KEY | Reference to moneytree_logins |
| company_id | bigint | NOT NULL, FOREIGN KEY | Reference to companies table |
| account_id | string | NOT NULL | Moneytree account identifier |
| card_number | string | | Masked card number |
| account_type | string | | Account type from Moneytree |
| account_subtype | string | | Account subtype |
| owner | string | | Card owner/nickname |
| card_source | string | | 'personal' or 'corporate' |
| available_balance | float | | Current card balance |
| moneytree_exist | boolean | DEFAULT true | Existence in Moneytree |
| moneytree_created_at | datetime | | Creation time in Moneytree |
| moneytree_deleted_at | datetime | | Deletion time in Moneytree |
| last_aggregated_at | datetime | | Last sync time |
| is_deleted | boolean | DEFAULT false | Soft delete flag |
| deleted_at | datetime | | Soft delete timestamp |
| created_at | timestamp | NOT NULL | Creation timestamp |
| updated_at | timestamp | NOT NULL | Update timestamp |

**Indexes:**
- `index_credit_cards_on_account_id_and_company_id` (UNIQUE)
- `index_credit_cards_on_company_id`
- `index_credit_cards_on_moneytree_login_id`

### credit_card_statements

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | bigint | PRIMARY KEY | Auto-increment ID |
| credit_card_id | bigint | NOT NULL, FOREIGN KEY | Reference to credit_cards |
| txn_id | bigint | NOT NULL | Moneytree transaction ID |
| use_date | date | | Transaction date |
| amount | decimal(10,2) | | Transaction amount |
| description | text | | Transaction description |
| is_refund | boolean | DEFAULT false | Refund flag |
| tenant_id | integer | | Tenant/company ID |
| is_deleted | boolean | DEFAULT false | Soft delete flag |
| deleted_at | datetime | | Soft delete timestamp |
| created_at | timestamp | NOT NULL | Creation timestamp |
| updated_at | timestamp | NOT NULL | Update timestamp |

**Indexes:**
- `index_credit_card_statements_on_credit_card_id_and_txn_id` (UNIQUE)
- `index_credit_card_statements_on_credit_card_id`

### t_expense_details (existing table, modified)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| ... | ... | | Existing columns |
| credit_card_statement_id | integer | FOREIGN KEY | Reference to credit_card_statements |

**Indexes:**
- `index_t_expense_details_on_credit_card_statement_id`

## Service Object Specifications

### MoneytreeConnection

**Purpose**: HTTP client for Moneytree API

**Methods:**

```ruby
# Initialize connection
def initialize(api_url)
  # Parse URL, setup HTTP client, configure SSL
end

# Build GET request
def get(access_token:)
  # Set Authorization header with Bearer token
  # Return self for method chaining
end

# Build POST request
def post(data)
  # Set Content-Type: application/json
  # Set request body
  # Return self for method chaining
end

# Execute request
def call
  # Execute HTTP request
  # Parse JSON response
  # Return OpenStruct with code and body
  # Handle errors and return nil on failure
end
```

**Error Handling:**
- Network errors → Return nil, log error
- HTTP errors → Return response with error code
- JSON parse errors → Return nil, log error

### FetchCreditCard

**Purpose**: Synchronize credit card list

**Initialization:**
```ruby
def initialize(card_source, moneytree_login)
  # card_source: 'personal' or 'corporate'
  # moneytree_login: MoneytreeLogin instance
  # Setup API URL based on card_source
  # Initialize pagination (page=1, per_page=500)
end
```

**Main Method:**
```ruby
def call
  # 1. Get existing card IDs
  # 2. Loop through paginated API responses
  # 3. Process each account
  # 4. Detect and mark deleted cards
end
```

**Key Logic:**
- Filter accounts where `account_type == 'credit_card'` or `account_subtype == 'credit_card'`
- Format card numbers: mask all but last 4 digits
- Track account IDs to detect deletions
- Update existing cards or create new ones

### FetchCreditCardStatement

**Purpose**: Synchronize transactions for a credit card

**Initialization:**
```ruby
def initialize(credit_card, moneytree_login)
  # credit_card: CreditCard instance
  # moneytree_login: MoneytreeLogin instance
  # Build transaction URL with account_id
  # Set date filter to current month start
  # Initialize pagination
end
```

**Main Method:**
```ruby
def call
  # 1. Loop through paginated API responses
  # 2. For each transaction:
  #    - Check if exists (by txn_id)
  #    - Create CreditCardStatement if new
  # 3. Continue until no more pages
end
```

**Key Logic:**
- Filter by `since=current_month_start` for performance
- Deduplicate by `txn_id`
- Store transaction details (date, amount, description)
- Determine refund status (negative amount = charge, positive = refund)

## Background Job Specifications

### moneytree_api:update_refresh_token

**Schedule**: Daily (configurable)

**Steps:**
1. Find all `MoneytreeLogin` where `token_is_active = true`
2. For each login:
   - Call `fetch_new_refresh_token`
   - Update tokens if successful
   - Mark inactive if refresh fails
3. For each active login:
   - Call `fetch_credit_card_list`
   - Sync personal and corporate cards
4. For each active login:
   - Get all credit cards for company
   - For each card:
     - Call `fetch_credit_card_statement`
5. For each login:
   - If token inactive:
     - Get company admins
     - Send expiration notification
   - For each card:
     - Get current month statements
     - Find pending statements (not in use)
     - Send notification if new statements today

**Error Handling:**
- Token refresh failure → Mark inactive, continue
- API errors → Log and continue with next company
- Network errors → Log and retry on next run

### moneytree_api:remove_moneytree_deleted_card

**Schedule**: Daily (configurable)

**Steps:**
1. Find all `MoneytreeLogin` with companies
2. For each login:
   - Find credit cards where:
     - `moneytree_exist = false`
     - `moneytree_deleted_at < 15.days.ago`
3. For each deleted card:
   - Get all statements for card
   - Find statements linked to expenses:
     - Query `TExpenseDetail` with `credit_card_statement_id`
     - Filter by application status (exclude rejected/cancelled)
   - Calculate unused statements
   - Soft delete unused statements
   - Soft delete card

**Safety Checks:**
- Never delete cards with active expense links
- Preserve statements linked to in-progress expenses
- Preserve statements linked to completed expenses
- Only delete after 15-day grace period

## Model Method Specifications

### MoneytreeLogin

```ruby
# Refresh access token
def fetch_new_refresh_token
  # 1. Check environment variables exist
  # 2. Build refresh token request
  # 3. Call Moneytree API
  # 4. Update tokens if successful
  # 5. Mark inactive if failed
end

# Fetch credit card list
def fetch_credit_card_list
  # 1. Check environment variables
  # 2. Call FetchCreditCard for 'personal'
  # 3. Call FetchCreditCard for 'corporate'
end

# Fetch transactions for card
def fetch_credit_card_statement(credit_card)
  # 1. Check environment variables
  # 2. Call FetchCreditCardStatement service
end
```

### CreditCardStatement

```ruby
# Find statements in progress
def self.fetch_in_progress_statements(statement_ids)
  # Join with expense details and application flows
  # Filter by app_status in [0, 1, 6] (draft, in-progress, revise)
  # Filter by is_wfs_end = false
  # Return statement IDs
end

# Find completed statements
def self.fetch_completed_statements(statement_ids)
  # Join with expense details and application flows
  # Filter by is_wfs_end = true
  # Exclude app_status in [3, 7] (rejected, cancelled)
  # Return statement IDs
end

# Find used statements (any status except rejected/cancelled)
def self.fetch_used_statements(statement_ids)
  # Join with expense details and application flows
  # Exclude app_status in [3, 7]
  # Return statement IDs
end
```

## API Route Specifications

```ruby
# OAuth endpoints
resources :te_account_moneytrees do
  collection do
    get "/fetch_login_url" => "te_account_moneytrees#fetch_login_url"
    post "/update_access_token" => "te_account_moneytrees#update_access_token"
  end
end

# Credit card endpoints
resources :te_credit_cards, only: [:index] do
  # Additional routes for card management
end
```

## Environment Variables

Required environment variables:

```bash
# Moneytree API Configuration
MONEYTREE_API_URL=https://api.moneytree.jp
MONEYTREE_API_CLIENT_ID=your_client_id
MONEYTREE_API_CLIENT_SECRET=your_client_secret
MONEYTREE_API_REDIRECT_URI=https://your-app.com/moneytree/callback

# Moneytree UI URLs
MONEYTREE_ADD_CARD_URL=https://app.moneytree.jp/add-card

# API Endpoint Paths
MONEYTREE_CARD_DETAILS_FETCH_URL=https://api.moneytree.jp/v1
MONEYTREE_PERSONAL_CORPORATE=personal_corporate_path
MONEYTREE_CORPORATE=corporate_path
```

## Performance Specifications

### Pagination
- Default page size: 500 records
- Maximum page size: 500 records
- Handle pagination in loops until no more pages

### Date Filtering
- Transaction sync: Current month only (`since=YYYY-MM-01`)
- Reduces API calls and processing time
- Full history can be fetched on demand

### Batch Processing
- Process companies sequentially
- Process cards sequentially per company
- Process transactions sequentially per card
- Prevents API rate limiting

### Database Queries
- Use eager loading (`includes`) to prevent N+1 queries
- Use indexed lookups for foreign keys
- Use batch updates for multiple records
- Use `pluck` for large result sets

## Error Handling Specifications

### API Errors

| Error Type | Handling Strategy |
|------------|-------------------|
| Invalid Token | Attempt refresh, mark inactive if fails |
| Network Timeout | Retry with exponential backoff (max 3 retries) |
| Rate Limit | Wait and retry (exponential backoff) |
| API Error (4xx) | Log error, skip and continue |
| Server Error (5xx) | Log error, retry on next run |

### Data Errors

| Error Type | Handling Strategy |
|------------|-------------------|
| Duplicate Record | Skip (unique constraint prevents) |
| Missing Required Field | Validation error, skip record |
| Invalid Data Format | Log error, skip record |
| Orphaned Records | Cleanup task handles |

### Business Logic Errors

| Error Type | Handling Strategy |
|------------|-------------------|
| Token Expired | Automatic refresh, notify if fails |
| Card Deleted | Mark for deletion, cleanup after grace period |
| Statement in Use | Prevent deletion, preserve record |

## Security Specifications

### Token Storage
- Store tokens in database (encrypt in production)
- Never log tokens in application logs
- Use environment variables for credentials
- Rotate credentials periodically

### API Communication
- Use HTTPS only
- Verify SSL certificates (in production)
- Use Bearer token authentication
- Implement request signing if required

### Data Access
- Company-scoped data access
- User authorization checks
- Prevent cross-company data access
- Audit logging for sensitive operations

## Monitoring Specifications

### Metrics to Track
- API call success rate
- Token refresh success rate
- Sync duration per company
- Cards synced per run
- Transactions synced per run
- Error rate by type
- Background job execution time

### Logging
- Log all API calls (without sensitive data)
- Log token refresh attempts
- Log synchronization results
- Log errors with full context
- Log performance metrics

### Alerts
- Token refresh failures
- Sync job failures
- High error rates
- API rate limit hits
- Data inconsistency issues

These specifications provide a complete technical reference for implementing the Moneytree integration.

