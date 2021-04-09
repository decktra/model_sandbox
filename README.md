# Data Model Design


## 1. Active Record Models

### User

Contains user’s attributes, a `has_many` association to Products and a `has_one` association to Account.

### Product

Contains product’s attributes, notably `price` (an integer representing USD cents), a `belongs_to` association to User and a `has_many` association to Purchases.

The migration for this model requires:

- An user index:`t.index ["user_id"], name: "index_products_on_user_id"` to efficiently search an user's products.
- A foreign key constraint: `add_foreign_key "products", "users"` to guarantee data integrity.

### Account

This model represents an user’s account. Its main attribute is `balance` (an integer representing USD cents) which represents the total amount of money in the user’s account. Its associations are:

1. A `belongs_to` association to User.
2. A `has_many` association to Entries, which are the actual money transactions (see more below).
3. A `has_many` association to BalanceSnapshot, which are historic records of balances at a specific date (see more below).

The migration for this model requires:

- An user index: `t.index ["user_id"], name: "index_accounts_on_user_id"` to efficiently search an user account.
- A foreign key constraint: `add_foreign_key "accounts", "users"` to guarantee data integrity.

### Entry

This models represents the actual money transactions. Its main attributes are `amount` (an integer representing USD cents) and `date`.

On this model, I propose to implement a narrow and shallow Single Table Inheritance to distinguish to types of account entries: `Debit` and `Credit`. This would facilitate the calculation of account balance by simply adding debits and subsctracting credits to the current account's balance or a BalanceSnapshot, depending of what balance you need to calculate.

Its associations are:

1. A `belongs_to` association to Account.
2. A polymorphic associations to an entriable: `belongs_to :entriable, polymorphic: true`.

I propose this polymorphism to easily expand the integration of other future models that represent money transactions on the user's account. Right now there is only Purchase, Refund and Payout (see definition below). But tomorrow there could be Bonus, Escrow, Gift, etc.

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

**Important:** Being faithful to Event Sourcing, past entries CANNOT be modified. New ones are required to be introduced to amend old ones.

### Purchase

This model `belongs_to` a Product and `has_one :entry, as: :entriable`. It can contain other attributes that are exclusively related to the purchase (acquisition_channel, conversion_funnel, etc.). It delegates amount and date to Entry.

The migration for this model requires a product_id index and foreign key constraint, similar to previous examples.

### Refund

This model `belongs_to` a Purchase and `has_one :entry, as: :entriable`. It can contain other attributes that are exclusively related to the refund (reason, partial/total, etc.). It delegates amount and date to Entry.

The migration for this model requires a purchase_id index and foreign key constraint, similar to previous examples.

### Payout

This model represent a transfer of money from the user's Gumroad account to its bank, paypal, etc. It `has_one :entry, as: :entriable` and can contain other attributes that are exclusively related to the payout (destination, etc.). It delegates amount and date to Entry.

### BalanceSnapshot

To avoid calculating a huge amount of historical entries when a balance in a given point of time is needed, a snapshot of a balance at the end of each date is recorded by this model. Therefore, if Account needs to calculate a balance for a specific time, it only has to get the previous date BalanceSnaphot and add/subsctract all debits/credits (entries) of that day.

An unique index on date/account_id is required to guarantee uniqueness and to search efficiently/fast: `t.index ["account_id", "date"], name: "index_balance_snapshots_on_account_id_and_date", unique: true`.



## 2. Service Objects

### BalanceSnapshotTaker

This class is responsible for taking a balance snapshot on a given date. To achive this, it takes as arguments `account_id` and `date`, it get the previous date BalanceSnaphot and add/subsctract all debits/credits (entries) of that day.

A Period Job (such as [SideKiq's](https://github.com/mperham/sidekiq/wiki/Ent-Periodic-Jobs)) could be run daily to loops through all accounts, creating in its turn indivual jobs with arguments `account_id` and yesterday's `date` to be process by Sidekiq. These individual jobs will then run this `BalanceSnapshotTaker` service  and calculate each account's previous date ending balance, saving a `BalanceSnapshot` record in the DB.

Given the number of sellers will be in the millions, to avoid this job process to start crashing due to memory issues, we can use Active Records's `find_each` and specify a batch size to iterate over all users in a more memory efficient way.

Also, each account balance snapshot should be taken in a individual Sidekiq job (i.e. one Sidekiq job per account/date) and jobs should be idempotent (i.e. if you run the same account/date you will create or update the same DB record).

### PaymentSender

This class is responsible for paying the user its payable balance. It has a constant `NON_PAYABLE_PERIOD = 7.days` needed to calculate the date until which the user's balance can be payed. The payable balance is then the account's BalanceSnapshot for the date `NON_PAYABLE_PERIOD` days back.

The `NON_PAYABLE_PERIOD` constant can be modified if the payout policy eventually changes.

A Period Job (such as [SideKiq's](https://github.com/mperham/sidekiq/wiki/Ent-Periodic-Jobs)) could be set at the appropiate frecuencyrun daily with to loop through all accounts and calculate the previous date ending balance.

Given the number of sellers will be in the millions, to avoid this job process to start crashing due to memory issues, we can use Active Records's `find_each` and specify a batch size to iterate over all users in a more memory efficient way.

Also, each account balance snapshot should be taken in a individual Sidekiq job (i.e. one Sidekiq job per account/date) and jobs should be idempotent (i.e. if you run the same account/date you will create or update the same DB record).


At this point, a `PaymentPolicy` class might be needed to approve payments. An example could be a case where total refunds amount after the `NON_PAYABLE_PERIOD` is higher than total purchases amounts after the `NON_PAYABLE_PERIOD`, therefore we will be paying the user/seller an amount he currently does not have. For instance, in an extreme case, we would pay the sellers on Friday the 10th for all their sales up to Friday the 3rd, but on the 7th most purchases were refunded.

Since we have an associaton Refund `belongs_to` Purchase (see Purchase above), we could easily create a Purchase `has_many` Refunds association and only pay part of the purchase that has not been refunded (if partials refunds exists) or not pay the purchase at all (if total refund took place). In any case, for the purpose of this exercise's limited time, I will not detail this important policy/validation consideration and I assume a negative `Account#balance` is technically possible. 

Determining the Available balance, as mentioned before, is Account’s responsibility and depends on:

The constant 


Notes

On a real will tackle this with Domain Driven Design, talking with domain experts to progressively distille a deep model with an ubiquitous language… Sorry, I’ve just re-read Eric Evans book :)
I’ll probably create a Billing namespace
Differentiate punctual payments vs recurring payments
Taxes and currencies
The term Purchase was proposed, but maybe the term Sale is better? Since a seller has many sales and the buyer has many purchases? Although Purchase is a more customer centric word



