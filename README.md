# Data Model Design

## 1. Active Record Models

### - User

Contains user’s attributes, a <code>has_many</code> association to Products and a <code>has_one</code> association to Account.

### - Product

Contains product’s attributes, notably price (an integer representing USD cents), a <code>belongs_to</code> association to User and a <code>has_many</code> association to Purchases.

The migration for this model requires:

- A seller index: <code>t.index ["user_id"], name: "index_products_on_user_id"</code> to efficiently search an user's products.
- A foreign key constraint: <code>add_foreign_key "products", "users"</code> to guarantee data integrity.

### - Account

This model represents an user’s account. Its main attribute is balance (an integer representing USD cents) which represents the total amount of money in the user’s account. Its associations are:

1. A <code>belongs_to</code> association to User.
2. A <code>has_many</code> association to Entries, which are the actual money transactions (see more below).
3. A <code>has_many</code> association to BalanceSnapshot, which is a historic records of balances at a specific date (see more below).

The migration for this model requires:

- A seller index: <code>t.index ["user_id"], name: "index_accounts_on_user_id"</code> to efficiently search an user account.
- A foreign key constraint: <code>add_foreign_key "accounts", "users"</code> to guarantee data integrity.
