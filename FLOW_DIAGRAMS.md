# Moneytree Integration - Flow Diagrams

## 1. OAuth 2.0 Authorization Flow

```
┌─────────┐                    ┌──────────────┐                    ┌─────────────┐
│  User   │                    │   Rails App  │                    │  Moneytree │
│         │                    │              │                    │     API    │
└────┬────┘                    └──────┬───────┘                    └─────┬──────┘
     │                                 │                                  │
     │  1. Click "Connect Moneytree"   │                                  │
     ├────────────────────────────────>│                                  │
     │                                 │                                  │
     │                                 │  2. GET /fetch_login_url         │
     │                                 │<─────────────────────────────────┤
     │                                 │                                  │
     │                                 │  3. Build OAuth URL              │
     │                                 │     (client_id, redirect_uri,    │
     │                                 │      scope, state)                │
     │                                 │                                  │
     │  4. Redirect to Moneytree       │                                  │
     │<────────────────────────────────┤                                  │
     │                                 │                                  │
     │  5. User authorizes              │                                  │
     │──────────────────────────────────────────────────────────────────>│
     │                                 │                                  │
     │  6. Redirect with code          │                                  │
     │<──────────────────────────────────────────────────────────────────┤
     │                                 │                                  │
     │  7. POST /update_access_token   │                                  │
     │     { code, grant_type, ... }   │                                  │
     │────────────────────────────────>│                                  │
     │                                 │                                  │
     │                                 │  8. POST /oauth/token            │
     │                                 │─────────────────────────────────>│
     │                                 │                                  │
     │                                 │  9. Return tokens                │
     │                                 │<─────────────────────────────────┤
     │                                 │     { access_token,              │
     │                                 │       refresh_token }             │
     │                                 │                                  │
     │                                 │ 10. Store tokens in DB            │
     │                                 │     (MoneytreeLogin)             │
     │                                 │                                  │
     │ 11. Success response             │                                  │
     │<────────────────────────────────┤                                  │
     │                                 │                                  │
```

## 2. Token Refresh Flow

```
┌─────────────────┐                    ┌──────────────┐                    ┌─────────────┐
│  Scheduled Job  │                    │   Rails App  │                    │  Moneytree │
│   (Rake Task)   │                    │              │                    │     API    │
└────────┬────────┘                    └──────┬───────┘                    └─────┬──────┘
         │                                    │                                  │
         │  1. Find active logins              │                                  │
         │────────────────────────────────────>│                                  │
         │                                    │                                  │
         │                                    │  2. For each login:              │
         │                                    │     - Get refresh_token          │
         │                                    │                                  │
         │                                    │  3. POST /oauth/token            │
         │                                    │     { grant_type: refresh_token, │
         │                                    │       refresh_token, ... }        │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  4. Return new tokens            │
         │                                    │<─────────────────────────────────┤
         │                                    │     { access_token,              │
         │                                    │       refresh_token }             │
         │                                    │                                  │
         │                                    │  5. Update MoneytreeLogin        │
         │                                    │     - Update access_token         │
         │                                    │     - Update refresh_token        │
         │                                    │     - Set token_is_active = true  │
         │                                    │                                  │
         │  6. Continue to next login          │                                  │
         │<────────────────────────────────────┤                                  │
         │                                    │                                  │
```

## 3. Credit Card Synchronization Flow

```
┌─────────────────┐                    ┌──────────────┐                    ┌─────────────┐
│  Scheduled Job  │                    │   Rails App  │                    │  Moneytree │
│                 │                    │              │                    │     API    │
└────────┬────────┘                    └──────┬───────┘                    └─────┬──────┘
         │                                    │                                  │
         │  1. For each active login:          │                                  │
         │     - Get company                   │                                  │
         │────────────────────────────────────>│                                  │
         │                                    │                                  │
         │                                    │  2. FetchCreditCard service      │
         │                                    │     - Initialize for 'personal'  │
         │                                    │     - Initialize for 'corporate' │
         │                                    │                                  │
         │                                    │  3. GET /accounts.json            │
         │                                    │     ?page=1&per_page=500          │
         │                                    │     Authorization: Bearer token  │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  4. Return accounts array         │
         │                                    │<─────────────────────────────────┤
         │                                    │     { accounts: [...] }           │
         │                                    │                                  │
         │                                    │  5. Process each account:         │
         │                                    │     - Filter credit_card type     │
         │                                    │     - Check if exists (account_id)│
         │                                    │     - Create or update            │
         │                                    │                                  │
         │                                    │  6. If more pages:               │
         │                                    │     - Increment page              │
         │                                    │     - Repeat from step 3          │
         │                                    │                                  │
         │                                    │  7. Detect deletions:             │
         │                                    │     - Compare fetched vs existing│
         │                                    │     - Mark missing as deleted     │
         │                                    │     - Set moneytree_deleted_at    │
         │                                    │                                  │
         │  8. Continue to next login          │                                  │
         │<────────────────────────────────────┤                                  │
         │                                    │                                  │
```

## 4. Transaction Synchronization Flow

```
┌─────────────────┐                    ┌──────────────┐                    ┌─────────────┐
│  Scheduled Job  │                    │   Rails App  │                    │  Moneytree │
│                 │                    │              │                    │     API    │
└────────┬────────┘                    └──────┬───────┘                    └─────┬──────┘
         │                                    │                                  │
         │  1. For each credit card:           │                                  │
         │────────────────────────────────────>│                                  │
         │                                    │                                  │
         │                                    │  2. FetchCreditCardStatement      │
         │                                    │     - Get card account_id          │
         │                                    │     - Build transaction URL        │
         │                                    │     - Set date filter (current month)│
         │                                    │                                  │
         │                                    │  3. GET /accounts/{id}/transactions│
         │                                    │     ?page=1&per_page=500           │
         │                                    │     &since=2024-01-01              │
         │                                    │     Authorization: Bearer token    │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  4. Return transactions array      │
         │                                    │<─────────────────────────────────┤
         │                                    │     { transactions: [...] }       │
         │                                    │                                  │
         │                                    │  5. Process each transaction:      │
         │                                    │     - Check if exists (txn_id)     │
         │                                    │     - Create CreditCardStatement   │
         │                                    │       if new                       │
         │                                    │                                  │
         │                                    │  6. If more pages:                 │
         │                                    │     - Increment page                │
         │                                    │     - Repeat from step 3           │
         │                                    │                                  │
         │  7. Continue to next card           │                                  │
         │<────────────────────────────────────┤                                  │
         │                                    │                                  │
```

## 5. Expense Linking Flow

```
┌─────────┐                    ┌──────────────┐                    ┌──────────────┐
│  User   │                    │   Rails App  │                    │   Database   │
│         │                    │              │                    │              │
└────┬────┘                    └──────┬───────┘                    └──────┬───────┘
     │                                 │                                  │
     │  1. View credit card statements │                                  │
     ├────────────────────────────────>│                                  │
     │                                 │                                  │
     │                                 │  2. Query CreditCardStatement     │
     │                                 │     WHERE credit_card_id = X      │
     │                                 │     AND is_deleted = false        │
     │                                 │─────────────────────────────────>│
     │                                 │                                  │
     │                                 │  3. Return statements             │
     │                                 │<─────────────────────────────────┤
     │                                 │                                  │
     │  4. Display statements list     │                                  │
     │<────────────────────────────────┤                                  │
     │                                 │                                  │
     │  5. Select statement to create  │                                  │
     │     expense                     │                                  │
     ├────────────────────────────────>│                                  │
     │                                 │                                  │
     │                                 │  6. Create TExpenseDetail         │
     │                                 │     - Set credit_card_statement_id│
     │                                 │     - Set other expense fields    │
     │                                 │─────────────────────────────────>│
     │                                 │                                  │
     │                                 │  7. Create expense record         │
     │                                 │<─────────────────────────────────┤
     │                                 │                                  │
     │  8. Expense created successfully│                                  │
     │<────────────────────────────────┤                                  │
     │                                 │                                  │
     │  9. Continue with expense report│                                  │
     │     workflow                    │                                  │
     │                                 │                                  │
```

## 6. Cleanup Flow (Deleted Cards)

```
┌─────────────────┐                    ┌──────────────┐                    ┌──────────────┐
│  Scheduled Job  │                    │   Rails App  │                    │   Database   │
│   (Cleanup)     │                    │              │                    │              │
└────────┬────────┘                    └──────┬───────┘                    └──────┬───────┘
         │                                    │                                  │
         │  1. Find deleted cards              │                                  │
         │     (moneytree_exist = false        │                                  │
         │      AND deleted 15+ days ago)      │                                  │
         │────────────────────────────────────>│                                  │
         │                                    │                                  │
         │                                    │  2. Query CreditCard              │
         │                                    │     WHERE moneytree_exist = false │
         │                                    │     AND moneytree_deleted_at <    │
         │                                    │     15.days.ago                   │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  3. Return deleted cards          │
         │                                    │<─────────────────────────────────┤
         │                                    │                                  │
         │                                    │  4. For each card:                │
         │                                    │     - Get all statements          │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  5. Check statement usage:         │
         │                                    │     - Query TExpenseDetail         │
         │                                    │       WHERE credit_card_statement_id│
         │                                    │       IN (statement_ids)           │
         │                                    │     - Check application status     │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  6. Return used statements         │
         │                                    │<─────────────────────────────────┤
         │                                    │                                  │
         │                                    │  7. Calculate unused statements:   │
         │                                    │     unused = all - used            │
         │                                    │                                  │
         │                                    │  8. Delete unused statements:     │
         │                                    │     UPDATE credit_card_statements  │
         │                                    │     SET is_deleted = true          │
         │                                    │     WHERE id IN (unused_ids)       │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  9. Delete card:                  │
         │                                    │     UPDATE credit_cards            │
         │                                    │     SET is_deleted = true          │
         │                                    │     WHERE id = card_id             │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │ 10. Continue to next card           │                                  │
         │<────────────────────────────────────┤                                  │
         │                                    │                                  │
```

## 7. Notification Flow

```
┌─────────────────┐                    ┌──────────────┐                    ┌──────────────┐
│  Scheduled Job  │                    │   Rails App  │                    │ Email Service│
│                 │                    │              │                    │              │
└────────┬────────┘                    └──────┬───────┘                    └──────┬───────┘
         │                                    │                                  │
         │  1. Check token status              │                                  │
         │────────────────────────────────────>│                                  │
         │                                    │                                  │
         │                                    │  2. If token_is_active = false:    │
         │                                    │     - Get company admins           │
         │                                    │     - Prepare expiration email      │
         │                                    │                                    │
         │                                    │  3. Send expire_token_notification │
         │                                    │───────────────────────────────────>│
         │                                    │                                    │
         │                                    │  4. Email sent to admins            │
         │                                    │<───────────────────────────────────┤
         │                                    │                                    │
         │                                    │  5. Check for new statements:       │
         │                                    │     - Get current month statements  │
         │                                    │     - Filter by created_at = today  │
         │                                    │     - Check if not in use           │
         │                                    │                                    │
         │                                    │  6. For each card owner:            │
         │                                    │     - Get pending statements        │
         │                                    │     - Prepare notification email    │
         │                                    │                                    │
         │                                    │  7. Send current_month_pending_      │
         │                                    │     statement notification         │
         │                                    │───────────────────────────────────>│
         │                                    │                                    │
         │                                    │  8. Email sent to card owners       │
         │                                    │<───────────────────────────────────┤
         │                                    │                                    │
         │  9. Continue processing             │                                    │
         │<────────────────────────────────────┤                                    │
         │                                    │                                    │
```

## 8. Error Handling Flow

```
┌─────────────────┐                    ┌──────────────┐                    ┌──────────────┐
│  Scheduled Job  │                    │   Rails App  │                    │  Moneytree │
│                 │                    │              │                    │     API    │
└────────┬────────┘                    └──────┬───────┘                    └─────┬──────┘
         │                                    │                                  │
         │  1. Attempt API call                │                                  │
         │────────────────────────────────────>│                                  │
         │                                    │                                  │
         │                                    │  2. API Request                   │
         │                                    │─────────────────────────────────>│
         │                                    │                                  │
         │                                    │  3. Error Response                │
         │                                    │<─────────────────────────────────┤
         │                                    │     { error: "invalid_token" }     │
         │                                    │                                  │
         │                                    │  4. Check error type:             │
         │                                    │     - Invalid token → Try refresh │
         │                                    │     - Network error → Retry       │
         │                                    │     - Rate limit → Wait & retry   │
         │                                    │                                    │
         │                                    │  5. If token error:                │
         │                                    │     - Attempt token refresh        │
         │                                    │─────────────────────────────────>│
         │                                    │                                    │
         │                                    │  6. If refresh fails:              │
         │                                    │     - Mark token_is_active = false │
         │                                    │     - Log error                     │
         │                                    │     - Send notification            │
         │                                    │                                    │
         │                                    │  7. If refresh succeeds:            │
         │                                    │     - Retry original request        │
         │                                    │─────────────────────────────────>│
         │                                    │                                    │
         │                                    │  8. Success response                │
         │                                    │<─────────────────────────────────┤
         │                                    │                                    │
         │  9. Continue processing             │                                    │
         │<────────────────────────────────────┤                                    │
         │                                    │                                    │
```

## 9. Complete System Flow (End-to-End)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          INITIAL SETUP                                       │
│  1. User authorizes → OAuth flow → Tokens stored                            │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SCHEDULED SYNC (Daily)                                   │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Step 1: Token Refresh                                                │  │
│  │   - Refresh all active tokens                                        │  │
│  │   - Update access_token in database                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Step 2: Sync Credit Cards                                            │  │
│  │   - Fetch card list from Moneytree                                   │  │
│  │   - Create/update card records                                      │  │
│  │   - Mark deleted cards                                              │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Step 3: Sync Transactions                                            │  │
│  │   - For each card: fetch transactions                                │  │
│  │   - Create statement records                                         │  │
│  │   - Filter by current month                                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                              │                                               │
│                              ▼                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ Step 4: Send Notifications                                           │  │
│  │   - Token expiration alerts (if any)                                 │  │
│  │   - New statement notifications                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    USER INTERACTION                                          │
│  - User views credit card statements                                        │
│  - User creates expense linked to statement                                 │
│  - Expense goes through approval workflow                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLEANUP TASK (Daily)                                      │
│  - Find cards deleted 15+ days ago                                          │
│  - Check statement usage                                                    │
│  - Delete unused statements and cards                                       │
└─────────────────────────────────────────────────────────────────────────────┘
```

These flow diagrams illustrate the complete system behavior from OAuth authorization through data synchronization to cleanup operations.

