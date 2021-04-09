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
3. A <code>has_many</code> association to BalanceSnapshot, which are historic records of balances at a specific date (see more below).

It also has a constant <code>NON_PAYABLE_PERIOD = 7.days</code> needed to calculate the date until which the user's balance can be payed and can be modified if the payout policy changes.

The migration for this model requires:

- An user index: <code>t.index ["user_id"], name: "index_accounts_on_user_id"</code> to efficiently search an user account.
- A foreign key constraint: <code>add_foreign_key "accounts", "users"</code> to guarantee data integrity.

### Entry

This models represents the actual money transactions. Its main attributes are amount (an integer representing USD cents) and date.

On this models, I propose to implement a narrow and shallow Single Table Inheritance to distinguish to types of account entries: Debit and Credit. This would facilitate the calculation of account balance by simply adding debits and subsctracting credits to the current account's balance or a BalanceSnapshot, depending of what balance you need to calculate.

Its associations are:

1. A <code>belongs_to</code> association to Account.
2. A polymorphic associations to an entriable: <code>belongs_to :entriable, polymorphic: true</code>

I propose this polymorphism to easily expand the integration of other future models that represent money transactions on the user's account. Right now there is only Purchase, Refund and Payout (see bellow). But tomorrow there could be Bonus, Escrow, Gift, etc.

The Entry DB migration might look like this:

```ruby
create_table "entries", force: :cascade do |t|
  t.integer "amount", null: false
  t.date "date", null: false
  t.integer "account_id", null: false
  t.string "type"
  t.string "entriable_type", null: false
  t.integer "entriable_id", null: false
  t.index ["account_id"], name: "index_entries_on_account_id"
  t.index ["entriable_type", "entriable_id"], name: "index_entries_on_entriable"
end

add_foreign_key "entries", "accounts"
```

### Purchase

This model <code>belongs_to</code> a Product and <code>has_one :entry, as: :entriable</code>. It can contain other attributes with exclusively realted to the purchase (acquisition, channel, conversion). It delegates amount and date to Entry.


Refund

Belongs_to a purchase and has_one :entry, as: :entriable. Can contain other attributes with info important to the refund (reason, partial/total). It delegates amount and date, to Entry.

Payout

has_one :entry, as: :entriable.  Can contain other attributes with info important to the refund (reason, partial/total). It delegates amount and date, to Entry..


BalanceSnapshot

To avoid calculating a huge amount of historical entries, a snapshot and a date

Unique index on date to guarantee uniqueness and to search efficiently/fast



Service Objects


BalanceSnapshotService

This class is responsible for taking a balance snapshot on a given date. This service could be run daily.


PaymentService

This class is responsible for paying the user its available balance. Determining the Available balance, as mentioned before, is Account’s responsibility and depends on:

The constant NON_PAYABLE_PERIOD
Perhaps a PaymentPolicy class is needed to approve payments. An example could be a case where total refunds amount after the Non Payable Period is higher than total purchases amounts after the Non Payable Period, therefore we will be paying the user/seller an amount he currently does not have.

Notes

On a real will tackle this with Domain Driven Design, talking with domain experts to progressively distille a deep model with an ubiquitous language… Sorry, I’ve just re-read Eric Evans book :)
I’ll probably create a Billing namespace
Differentiate punctual payments vs recurring payments
Taxes and currencies
The term Purchase was proposed, but maybe the term Sale is better? Since a seller has many sales and the buyer has many purchases? Although Purchase is a more customer centric word



