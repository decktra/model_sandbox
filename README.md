# Data Model Design

## 1. Active Record Models

### User

Contains user’s attributes, a <code>has_many</code> association to Products and a <code>has_one</code> association to Account.

### Product

Contains product’s attributes, notably price (an integer representing USD cents), a <code>belongs_to</code> association to User and a <code>has_many</code> association to Purchases.

The migration for this model requires:

- An user index: <code>t.index ["user_id"], name: "index_products_on_user_id"</code> to efficiently search an user's products.
- A foreign key constraint: <code>add_foreign_key "products", "users"</code> to guarantee data integrity.

### Account

This model represents an user’s account. Its main attribute is balance (an integer representing USD cents) which represents the total amount of money in the user’s account. Its associations are:

1. A <code>belongs_to</code> association to User.
2. A <code>has_many</code> association to Entries, which are the actual money transactions (see more below).
3. A <code>has_many</code> association to BalanceSnapshot, which is a historic records of balances at a specific date (see more below).

It also has a constant <code>NON_PAYABLE_PERIOD = 7.days</code> needed to calculate the date until which the user's balance can be payed.

The migration for this model requires:

- An user index: <code>t.index ["user_id"], name: "index_accounts_on_user_id"</code> to efficiently search an user account.
- A foreign key constraint: <code>add_foreign_key "accounts", "users"</code> to guarantee data integrity.

### Entry

This models represents the actual money transactions. Its main attributes are amount (an integer representing USD cents) and date. I propose to implement a narrow and shallow Single Table Inheritance to distinguish to types of account entries: Debit and Credit. This would facilitate the calculation of account balance by simply adding debits and subsctracting credits to the current account's balance or a BalanceSnapshot, depending of what balance you need to calculate.

Its associations are:

1. A <code>belongs_to</code> association to Account.
2. A polymorphic associations to an entriable: <code>belongs_to :entriable, polymorphic: true</code>

I propose this polymorphism to easily expand the integration of other future models that represent money transactions. Right now the is only Purchase, Refund and Payout. But tomorrow there couild be Bonus, Escrows, Gift, etc.


