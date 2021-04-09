# Data Model Design

## 1. Active Record Models

### - User

Contains user’s attributes and a *has_many* association to Products.

### - Product

Contains product’s attributes (notably price, an integer representing USD cents), a *belongs_to* association to User and a *has_many* association to Purchases.

The migration for this model requires:

A seller index: <code>t.index ["user_id"], name: "index_products_on_user_id"</code> to efficiently search a seller's products.
A foreign key constraint: <code>add_foreign_key "products", "users"</code> to guarantee data integrity.

### - Account

This model represents an user’s account. Its main attribute is balance which represents the total amount of money in the user’s account. A *belongs_to* association to User and a *has_many* association to Entries, which are the actual money transactions.
