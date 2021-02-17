# Introduction to Money SQL
![Build Status](http://sweatbox.noexpectations.com.au:8080/buildStatus/icon?job=money_sql)
[![Hex pm](http://img.shields.io/hexpm/v/ex_money_sql.svg?style=flat)](https://hex.pm/packages/ex_money_sql)
[![License](https://img.shields.io/badge/license-Apache%202-blue.svg)](https://github.com/kipcole9/money_sql/blob/master/LICENSE)

Money_SQL implements a set of functions to store and retrieve data structured as a `%Money{}` type that is composed of an ISO 4217 currency code and a currency amount. See [ex_money](https://hex.pm/packages/ex_money) for details of using `Money`.  Note that `ex_money_sql` depends on `ex_money`.

## Prerequisities

* `Money_SQL` is supported on Elixir 1.6 and later only

## Serializing to a Postgres database with Ecto

`Money_SQL` provides custom Ecto data types and a custom Postgres data type to provide serialization of `Money.t` types without losing precision whilst also maintaining the integrity of the `{currency_code, amount}` relationship.  To serialise and retrieve money types from a database the following steps should be followed:

1. First generate the migration to create the custom type:

```elixir
mix money.gen.postgres.money_with_currency_migration
* creating priv/repo/migrations
* creating priv/repo/migrations/20161007234652_add_money_with_currency_type_to_postgres.exs
```

2. Then migrate the database:

```elixir
mix ecto.migrate
07:09:28.637 [info]  == Running MoneyTest.Repo.Migrations.AddMoneyWithCurrencyTypeToPostgres.up/0 forward
07:09:28.640 [info]  execute "CREATE TYPE public.money_with_currency AS (currency_code char(3), amount numeric)"
07:09:28.647 [info]  == Migrated in 0.0s
```

3. Create your database migration with the new type (don't forget to `mix ecto.migrate` as well):

```elixir
defmodule MoneyTest.Repo.Migrations.CreateLedger do
  use Ecto.Migration

  def change do
    create table(:ledgers) do
      add :amount, :money_with_currency
      timestamps()
    end
  end
end
```

4. Create your schema using the `Money.Ecto.Composite.Type` ecto type:

```elixir
defmodule Ledger do
  use Ecto.Schema

  schema "ledgers" do
    field :amount, Money.Ecto.Composite.Type

    timestamps()
  end
end
```

5. Insert into the database:

```elixir
iex> Repo.insert %Ledger{amount: Money.new(:USD, "100.00")}
[debug] QUERY OK db=4.5ms
INSERT INTO "ledgers" ("amount","inserted_at","updated_at") VALUES ($1,$2,$3)
[{"USD", #Decimal<100.00>}, {{2016, 10, 7}, {23, 12, 13, 0}}, {{2016, 10, 7}, {23, 12, 13, 0}}]
```

6. Retrieve from the database:

```elixir
iex> Repo.all Ledger
[debug] QUERY OK source="ledgers" db=5.3ms decode=0.1ms queue=0.1ms
SELECT l0."amount", l0."inserted_at", l0."updated_at" FROM "ledgers" AS l0 []
[%Ledger{__meta__: #Ecto.Schema.Metadata<:loaded, "ledgers">, amount: #<:USD, 100.00>,
  inserted_at: ~N[2017-02-21 00:15:40.979576],
  updated_at: ~N[2017-02-21 00:15:40.991391]}]
```

## Serializing to a MySQL (or other non-Postgres) database with Ecto

Since MySQL does not support composite types, the `:map` type is used which in MySQL is implemented as a `JSON` column.  The currency code and amount are serialised into this column.

    defmodule MoneyTest.Repo.Migrations.CreateLedger do
      use Ecto.Migration

      def change do
        create table(:ledgers) do
          add :amount, :map
          timestamps()
        end
      end
    end

Create your schema using the `Money.Ecto.Map.Type` ecto type:

    defmodule Ledger do
      use Ecto.Schema

      schema "ledgers" do
        field :amount, Money.Ecto.Map.Type

        timestamps()
      end
    end

Insert into the database:

    iex> Repo.insert %Ledger{amount_map: Money.new(:USD, 100)}
    [debug] QUERY OK db=25.8ms
    INSERT INTO "ledgers" ("amount_map","inserted_at","updated_at") VALUES ($1,$2,$3)
    RETURNING "id" [%{amount: "100", currency: "USD"},
    {{2017, 2, 21}, {0, 15, 40, 979576}}, {{2017, 2, 21}, {0, 15, 40, 991391}}]

    {:ok,
     %MoneyTest.Thing{__meta__: #Ecto.Schema.Metadata<:loaded, "ledgers">,
      amount: nil, amount_map: #Money<:USD, 100>, id: 3,
      inserted_at: ~N[2017-02-21 00:15:40.979576],
      updated_at: ~N[2017-02-21 00:15:40.991391]}}

Retrieve from the database:

    iex> Repo.all Ledger
    [debug] QUERY OK source="ledgers" db=16.1ms decode=0.1ms
    SELECT t0."id", t0."amount_map", t0."inserted_at", t0."updated_at" FROM "ledgers" AS t0 []
    [%Ledger{__meta__: #Ecto.Schema.Metadata<:loaded, "ledgers">,
      amount_map: #Money<:USD, 100>, id: 3,
      inserted_at: ~N[2017-02-21 00:15:40.979576],
      updated_at: ~N[2017-02-21 00:15:40.991391]}]

### Notes:

1.  In order to preserve precision of the decimal amount, the amount part of the `%Money{}` struct is serialised as a string. This is done because JSON serializes numeric values as either `integer` or `float`, neither of which would preserve precision of a decimal value.

2.  The precision of the serialized string value of amount is affected by the setting of `Decimal.get_context`.  The default is 28 digits which should cater for your requirements.

3.  Serializing the amount as a string means that SQL query arithmetic and equality operators will not work as expected.  You may find that `CAST`ing the string value will restore some of that functionality.  For example:

```sql
CAST(JSON_EXTRACT(amount_map, '$.amount') AS DECIMAL(20, 8)) AS amount;
```

## Postgres Database functions

Since the datatype used to store `Money` in Postgres is a composite type (called `:money_with_currency`), the standard aggregation functions like `sum` and `average` are not supported and the `order_by` clause doesn't perform as expected.  `Money` provides mechanisms to provide these functions.

### Aggregate functions: sum()

`Money` provides a migration generator which, when migrated to the database with `mix ecto.migrate`, supports performing `sum()` aggregation on `Money` types. The steps are:

1. Generate the migration by executing `mix money.gen.postgres.aggregate_functions`

2. Migrate the database by executing `mix ecto.migrate`

3. Formulate an Ecto query to use the aggregate function `sum()`

```elixir
  # Formulate the query.  Note the required use of the type()
  # expression which is needed to inform Ecto of the return
  # type of the function
  iex> q = Ecto.Query.select Item, [l], type(sum(l.price), l.price)
  #Ecto.Query<from l in Item, select: type(sum(l.price), l.price)>
  iex> Repo.all q
  [debug] QUERY OK source="items" db=6.1ms
  SELECT sum(l0."price")::money_with_currency FROM "items" AS l0 []
  [#Money<:USD, 600>]
```

The function `Repo.aggregate/3` can also be used. However at least [ecto version 3.2.4](https://hex/pm/packages/ecto/3.2.4) is required for this to work correctly for custom ecto types such as `:money_with_currency`.

```elixir
  iex> Repo.aggregate(Item, :sum, :price)
  #Money<:USD, 600>
```

**Note** that to preserve the integrity of `Money` it is not permissable to aggregate money that has different currencies.  If you attempt to aggregate money with different currencies the query will abort and an exception will be raised:
```elixir
  iex> Repo.all q
  [debug] QUERY ERROR source="items" db=4.5ms
  SELECT sum(l0."price")::money_with_currency FROM "items" AS l0 []
  ** (Postgrex.Error) ERROR 22033 (): Incompatible currency codes. Expected all currency codes to be USD
```

### Order_by with Money

Since `:money_with_currency` is a composite type, the default `order_by` results may surprise since the ordering is based upon the type structure, not the money amount.  Postgres defines a means to access the components of a composite type and therefore sorting can be done in a more predictable fashion.  For example:
```elixir
  # In this example we are decomposing the the composite column called
  # `price` and using the sub-field `amount` to perform the ordering.
  iex> q = from l in Item, select: l.price, order_by: fragment("amount(price)")
  #Ecto.Query<from l in Item, order_by: [asc: fragment("amount(price)")],
   select: l.amount>
  iex> Repo.all q
  [debug] QUERY OK source="items" db=2.0ms
  SELECT l0."price" FROM "items" AS l0 ORDER BY amount(price) []
  [#Money<:USD, 100.00000000>, #Money<:USD, 200.00000000>,
   #Money<:USD, 300.00000000>, #Money<:AUD, 300.00000000>]
```
**Note** that the results may still be unexpected.  The example above shows the correct ascending ordering by `amount(price)` however the ordering is not currency code aware and therefore mixed currencies will return a largely meaningless order.

## Installation

`Money` can be installed by adding `ex_money_sql` to your list of dependencies in `mix.exs` and then executing `mix deps.get`

```elixir
def deps do
  [
    {:ex_money_sql, "~> 1.0"},
    ...
  ]
end
```
