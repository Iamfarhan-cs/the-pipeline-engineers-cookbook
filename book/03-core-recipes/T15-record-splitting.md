# T15 — Record Splitting

> **Goal:** Transform one input record into multiple well-defined output records while preserving lineage, deterministic identity, ordering, accounting, and recovery semantics.

Record splitting occurs when one source record contains a collection or multiple logical entities that downstream processing needs as separate records.

Examples:

    order → order lines
    invoice → invoice items
    shipment → packages
    transaction → allocations
    customer → contact methods
    event → individual measurements

The central risk is that one input record can produce many outputs, making duplicates, missing children, unstable IDs, and incorrect accounting easy to introduce.

The central rule is:

> **Every generated child record must have a deterministic relationship to its parent and the split must be fully accountable.**

---

## 1. Problem Recognition

### 1.1 Typical splitting problem

Suppose an order arrives as:

    order_id = 1001
    customer_id = 42
    items = [
        {sku: "A", quantity: 2},
        {sku: "B", quantity: 1},
        {sku: "C", quantity: 4}
    ]

The source contains one order record.

The downstream order-line model requires:

    order_id | line_number | sku | quantity

The transformation is:

    1 parent → 3 child records

### 1.2 Splitting is not filtering

Filtering decides whether a record participates.

Splitting creates multiple logical records from one input.

Example:

    order → order lines

is splitting.

    orders → completed orders only

is filtering.

T13 teaches filtering. T15 teaches **controlled one-to-many expansion**.

### 1.3 Splitting is not enrichment

Enrichment adds attributes while normally preserving grain.

Splitting changes the grain:

    order grain
       ↓
    order-line grain

T14 teaches enrichment. T15 teaches **changing one record into multiple records**.

### 1.4 Red flags

Investigate when:

- output row count grows unexpectedly
- child records cannot be traced to a parent
- repeated runs create duplicate children
- child ordering changes between runs
- empty collections behave inconsistently
- nested records disappear
- parent totals no longer reconcile with children
- generated child identifiers are random or unstable.

---

## 2. Concept and Reasoning

### 2.1 Define the split contract

Before implementation, define:

    input grain
    output grain
    split source
    child identity
    parent identity
    ordering semantics
    empty-collection behavior
    null behavior
    accounting rules
    provenance requirements

Example:

| Property | Contract |
|---|---|
| Input | one order |
| Output | one order line |
| Split field | `items` |
| Parent key | `order_id` |
| Child key | `order_id + line_number` |
| Empty items | quarantine |
| Ordering | source sequence |

### 2.2 Grain must be explicit

An input row may represent:

    order

while the output represents:

    order line

Do not describe the result merely as “more rows.”

The grain tells you what one output row means.

### 2.3 Child identity must be deterministic

Bad:

    child_id = random_uuid()

Every replay produces different identifiers.

Better:

    child_id = order_id + line_number

or another deterministic identifier defined by the business key.

For example:

    order_id = 1001
    line_number = 2
    child_id = "1001:2"

Stable identity is critical for idempotency and reconciliation.

### 2.4 Preserve parent lineage

Every child should normally retain:

    parent_id
    source_record_id
    pipeline_run_id
    child_position

This allows a child to be traced back to its source.

### 2.5 Ordering is not automatically business meaning

A source array may contain:

    [A, B, C]

The positions:

    0, 1, 2

can be preserved if source order is meaningful or required for deterministic identity.

Do not assume array position represents business priority unless the source contract says so.

### 2.6 Empty collection semantics

Suppose:

    items = []

Possible behavior:

    produce zero children
    produce one placeholder child
    quarantine parent
    reject parent

The correct behavior depends on the target model.

Do not accidentally create a fake child row just because an outer join was used.

### 2.7 NULL collection versus empty collection

These may have different meanings:

    items = NULL
    items = []

`NULL` might mean the source did not provide the collection.

`[]` might mean the source explicitly provided zero children.

Preserve this distinction when it matters to the contract.

### 2.8 Parent-level attributes

Parent attributes often need to be copied to each child:

    order_id
    customer_id
    currency

But do not blindly duplicate large payloads or sensitive data into every child.

Copy only the attributes required by the child model.

### 2.9 Monetary and aggregate reconciliation

If the parent contains:

    total_amount = 150

and children contain:

    line_amount = 100
    line_amount = 50

the split should support a reconciliation rule:

    sum(child_amount) = parent_total

when the business contract says the values should reconcile.

### 2.10 Splitting and downstream filtering

Order matters.

Suppose one parent contains ten children and only two are eligible downstream.

Decide whether the business rule requires:

    split → filter children

or:

    filter parent → split

These are not necessarily equivalent.

Define the intended population at each grain.

---

## 3. Implementation

## 3.1 Python list expansion

Example input:

    order = {
        "order_id": 1001,
        "customer_id": 42,
        "items": [
            {"sku": "A", "quantity": 2},
            {"sku": "B", "quantity": 1},
        ],
    }

Deterministic split:

    def split_order(order: dict) -> list[dict]:
        items = order.get("items")

        if items is None:
            raise ValueError("items collection is missing")

        if not items:
            raise ValueError("order contains no items")

        children = []

        for position, item in enumerate(items, start=1):
            child_id = f"{order['order_id']}:{position}"

            children.append({
                "child_id": child_id,
                "parent_order_id": order["order_id"],
                "line_number": position,
                "sku": item["sku"],
                "quantity": item["quantity"],
            })

        return children

## 3.2 Preserve source sequence

If the source contract guarantees meaningful sequence:

    for position, item in enumerate(items, start=1):
        ...

Store the position explicitly:

    line_number = position

Do not derive identity from a mutable array index if the source can reorder elements without preserving logical identity.

## 3.3 Prefer source child identifiers

If the source provides:

    item_id

prefer:

    child_id = item_id

or a composite identity such as:

    source_order_id + source_item_id

over generating an identity from position.

Positions can change when the source reorders the collection.

## 3.4 SQL array splitting

PostgreSQL can expand JSON arrays with `jsonb_array_elements`.

Example:

    SELECT
        o.order_id,
        item->>'sku' AS sku,
        (item->>'quantity')::integer AS quantity
    FROM orders AS o
    CROSS JOIN LATERAL jsonb_array_elements(o.items) AS item;

`CROSS JOIN LATERAL` produces one output row per array element.

## 3.5 Preserve position in PostgreSQL

Use `WITH ORDINALITY` when source sequence is required:

    SELECT
        o.order_id,
        item->>'sku' AS sku,
        (item->>'quantity')::integer AS quantity,
        item_position
    FROM orders AS o
    CROSS JOIN LATERAL
        jsonb_array_elements(o.items) WITH ORDINALITY AS x(item, item_position);

The position can become part of a deterministic child key when the source has no stable child identifier.

## 3.6 Validate the child structure

Before creating children, validate:

    collection exists
    collection type is correct
    required child fields exist
    child values have correct types
    child identifiers are unique

Do not allow malformed children to become valid-looking rows.

## 3.7 Child identity constraint

For a relational target:

    CREATE TABLE order_lines (
        order_id bigint NOT NULL,
        line_number integer NOT NULL,
        sku text NOT NULL,
        quantity integer NOT NULL,
        PRIMARY KEY (order_id, line_number)
    );

The database constraint protects the child grain.

## 3.8 Idempotent loading

With a deterministic child key, replay can safely use an idempotent write pattern:

    INSERT INTO order_lines (
        order_id,
        line_number,
        sku,
        quantity
    )
    VALUES ($1, $2, $3, $4)
    ON CONFLICT (order_id, line_number)
    DO UPDATE SET
        sku = EXCLUDED.sku,
        quantity = EXCLUDED.quantity;

Choose update versus ignore semantics according to the source's correction model.

## 3.9 Accounting

For each parent, calculate:

    child_count

Example:

    parent_items_count = len(order["items"])
    produced_children = len(split_order(order))

Validate:

    parent_items_count == produced_children

unless the contract explicitly allows rejected child elements.

## 3.10 Child-level rejection

Some pipelines allow valid children to continue when one child is invalid.

Then account separately:

    input_children
    accepted_children
    rejected_children

Do not hide child-level loss inside the parent-level output count.

---

## 4. Testing

### 4.1 One parent creates expected children

    def test_order_is_split():
        order = {
            "order_id": 1001,
            "items": [
                {"sku": "A", "quantity": 2},
                {"sku": "B", "quantity": 1},
            ],
        }

        children = split_order(order)

        assert len(children) == 2
        assert children[0]["parent_order_id"] == 1001
        assert children[1]["line_number"] == 2

### 4.2 Deterministic identity

    def test_child_identity_is_stable():
        order = {
            "order_id": 1001,
            "items": [{"sku": "A", "quantity": 2}],
        }

        first = split_order(order)
        second = split_order(order)

        assert first == second

### 4.3 Empty collection

Test the declared behavior for:

    items = []

Do not let the implementation accidentally choose between zero children and a placeholder child.

### 4.4 NULL collection

Test:

    items = None

and verify the documented disposition.

### 4.5 Duplicate child identifiers

Test duplicate source child IDs or duplicate generated positions.

The pipeline should reject or handle the collision explicitly.

### 4.6 Parent-child accounting

Test:

    parent_count
    produced_child_count
    rejected_child_count

and validate the expected relationship.

### 4.7 Aggregate reconciliation

When applicable:

    parent_total == sum(child_total)

Use decimal-safe arithmetic for monetary values.

### 4.8 Ordering

Test that:

    source position 1 → child position 1
    source position 2 → child position 2

when ordering is part of the contract.

### 4.9 Idempotence

Running the split twice against the same source should produce the same child identities and values.

---

## 5. Observability

Monitor:

| Signal | Why it matters |
|---|---|
| parent records | Input population |
| child records | Expansion volume |
| average children/parent | Detects structural changes |
| maximum children/parent | Detects pathological records |
| zero-child parents | Detects missing/empty collections |
| rejected children | Detects child-level quality defects |
| duplicate child IDs | Detects identity failures |
| reconciliation failures | Detects parent/child inconsistency |

Useful structured fields:

    pipeline_run_id
    parent_record_id
    child_record_id
    split_rule_version
    child_position
    disposition

Alert on:

- sudden change in children-per-parent
- unexpected zero-child rate
- duplicate child identifiers
- parent/child reconciliation failures
- unusually large collections.

Do not log full nested payloads merely for observability when they contain sensitive information.

---

## 6. Intentional Failure

### Failure A — Random child IDs

Generate a random UUID for every child.

Expected result:

- replay produces different child identities
- idempotency fails.

### Failure B — Duplicate source child

Provide two children with the same source item identifier.

Expected result:

- uniqueness validation fails
- the pipeline does not silently merge them.

### Failure C — Missing collection

Set the collection to NULL.

Expected result:

- the declared missing-data policy is triggered.

### Failure D — Empty collection mishandled

Use an empty list and accidentally create a placeholder row.

Expected result:

- child-count test fails.

### Failure E — Parent/child amount mismatch

Change one child amount so that the child sum differs from the parent total.

Expected result:

- reconciliation fails
- downstream publication is blocked or flagged according to policy.

### Failure F — Unbounded expansion

Send a parent containing millions of children.

Expected result:

- resource safeguards or bounded processing prevent memory exhaustion.

---

## 7. Recovery

### Scenario 1 — Incorrect child identity

1. Identify the split rule version.
2. Determine affected parents.
3. Preserve the original source records.
4. Correct the identity rule.
5. Rebuild affected child records.
6. Reconcile child counts.
7. Verify downstream references before closing the incident.

### Scenario 2 — Missing children

1. Identify affected parents.
2. Compare source collection counts with produced child counts.
3. Determine whether the loss occurred during parsing, validation, or loading.
4. Correct the failing stage.
5. Replay affected parents.
6. Reconcile child counts and aggregates.

### Scenario 3 — Invalid child causes excessive parent loss

1. Identify whether the business rule permits partial child acceptance.
2. Separate valid and invalid child records.
3. Quarantine invalid children when appropriate.
4. Preserve valid children.
5. Reconcile parent-level and child-level accounting.

### Scenario 4 — Oversized collection

1. Stop or throttle the affected partition.
2. Identify the maximum collection size.
3. Determine whether the source record is valid.
4. Process children in bounded batches.
5. Persist checkpoints if the split is restartable.
6. Reconcile the complete child set.

---

## 8. Production Tools You Should Know

### 8.1 PostgreSQL

Useful for JSON/array expansion, relational child tables, uniqueness constraints, foreign keys, and reconciliation queries.

The underlying mechanism remains deterministic one-to-many expansion with explicit child identity.

### 8.2 Python

Useful for nested object expansion, validation, deterministic child construction, and unit tests.

Use streaming or bounded iteration for large collections instead of materializing unbounded child lists.

### 8.3 dbt

Useful for SQL-based transformations that change grain, documenting the parent/child relationship, and testing uniqueness and relationship constraints.

Grain should be documented explicitly in the model.

---

## 9. Production Runbook

### Before deployment

- [ ] Input grain is documented.
- [ ] Output grain is documented.
- [ ] Parent-child relationship is defined.
- [ ] Child identity is deterministic.
- [ ] Empty and NULL collection behavior is defined.
- [ ] Source child identity is preserved where available.
- [ ] Ordering semantics are defined.
- [ ] Parent/child accounting is implemented.
- [ ] Aggregate reconciliation exists where required.
- [ ] Maximum collection size is defined.

### During execution

- [ ] Monitor parent count.
- [ ] Monitor child count.
- [ ] Monitor children-per-parent distribution.
- [ ] Monitor zero-child parents.
- [ ] Monitor duplicate child IDs.
- [ ] Monitor reconciliation failures.
- [ ] Monitor oversized collections.

### When child counts change unexpectedly

1. Identify the first affected run.
2. Compare source collection sizes.
3. Compare split rule version.
4. Check parser and validation behavior.
5. Check child identity collisions.
6. Check downstream load constraints.
7. Correct the failing stage.
8. Replay affected parents.
9. Reconcile before publication.

---

## 10. Common Mistakes

### Mistake 1 — Random child identifiers

Random IDs make replay and reconciliation harder.

### Mistake 2 — Losing parent lineage

Every child should remain traceable to its parent when lineage is required.

### Mistake 3 — Ignoring grain

Without an explicit output grain, row-count changes are difficult to reason about.

### Mistake 4 — Silently dropping children

Child-level loss must be accounted for.

### Mistake 5 — Treating NULL and empty collections as identical

They can represent different source semantics.

### Mistake 6 — Assuming array position is business identity

Positions can change when the source reorders children.

### Mistake 7 — Hiding cardinality problems with DISTINCT

Duplicate children should be investigated, not masked.

### Mistake 8 — Unbounded expansion

One large parent can exhaust worker memory or create an oversized transaction.

---

## 11. Definition of Done

A production-grade record-splitting implementation is complete when you can:

- [ ] Define input and output grain.
- [ ] Define parent and child identity.
- [ ] Preserve lineage.
- [ ] Define NULL and empty-collection behavior.
- [ ] Define ordering semantics.
- [ ] Validate child structure.
- [ ] Account for every generated child.
- [ ] Handle child-level rejection explicitly.
- [ ] Reconcile parent and child aggregates where required.
- [ ] Implement deterministic splitting.
- [ ] Test identity, cardinality, boundaries, and malformed children.
- [ ] Prove replay produces stable children.
- [ ] Observe expansion and reconciliation metrics.
- [ ] Intentionally break the split.
- [ ] Diagnose missing, duplicate, or excessive children.
- [ ] Recover by replaying from preserved source data.
- [ ] Handle oversized collections safely.
- [ ] Explain the relevant production tools.
- [ ] Operate the mechanism using the runbook.

---

## 12. What You Learned

Record splitting is a grain-changing transformation.

The production reasoning pattern is:

    define parent grain
          ↓
    define child grain
          ↓
    define deterministic child identity
          ↓
    expand one-to-many
          ↓
    validate child structure
          ↓
    account and reconcile
          ↓
    preserve lineage
          ↓
    replay safely

The hardest failures are not simply “too many rows.” They are missing lineage, unstable child identity, silent child loss, and incorrect parent/child relationships. A production-grade split makes the new grain explicit and every generated record accountable.

---

## Next Recipe

**T16 — Record Merging**