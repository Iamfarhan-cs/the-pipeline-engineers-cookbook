# Recipe 12 — Finding Migrations

Database migrations are one of the most important parts of a Data Engineering repository.

They explain how the database changed over time.

If you are investigating a pipeline and find a table that does not look the way you expected, the migration history is often the missing piece.

A migration can tell you:

- when a table was created
- when a column was added
- when a constraint changed
- when an index was introduced
- when a column was renamed or removed
- when data was transformed during a schema change
- and sometimes why the change was made

This chapter is about learning how to find that history and connect it to the code that uses the database.

---

## 1. What Is a Database Migration?

A migration is a controlled change to a database schema.

For example, imagine a table initially contains:

```text
users
-----
id
name
email
```

Later, the application needs to store account status.

A migration might add:

```text
status
```

The important point is that the database is not changed manually without a record.

The migration records the change so another environment can reproduce it.

A simple lifecycle looks like this:

```text
Migration file
      |
      v
Migration tool
      |
      v
Database schema
      |
      v
Application code
```

Migrations are therefore both **database changes** and **historical records of schema evolution**.

---

## 2. Why Migration Investigation Matters

Suppose you are asked:

> Why does this table contain this column?

The application code may not answer the question.

A migration might.

Suppose you find this sequence:

```text
001_create_orders
002_add_customer_id
003_add_status
004_add_created_at
005_add_index_on_customer_id
```

Now you have a timeline.

You can understand that the table did not always have its current structure.

This matters when:

- debugging production problems
- adding a new feature
- changing an existing table
- investigating missing data
- understanding old code
- preparing a backfill
- reviewing a schema change
- restoring a database
- reproducing an environment
- investigating migration failures
- and understanding why a constraint exists

Migration investigation is especially important when working in an unfamiliar repository.

---

## 3. Where Migrations Usually Live

There is no single universal migration directory.

Common locations include:

```text
migrations/
db/migrations/
database/migrations/
sql/migrations/
alembic/
prisma/migrations/
flyway/
```

Some projects keep migrations close to application code.

Others keep them in a separate database directory.

Do not assume the directory name.

Search the repository.

Useful search terms include:

```text
migration
migrations
schema
alembic
flyway
liquibase
prisma
knex
sequelize
upgrade
downgrade
revision
```

Also inspect configuration files and dependency files. They can reveal which migration system the repository uses.

---

## 4. First Identify the Migration Tool

Before reading individual migration files, identify the tool.

Different tools organize migrations differently.

Examples include:

- Alembic for Python projects
- Flyway
- Liquibase
- Prisma Migrate
- Django migrations
- Rails Active Record migrations
- Knex migrations
- custom SQL migration systems

These are examples, not a statement that a particular repository uses one of them.

Look for evidence in the repository.

For example:

```text
requirements.txt
pyproject.toml
package.json
pom.xml
build.gradle
docker-compose.yml
Makefile
scripts/
```

Then look for migration commands.

Generic examples:

```bash
alembic upgrade head

flyway migrate

python manage.py migrate
```

These commands are examples only. Use the command actually configured by the repository.

---

## 5. Start With the Migration Directory

Once you know where migrations are stored, inspect the directory structure.

A repository might look like:

```text
project/
├── src/
├── tests/
├── migrations/
│   ├── 001_create_users.sql
│   ├── 002_create_orders.sql
│   ├── 003_add_order_status.sql
│   └── 004_add_customer_index.sql
├── Dockerfile
└── README.md
```

Do not immediately read every line.

First build a map.

Ask:

1. How are migrations numbered?
2. Are they SQL files or application code?
3. Is there one migration per change?
4. Is there a migration metadata table?
5. Are migrations reversible?
6. Are there separate upgrade and downgrade operations?
7. Is there a naming convention?
8. Are there data migrations as well as schema migrations?

These answers tell you how the project manages schema history.

---

## 6. Read Migrations in Order

Migration order matters.

Suppose you find:

```text
001_create_payments
002_add_currency
003_add_status
004_create_payment_events
005_add_payment_event_index
```

Reading only migration 005 may not tell you enough.

You need to understand what existed before it.

A useful approach is:

```text
Migration 001
    ↓
Migration 002
    ↓
Migration 003
    ↓
Migration 004
    ↓
Migration 005
    ↓
Current schema
```

Think of the migration chain as a timeline.

Each migration modifies the state created by the previous migrations.

---

## 7. Find the Migration That Created the Table

When investigating a table, start by finding its creation migration.

Suppose the application uses a table called:

```text
payment_events
```

Search for:

```text
CREATE TABLE payment_events
```

You may find:

```sql
CREATE TABLE payment_events (
    event_id UUID PRIMARY KEY,
    event_name TEXT NOT NULL,
    occurred_at TIMESTAMP NOT NULL
);
```

Now you know the starting structure.

Next, search for later migrations that mention the same table.

```text
ALTER TABLE payment_events
```

This gives you the evolution of the table.

---

## 8. Follow ALTER TABLE Changes

Many schema changes appear as `ALTER TABLE` statements.

Common operations include:

```sql
ALTER TABLE orders ADD COLUMN status TEXT;
```

```sql
ALTER TABLE orders DROP COLUMN legacy_status;
```

```sql
ALTER TABLE orders ADD CONSTRAINT ...;
```

```sql
ALTER TABLE orders ALTER COLUMN amount ...;
```

When investigating a table, search for all of these changes.

A table's current definition is only the final state.

The migration history explains how it reached that state.

---

## 9. Find Index Migrations

Indexes are often added separately from table creation.

For example:

```sql
CREATE INDEX idx_orders_customer_id
ON orders(customer_id);
```

If a query is slow, do not assume the table simply needs an index.

First investigate:

- whether an index already exists
- which migration created it
- whether a later migration removed it
- whether the query uses the indexed column
- whether the index definition changed

Migration history can therefore help explain database performance decisions.

---

## 10. Find Constraint Migrations

Constraints are important because they define database rules.

Examples include:

- primary keys
- unique constraints
- foreign keys
- check constraints
- not-null requirements

Suppose you find:

```sql
UNIQUE (event_id)
```

This may be directly related to idempotent processing.

If an ingestion pipeline tries to insert the same event twice, the database can reject the duplicate.

That means the migration history may explain part of the pipeline's reliability design.

Do not treat constraints as minor database details.

They can be part of the application's correctness model.

---

## 11. Find Foreign Key Changes

Foreign keys connect tables.

For example:

```text
orders.customer_id
        |
        v
customers.id
```

A migration may create that relationship.

Later, another migration may change its behavior.

Generic example:

```sql
ALTER TABLE orders
ADD CONSTRAINT fk_orders_customer
FOREIGN KEY (customer_id)
REFERENCES customers(id);
```

When investigating a failed insert, check whether a foreign key was added or changed.

The application may be sending a value that was accepted before the constraint existed but is rejected now.

---

## 12. Schema Migration vs Data Migration

Not every migration only changes the schema.

Some migrations also change existing data.

Schema migration:

```sql
ALTER TABLE users ADD COLUMN normalized_email TEXT;
```

Data migration:

```sql
UPDATE users
SET normalized_email = LOWER(email);
```

A migration can contain both.

This distinction matters because data migrations can affect large amounts of existing data.

They can also take longer, lock tables, fail partway through, or require careful recovery planning.

When investigating a migration, ask:

- Does it change only metadata/schema?
- Does it modify existing rows?
- Does it create new records?
- Can it be safely rerun?
- Does it require a special deployment sequence?

---

## 13. Find the Migration History Table

Many migration systems maintain a table that records which migrations have been applied.

A generic example is:

```text
migration_history
-----------------
version
applied_at
```

The actual table name depends on the migration framework.

Do not assume the name.

Search the migration tool configuration and database schema.

This history is useful when you need to answer:

- Which migrations have run?
- Which migrations are pending?
- Did production receive a particular schema change?
- Are two environments at the same migration version?

Migration files tell you what *can* be applied.

Migration history tells you what *has been recorded as applied*.

Those are different questions.

---

## 14. Connect Migrations to Application Code

Finding a migration is only half the investigation.

Next, find the code that depends on the schema.

Suppose a migration adds:

```text
account_type
```

Search the repository for:

```text
account_type
```

You may find it in:

- SQL queries
- ORM models
- serializers
- API handlers
- validation logic
- tests
- reports
- ETL transformations

Now you can connect the database change to application behavior.

The investigation becomes:

```text
Migration
   ↓
Database column
   ↓
Application code
   ↓
Pipeline behavior
   ↓
Tests
```

This is much more useful than reading migration files in isolation.

---

## 15. Find Code That Runs Migrations

Another important question is:

> Who actually runs the migrations?

Search for migration commands in:

- Dockerfiles
- Docker Compose files
- shell scripts
- Makefiles
- CI workflows
- deployment scripts
- startup scripts
- release scripts

Generic flow:

```text
Deployment
    ↓
Migration command
    ↓
Database schema update
    ↓
Application startup
```

Sometimes the order is different.

For example:

```text
Migration
    ↓
Application deployment
```

The repository is the source of truth for the actual sequence.

Do not infer the deployment process from the migration directory alone.

---

## 16. Migration Ordering and Application Compatibility

A common production problem happens when a database change and application deployment are not compatible.

Suppose version 1 of the application expects:

```text
orders.id
orders.amount
```

Then a new application version expects:

```text
orders.id
orders.amount
orders.currency
```

If the new application starts before the migration adds `currency`, the application may fail.

A safer deployment may require:

```text
1. Add compatible database structure
2. Deploy application code
3. Start using the new field
4. Remove old structure later
```

This is commonly called an expand-and-contract approach.

Use it when the repository's deployment model requires compatibility between application versions.

---

## 17. Find Renames Carefully

Renames deserve special attention.

Suppose a column changes from:

```text
customer_id
```

to:

```text
client_id
```

A migration may rename the column directly.

But sometimes a project creates a new column and gradually moves to it.

These approaches have different operational effects.

When investigating a rename, search for both names:

```text
customer_id
client_id
```

Then determine:

- which migration introduced each name
- which code still uses the old name
- whether both columns temporarily existed
- whether data was copied
- whether the old column was removed

Do not assume that a new name means the old field disappeared immediately.

---

## 18. Finding Removed Columns

Deleted columns are useful clues during investigations.

Suppose current code does not contain:

```text
legacy_status
```

But an older migration contains it.

That tells you the repository has historical behavior that may still matter when reading old data or old migrations.

Search for:

```text
DROP COLUMN legacy_status
```

Then inspect the migrations before and after the removal.

This can reveal:

- why the field existed
- how it was replaced
- whether data was copied elsewhere
- when application code stopped using it

Old migrations are often valuable documentation.

---

## 19. Migration Rollbacks

Some migration systems support reversing a migration.

For example, a migration may conceptually contain:

```text
upgrade:
    add column

downgrade:
    remove column
```

Rollback support is useful, but do not assume every migration is safely reversible.

A migration that changes existing data may lose information when reversed.

For example:

```text
Old data
   ↓
Data transformation
   ↓
New representation
```

Reversing the schema change may not reconstruct the exact original data.

Always inspect the actual downgrade logic if rollback safety matters.

---

## 20. Migration Investigation Workflow

Here is a practical workflow you can reuse.

### Step 1 — Identify the database

Find the database technology used by the repository.

### Step 2 — Identify the migration tool

Look at dependencies, configuration, scripts, and documentation.

### Step 3 — Find the migration directory

Search rather than assuming a path.

### Step 4 — Find the target table

Search for its creation statement.

### Step 5 — Read later changes

Search for `ALTER TABLE`, indexes, constraints, and related changes.

### Step 6 — Check migration ordering

Understand which migration depends on which earlier migration.

### Step 7 — Find migration history

Determine how the project records applied migrations.

### Step 8 — Find the application code

Search for the table and column names in application code.

### Step 9 — Find tests

Look for tests that depend on the schema.

### Step 10 — Find deployment execution

Determine where and when migrations are actually run.

### Step 11 — Build the timeline

Write down the schema evolution.

Example:

```text
001  create events table
002  add source column
003  add unique event_id
004  add processing_status
005  add index on processing_status
```

Now you have a usable schema history.

---

## 21. Reading a Migration as an Engineer

Do not read a migration only as SQL.

Ask what engineering problem it solves.

For every important migration, ask:

```text
What changed?
Why was it changed?
Which data does it affect?
Which code depends on it?
Can it fail?
Can it be retried?
Can it be rolled back?
Does it require a deployment order?
Does it affect existing rows?
Does it affect performance?
Does it change data quality rules?
```

This turns migration reading into system investigation.

---

## 22. Example: Investigating a New Unique Constraint

Imagine a pipeline starts producing duplicate records.

You find this database constraint:

```sql
UNIQUE (event_id)
```

First, find the migration that created it.

Then inspect the code that writes the table.

You may discover:

```text
Event arrives
    ↓
Pipeline processes event
    ↓
INSERT event_id
    ↓
Duplicate event
    ↓
Database unique constraint rejects duplicate
```

Now the investigation has connected:

```text
Migration
   ↓
Database constraint
   ↓
Insert behavior
   ↓
Duplicate handling
```

This is the kind of connection a Data Engineer needs to make during debugging.

---

## 23. Example: Migration Failure

Suppose a deployment fails while applying a migration.

Do not immediately rerun the command repeatedly.

First determine:

1. Which migration failed?
2. What database operation was running?
3. Did the database transaction roll back?
4. Did any part of the migration remain applied?
5. Does the migration tool record it as applied?
6. Is the migration safe to rerun?
7. Is the application compatible with the current schema?

A useful investigation map is:

```text
Deployment
    ↓
Migration command
    ↓
Migration N
    ↓
Database operation
    ↓
Failure
    ↓
Database state
    ↓
Migration history
    ↓
Recovery decision
```

The correct recovery depends on the migration tool, database behavior, transaction boundaries, and actual repository implementation.

---

## 24. Migration Investigation and Tests

Tests can reveal how the project expects migrations to behave.

Look for tests that:

- create a test database
- apply migrations
- verify table structure
- insert records
- verify constraints
- test indexes or queries
- test migration upgrades
- test migration downgrades

Tests can also reveal assumptions that are not obvious from the migration itself.

For example, a test may show that:

```text
event_id must be unique
status cannot be NULL
customer_id must reference customers
```

That is useful when investigating the schema.

---

## 25. Common Migration Mistakes

### Mistake 1 — Looking only at the current schema

The current schema tells you where the database is now.

It does not explain how it got there.

### Mistake 2 — Assuming the migration directory

Different projects use different structures.

Search first.

### Mistake 3 — Reading migrations without application code

A database change often exists because application behavior changed.

Follow the column into the code.

### Mistake 4 — Ignoring data migrations

An `UPDATE` inside a migration can be more operationally important than an `ALTER TABLE`.

### Mistake 5 — Assuming migrations are always reversible

Data transformations may not be perfectly reversible.

### Mistake 6 — Ignoring deployment order

An application and database must often remain compatible during deployment.

### Mistake 7 — Treating constraints as implementation details

Constraints can enforce important correctness rules.

### Mistake 8 — Re-running a failed migration blindly

First determine the actual database and migration state.

---

## 26. Production Considerations

Migration work becomes more important as the database grows.

Before applying a significant migration, consider:

- table size
- lock behavior
- query impact
- index creation time
- existing traffic
- transaction duration
- data transformation cost
- rollback behavior
- deployment compatibility
- backup and recovery plans
- monitoring during the change

A migration that takes one second on a development database may behave very differently on a production table with millions of rows.

Do not use development timing as proof of production behavior.

Measure and verify in an environment appropriate to the change.

---

## 27. Build a Schema Timeline

One of the most useful outputs of migration investigation is a schema timeline.

Generic example:

```text
Day 1
  Create customers

Day 2
  Create orders

Day 5
  Add orders.customer_id

Day 7
  Add unique constraint to external_order_id

Day 10
  Add processing_status

Day 12
  Add index for processing_status
```

This timeline gives you a much clearer picture than a folder full of migration files.

It also helps when explaining the system to another engineer.

---

## 28. Practical Checklist

Before saying that you understand a repository's migrations, check:

- [ ] I know which database is used.
- [ ] I know which migration system is used.
- [ ] I found the migration files.
- [ ] I know how migrations are ordered.
- [ ] I found the migration that created the table I am investigating.
- [ ] I found later changes to that table.
- [ ] I checked indexes.
- [ ] I checked constraints.
- [ ] I checked foreign keys.
- [ ] I checked data migrations.
- [ ] I found the migration history mechanism.
- [ ] I found application code that depends on the schema.
- [ ] I found relevant tests.
- [ ] I know how migrations are executed during deployment.
- [ ] I understand important compatibility requirements.
- [ ] I understand what happens if a migration fails.

---

## 29. What You Learned

In this recipe, you learned how to investigate database migrations instead of treating the current schema as the whole story.

You learned how to:

- identify the migration system
- find migration files
- trace table creation
- follow schema changes
- investigate indexes and constraints
- distinguish schema changes from data migrations
- find migration history
- connect migrations to application code
- investigate migration execution
- understand deployment compatibility
- investigate migration failures
- use tests as migration documentation
- and build a schema timeline

The important lesson is simple:

> A database schema is the result of a history of changes.

To understand the schema properly, investigate that history.

---

## Recipe Preview

The next chapters move from repository investigation into the implementation recipes.

That means the questions become more concrete:

- Where should data enter the pipeline?
- Which files should change?
- Which database objects are required?
- How should validation work?
- How should duplicates be handled?
- How should failures be recovered?

The repository investigation skills from Part II are now the foundation for those implementation tasks.

Next: **Chapter 13 — Finding Tests**.