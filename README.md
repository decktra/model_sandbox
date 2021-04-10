# Data Model Design

![Data Model](https://user-images.githubusercontent.com/53051293/114286465-454a4d00-9a5f-11eb-950c-8987ce452440.jpg)


## 1. Active Record Models

### User

Contains user’s attributes, a `has_many` association to Products and a `has_one` association to Account.

### Product

Contains product’s attributes, notably `price` (an integer representing USD cents), a `belongs_to` association to User and a `has_many` association to Purchases.

The migration for this model requires:

- An user index:`t.index ["user_id"], name: "index_products_on_user_id"` to efficiently search an user's products.
- A foreign key constraint: `add_foreign_key "products", "users"` to guarantee data integrity.

### Account

This model represents an user’s account. Its main attribute is `balance` (an integer for USD cents) which represents the total amount of money in the user’s account. Its associations are:

1. A `belongs_to` association to User.
2. A `has_many` association to Entries, which are the actual money transactions (see more below).
3. A `has_many` association to BalanceSnapshot, which are historic records of balances at a specific date (see more below).

The migration for this model requires:

- An user index: `t.index ["user_id"], name: "index_accounts_on_user_id"` to efficiently search an user account.
- A foreign key constraint: `add_foreign_key "accounts", "users"` to guarantee data integrity.

### Entry

This models represents the actual money transactions. Its main attributes are `amount` (an integer for USD cents) and `date`.

On this model, I propose to implement a narrow and shallow Single Table Inheritance to distinguish to types of account entries: `Debit` and `Credit`. This would facilitate the calculation of a new account's balance by simply adding debits or subsctracting credits to the current account's balance.

Its associations are:

1. A `belongs_to` association to Account.
2. A polymorphic associations to an entriable: `belongs_to :entriable, polymorphic: true`.

I propose this polymorphism to easily expand the integration of other future models that represent money transactions on the user's account. Right now there is only Purchase, Refund and Payout (see definitions below). But in the future there could be Bonus, Escrow, Gift, etc.

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

**Important:** I propose to follow Event Sourcing principles, thus past entries CANNOT be modified. New ones are required to be introduced to amend old ones.

### Purchase

This model `belongs_to` a Product and `has_one :entry, as: :entriable`. It can contain other attributes that are exclusively related to the purchase (quantity, acquisition_channel, conversion_funnel, etc.). It delegates amount and date to Entry.

The migration for this model requires a `product_id` index and foreign key constraint, similar to previous examples.

### Refund

This model `belongs_to` a Purchase and `has_one :entry, as: :entriable`. It can contain other attributes that are exclusively related to the refund (reason, partial/total, etc.). It delegates amount and date to Entry.

The migration for this model requires a `purchase_id` index and foreign key constraint, similar to previous examples.

### Payout

This model represent a transfer of money from the user's Gumroad account to its bank, paypal, etc. It `has_one :entry, as: :entriable` and can contain other attributes that are exclusively related to the payout (destination, channel, etc.). It delegates amount and date to Entry.

**Important:** Given that each "money movement" in our data model requires to create or update multiple records (for example, when a product is bought, both a Purchase and a Debit Entry record are created, and the account's balance is updated), a Commitment Control mechanism is required to guarantee data consistency. We should then wrap these operations in [Active Record Transaction's](https://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html) protective blocks, where SQL statements are only permanent if they can all succeed as one atomic action.

### BalanceSnapshot

To avoid calculating a huge amount of historical entries when a balance in a given point of time is needed, a snapshot of a balance at the end of each date is recorded by this model. Therefore, if Account needs to calculate a balance for a specific time, it only has to get the previous date BalanceSnaphot and add/subsctract all debits/credits (entries) of that date.

An unique index on date/account_id is required to guarantee uniqueness and to search efficiently/fast: `t.index ["account_id", "date"], name: "index_balance_snapshots_on_account_id_and_date", unique: true`.


## 2. Service Objects

### BalanceSnapshotTaker

This service class is responsible for taking a snapshot of an account's ending balance on a given date. To achive this, it takes as arguments `account_id` and `date`, it gets the previous date BalanceSnaphot, and finally it adds/subsctracts all debits/credits (entries) of that specific account for that specific date.

In order to store a daily BalanceSnapshots for each account, a Period Job (such as [SideKiq's cron jobs](https://github.com/mperham/sidekiq/wiki/Ent-Periodic-Jobs)) could be configure to daily:

1. Loop through all accounts to create indivual jobs with arguments `account_id` and yesterday's `date` to be process by Sidekiq.
2. These individual jobs will then run the `BalanceSnapshotTaker` service and calculate each account's previous date ending balance, saving a `BalanceSnapshot` record in the DB.

Given that the number of sellers will be in the millions, to avoid this period job process to start crashing due to memory issues, we can use Active Records's `find_each` and specify a batch size to iterate over all users in a more memory efficient way.

Also, those individual account/date Sidekiq jobs should be idempotent (i.e. if you run the same account/date worker, you will create or update the same DB record).

### PaymentSender

This service class is responsible for paying the user its payable balance. To achive this, it takes as argument `account_id`, it uses a constant `NON_PAYABLE_PERIOD = 7.days` to determine the payable date (7 days back from today), it find the account's BalanceSnaphot amount for that date, and finally it creates a Payout record and its respective Credit Entry with that amount.

The `NON_PAYABLE_PERIOD` constant can be modified if the payout policy eventually changes.


In order to pay all sellers, a Period Job (such as [SideKiq's cron jobs](https://github.com/mperham/sidekiq/wiki/Ent-Periodic-Jobs)) could be configure to biweekly (according to this exercise description):

1. Loop through all accounts to create indivual jobs with argument `account_id` to be process by Sidekiq.
2. These individual jobs will then run the `PaymentSender` service for all account's that have a positive `BalanceSnapshot` amount for that payable date.

The Period Job frecuency can be modified if the payout policy eventually changes.

Again, given that the number of sellers will be in the millions, to avoid this job process to start crashing due to memory issues, we can use Active Records's `find_each` and specify a batch size to iterate over all users in a more memory efficient way.

At this point, a `PaymentPolicy` class might be needed to approve payments. An example could be a case where total refunds amount after the `NON_PAYABLE_PERIOD` is higher than total purchases amounts after the `NON_PAYABLE_PERIOD`, therefore we will be paying the user/seller an amount he currently does not have. For instance, in an extreme case, we would pay the sellers on Friday the 10th for all their sales up to Friday the 3rd, but on the 7th most purchases were refunded.

Since we have an associaton Refund `belongs_to` Purchase (see Purchase above), we could easily create a Purchase `has_many` Refunds association and only pay part of the purchase that has not been refunded (if partials refunds exists) or not pay the purchase at all (if total refund took place). In any case, for the purpose of this exercise's limited time, I will not detail this important policy/validation consideration and I assume a negative `Account#balance` is technically possible. 

## 3. Notes

- On a real scenario, I will tackle this with Domain Driven Design, talking with domain experts to progressively distille a deep model with an ubiquitous language. 
- I’ll probably also create a Billing namespace to modularize the app.
- This model as it is, should be extensible to differentiate punctual payments vs recurring payments, and incorporate taxes and currencies.
- The term Purchase was proposed, but maybe the term Sale is better? Since a seller has many sales and the buyer has many purchases?


