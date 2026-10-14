# Star Schema Model: Dental Supply Purchasing

Built from `data_dictionary.md` (MongoDB collections `users`, `orders`, `bills`).

**Business story:** a clinic user fills a cart from several suppliers, checks out (`orders`), each supplier later sends an invoice (`bills`), and the invoice is approved or disputed.

---

## 1. Summary

- 25 tables in total
- 13 dimensions
- 4 bridge tables
- 1 supplier crosswalk table
- 7 fact tables
- Every dimension has one "Unknown" row with key `-1`
- Build order is smallest table first, biggest last
- Every table is built, named and committed in the same order

---

## 2. Business processes and fact grain

Each fact has its own grain. They are never mixed in one table.

| Fact | One row is | Source |
|---|---|---|
| `fact_order` | one checkout | `orders` header |
| `fact_supplier_po` | one supplier purchase order inside a checkout | `orders.meta_data.<SUP>` |
| `fact_order_line` | one product line of a checkout | `orders.products[]` |
| `fact_invoice` | one supplier invoice | `bills` header |
| `fact_invoice_line` | one printed line of an invoice | `bills.line_items[]` |
| `fact_cart_item` | one open cart line, per user, per snapshot day | `users.cart` |
| `fact_price_snapshot` | one cached supplier price, per user, per snapshot day | `users.priceMap` |

Why checkout and supplier PO are separate facts:

- A checkout contains many supplier POs
- The checkout total includes the failed supplier
- Supplier spend must come from the PO grain

---

## 3. Naming conventions

Decided before any table was designed.

### 3.1 Objects

| Object | Rule | Example |
|---|---|---|
| Schema | one schema, `dw` | `dw` |
| Dimension | `dim_<singular noun>` | `dim_supplier` |
| Fact | `fact_<business event>` | `fact_supplier_po` |
| Bridge (many-to-many) | `bridge_<parent>_<child>` | `bridge_user_role` |
| Crosswalk (mapping) | `xref_<what is mapped>` | `xref_supplier_alias` |
| Case | lower `snake_case` | `order_status_key` |

### 3.2 Columns

| Kind | Rule | Example |
|---|---|---|
| Surrogate key | `<dimension>_key`, integer identity | `supplier_key` |
| Source id | source name in snake_case, `*_id` for ObjectIds | `group_id`, `supplier_product_id` |
| Date key | `<role>_date_key`, integer yyyymmdd, points to `dim_date` | `invoice_date_key` |
| Timestamp | `<event>_at`, timestamp with time zone, always UTC | `processed_at` |
| Money | `<what>_amt`, numeric (12,2) | `total_amt` |
| Unit price | `<what>_amt`, numeric (12,4) | `unit_price_amt` |
| Quantity | `*_qty` | `qty`, `incr_qty` |
| Count | `*_cnt` | `supplier_login_cnt` |
| Flag | `is_*`, `has_*`, `can_*`, `requires_*`, boolean | `is_test_user` |
| Raw copy of a source value | `*_raw`, kept next to the cleaned column | `supplier_order_id_raw` |
| Audit | `etl_loaded_at`, `source_updated_at` | |

Two rules that come straight from the dictionary:

- `departmentId` on users becomes `org_department_id`
- The accounting `department` on order lines becomes `gl_department_code`
- They are different things and are never joined without a mapping

### 3.3 Constraints and indexes

| Object | Rule | Example |
|---|---|---|
| Primary key | `pk_<table>` | `pk_dim_user` |
| Unique | `uq_<table>_<purpose>` | `uq_dim_user_id` |
| Foreign key | `fk_<table>_<referenced thing>` | `fk_fact_order_status` |
| Check | `ck_<table>_<purpose>` | `ck_dim_supplier_origin` |
| Index | `ix_<table>_<columns>` | `ix_fact_order_user_key` |

### 3.4 Special values

- Every dimension has an unknown member with key `-1`
- Facts default to `-1` instead of NULL, so joins never drop rows
- One exception: `fact_invoice.supplier_po_key` is NULL when a bill has no real link to an order
- Dimensions loaded from data get a new row the first time a value is seen

---

## 4. Build order at a glance

| Step | Table | Type | Expected size |
|---|---|---|---|
| 0 | schema `dw` | setup | |
| 1 | `dim_order_channel` | dimension | 4 rows |
| 2 | `dim_order_status` | dimension | under 10 rows |
| 3 | `dim_payment_method` | dimension | under 10 rows |
| 4 | `dim_document_type` | dimension | under 10 rows |
| 5 | `dim_invoice_status` | dimension | under 10 rows |
| 6 | `dim_shipping_method` | dimension | tens |
| 7 | `dim_invoice_category` | dimension | tens |
| 8 | `dim_gl_coding` | dimension | tens to hundreds |
| 9 | `dim_date` | dimension | 4,750 rows, generated |
| 10 | `dim_group` | dimension | one per group |
| 11 | `dim_supplier` | dimension | tens |
| 12 | `xref_supplier_alias` | crosswalk | about 2 per supplier |
| 13 | `dim_user` | dimension | one per user |
| 14 | `bridge_user_subgroup` | bridge | about 1 per user |
| 15 | `bridge_user_role` | bridge | about 1 per user |
| 16 | `bridge_user_org_department` | bridge | about 1 per user |
| 17 | `dim_product` | dimension | one per supplier SKU |
| 18 | `fact_order` | fact | one per checkout |
| 19 | `fact_supplier_po` | fact | about 9 per checkout in the sample |
| 20 | `fact_invoice` | fact | one per invoice |
| 21 | `bridge_invoice_approver` | bridge | one per approver per invoice |
| 22 | `fact_order_line` | fact | many per checkout |
| 23 | `fact_invoice_line` | fact | many per invoice |
| 24 | `fact_cart_item` | fact | cart lines times snapshot days |
| 25 | `fact_price_snapshot` | fact | biggest: users times SKUs times days |

- Sizes are estimates from the sample, not measured
- A table only depends on tables with a lower step number

---

## 5. Step-by-step table design


### Step 1: `dim_order_channel`

- **Purpose:** how a supplier PO was sent out
- **Grain:** one row per channel
- **Source:** no source field holds it, derived from `orders.meta_data.<SUP>`
- **Type:** static lookup, no history

| Column | Type | Meaning |
|---|---|---|
| `order_channel_key` | small integer, PK | surrogate key |
| `channel_code` | text(10), unique | `WEB`, `EMAIL`, `DIRECT`, `UNKNOWN` |
| `channel_name` | text(30) | readable name |
| `channel_rule` | text(200) | how the load decides this channel |

Rows:

- `-1` UNKNOWN: channel could not be derived
- `1` WEB: supplier returned an external order id (HS, NOBEL, ULTRADENT)
- `2` EMAIL: PO block has an email delivery event (ONEREP, STERICYCLE)
- `3` DIRECT: everything else (STAPLES, STRSDIMD, PATTERSON, MKSN)

Note: the rule reproduces the 3 + 2 + 4 split in the sample, but it is an assumption.

---

### Step 2: `dim_order_status`

- **Purpose:** overall status of a checkout
- **Grain:** one row per distinct status value
- **Source:** `orders.status`

| Column | Type | Meaning |
|---|---|---|
| `order_status_key` | small integer, PK | surrogate key |
| `status_code` | text(30), unique | status as stored |
| `status_description` | text(200) | readable meaning |

Rows:

- `UNKNOWN` (`-1`)
- `ERRORED`: at least one supplier PO failed, others may have succeeded

Note: the sample only shows ERRORED, other values are added when first seen.

---

### Step 3: `dim_payment_method`

- **Purpose:** how a supplier PO is paid
- **Grain:** one row per payment method
- **Source:** `orders.payment_methods.<SUP>`

| Column | Type | Meaning |
|---|---|---|
| `payment_method_key` | small integer, PK | surrogate key |
| `payment_method_name` | text(50), unique | method as stored |
| `payment_method_note` | text(200) | readable meaning |

Rows:

- `UNKNOWN` (`-1`)
- `Bill On Account`: pay later against the supplier invoice, which is why bills arrive after the order

---

### Step 4: `dim_document_type`

- **Purpose:** kind of document the OCR read
- **Grain:** one row per distinct document type
- **Source:** `bills.document_type`
- **Dropped duplicate:** `bills.inv_type` has the same value in the sample

| Column | Type | Meaning |
|---|---|---|
| `document_type_key` | small integer, PK | surrogate key |
| `document_type_code` | text(30), unique | value as stored |

Note: values are not listed in the dictionary, so only `UNKNOWN` is seeded.

---

### Step 5: `dim_invoice_status`

- **Purpose:** where an invoice is in the approval workflow
- **Grain:** one row per distinct status
- **Source:** `bills.status`

| Column | Type | Meaning |
|---|---|---|
| `invoice_status_key` | small integer, PK | surrogate key |
| `status_code` | text(30), unique | status as stored |

Rows:

- `UNKNOWN` (`-1`)
- `DISPUTED`: the only value visible in the sample

---

### Step 6: `dim_shipping_method`

- **Purpose:** delivery speed or method of a supplier PO
- **Grain:** one row per distinct shipping text
- **Source:** `orders.meta_data.<SUP>.shipping_method`

| Column | Type | Meaning |
|---|---|---|
| `shipping_method_key` | small integer, PK | surrogate key |
| `shipping_method_name` | text(100), unique | text as stored |

Note: supplier-specific free text, expect clean-up after the first real load.

---

### Step 7: `dim_invoice_category`

- **Purpose:** spend category assigned to an invoice
- **Grain:** one row per distinct category
- **Source:** `bills.invoice_category`

| Column | Type | Meaning |
|---|---|---|
| `invoice_category_key` | small integer, PK | surrogate key |
| `invoice_category_name` | text(100), unique | category as stored |

Note: different from the GL expense code on order lines.

---

### Step 8: `dim_gl_coding`

- **Purpose:** accounting (general ledger) coding of an order line
- **Grain:** one row per department + location + expense code combination
- **Source:** `orders.products[].account_code_values`

| Column | Type | Meaning |
|---|---|---|
| `gl_coding_key` | integer, PK | surrogate key |
| `gl_department_code` | text(30) | `account_code_values.department` |
| `gl_location_code` | text(30) | `account_code_values.location_id` |
| `gl_expense_code` | text(30) | `account_code_values.expense_code` |

- The three codes together are unique
- Missing parts load as `UNKNOWN`
- Expense codes seen: 52015, 52005, 52035, 52040, 52050, 62401
- This is the accounting department, not the user's organisational department

---

### Step 9: `dim_date`

- **Purpose:** calendar for every date key in every fact
- **Grain:** one row per day, UTC
- **Range:** 2020-01-01 to 2032-12-31 (4,749 days plus the unknown row)
- **Source:** generated

| Column | Type | Meaning |
|---|---|---|
| `date_key` | integer, PK | yyyymmdd, for example 20250314 |
| `full_date` | date, unique | the calendar date |
| `day_of_week_num` | small integer | ISO, Monday = 1 |
| `day_name` | text(10) | Monday ... Sunday |
| `day_of_month` | small integer | 1 to 31 |
| `day_of_year` | small integer | 1 to 366 |
| `iso_week_num` | small integer | ISO week |
| `month_num` | small integer | 1 to 12 |
| `month_name` | text(10) | January ... December |
| `month_short` | text(3) | Jan ... Dec |
| `quarter_num` | small integer | 1 to 4 |
| `year_num` | small integer | calendar year |
| `year_month` | text(7) | for example 2025-03 |
| `is_weekend` | boolean | Saturday or Sunday |

- Unknown row: key `-1`, date 1900-01-01
- Start year 2020 leaves room for back-filling (oldest value in the sample is a 2023 price fetch)

---

### Step 10: `dim_group`

- **Purpose:** parent organisation (for example a dental chain)
- **Grain:** one row per group
- **Source:** `group_id` on users, orders and bills
- **Type:** SCD 1 (overwrite)

| Column | Type | Meaning |
|---|---|---|
| `group_key` | integer, PK | surrogate key |
| `group_id` | text(24), unique | Mongo ObjectId as text |
| `group_name` | text(200) | empty for now |
| `etl_loaded_at` | timestamp | load time |

Why it is a dimension with only an id: orders and bills both need to be sliced by group. No group master was supplied.

---

### Step 11: `dim_supplier`

- **Purpose:** one row per real supplier, whatever code system refers to it
- **Grain:** one row per supplier
- **Source:** name, address, website and remit-to from `bills.supplier_extra_info`
- **Type:** SCD 1

| Column | Type | Meaning |
|---|---|---|
| `supplier_key` | integer, PK | surrogate key |
| `supplier_code` | text(40), unique | canonical code, the orders-side code when known |
| `supplier_name` | text(200) | legal name |
| `supplier_address` | text | supplier address |
| `website_url` | text(300) | empty in the sample |
| `remit_to_address` | text | where payment is sent |
| `supplier_origin` | text(15) | `CATALOG`, `USER_CREATED` or `UNKNOWN` |
| `etl_loaded_at` | timestamp | load time |

Note: `USER_CREATED` rests on the unconfirmed guess that the `UC` prefix means "user created".

---

### Step 12: `xref_supplier_alias`

- **Purpose:** crosswalk between the two supplier code systems
- **Grain:** one row per namespace + code
- **Source:** maintained by the load or by hand, nothing in the source links them

| Column | Type | Meaning |
|---|---|---|
| `alias_namespace` | text(10), PK part 1 | `ORDERS` or `BILLS` |
| `alias_code` | text(40), PK part 2 | code in that namespace |
| `supplier_key` | integer, FK | points to `dim_supplier` |
| `is_confirmed` | boolean | false until the business confirms the match |
| `etl_loaded_at` | timestamp | load time |

- `ORDERS` namespace: orders, cart and price cache (PATTERSON, HS, MKSN, DC, TDSC)
- `BILLS` namespace: bills (UC100111)
- Facts never use alias codes directly

---

### Step 13: `dim_user`

- **Purpose:** one clinic user account
- **Grain:** one row per user
- **Source:** `users` collection, sections 2.1 to 2.5 of the dictionary
- **Type:** SCD 1 for now
- **Contains PII** (name, email): mask before sharing

| Group | Column | Source / meaning |
|---|---|---|
| Key | `user_key` | surrogate key, PK |
| Key | `user_id` | `_id`, unique |
| Identity | `email`, `username` | username equals email in the sample, both kept |
| Identity | `first_name`, `last_name` | `fname`, `lname` |
| Clinic | `clinic_name`, `street`, `city`, `state`, `zip_code` | zip is text, never a number |
| Clinic | `group_key` | FK to `dim_group` |
| Clinic | `primary_subgroup_id` | `subgroup_id` |
| Account type | `is_clinic_account` | `user_type` contains `clinic` |
| Account type | `is_group_admin` | `user_type` contains `grp_admin` |
| Account type | `is_test_user` | `user_type` contains `test` |
| Feature flags | `has_formulary`, `can_edit_formulary` | `has_formulary`, `edit_formulary` |
| Feature flags | `is_sourcing_enabled`, `is_group_enabled` | `isSourcingEnabled`, `is_group_enabled` |
| Onboarding | `is_order_email_disabled`, `is_approval_email_disabled` | email opt-outs |
| Onboarding | `is_email_verified`, `is_invite_accepted`, `verification_status` | account state |
| Marketing | `referral_type`, `short_ref_url` | acquisition channel, referral link |
| Marketing | `offer_popup_seen_cnt`, `supplier_login_cnt` | engagement counters |
| Lifecycle | `created_at`, `last_login_at` | account timestamps |
| Lifecycle | `supplier_first_linked_at` | `flink`, meaning is a guess |
| Lifecycle | `supplier_last_linked_at` | `supplier_linked_at` |
| Lifecycle | `price_synced_at`, `invoices_first_viewed_at` | `price_synced_at`, `inv_first_viewed_at` |
| Lineage | `source_updated_at`, `etl_loaded_at` | `_meta.at`, load time |

Not loaded (reasons in section 9):

- password
- phone numbers
- sequence number
- UI-state fields
- all background-sync flags
- cart and price cache (they have their own facts)

---

### Step 14: `bridge_user_subgroup`

- **Purpose:** a user can belong to many subgroups
- **Grain:** one row per user + subgroup
- **Source:** `users.subgroup_ids[]`

| Column | Type | Meaning |
|---|---|---|
| `user_key` | big integer, PK part 1, FK | points to `dim_user` |
| `subgroup_id` | text(24), PK part 2 | subgroup ObjectId |
| `is_primary` | boolean | true for `users.subgroup_id` |

Note: no subgroup dimension yet, only ids are known.

---

### Step 15: `bridge_user_role`

- **Purpose:** roles held by a user
- **Grain:** one row per user + role
- **Source:** `users.roleId[]`

| Column | Type | Meaning |
|---|---|---|
| `user_key` | big integer, PK part 1, FK | points to `dim_user` |
| `role_id` | text(24), PK part 2 | role ObjectId |

Note: the role master is not in the sample, so names are unknown.

---

### Step 16: `bridge_user_org_department`

- **Purpose:** organisational departments of a user
- **Grain:** one row per user + org department
- **Source:** `users.departmentId[]`

| Column | Type | Meaning |
|---|---|---|
| `user_key` | big integer, PK part 1, FK | points to `dim_user` |
| `org_department_id` | text(24), PK part 2 | department ObjectId |

Note: not the same as the GL department in `dim_gl_coding`.

---

### Step 17: `dim_product`

- **Purpose:** a product as a supplier sells it
- **Grain:** one row per supplier + supplier SKU
- **Source:** `orders.products[]`
- **Type:** SCD 1

| Column | Type | Meaning |
|---|---|---|
| `product_key` | big integer, PK | surrogate key |
| `supplier_key` | integer, FK | points to `dim_supplier` |
| `supplier_product_id` | text(60) | supplier SKU, for example 5701-091 |
| `product_id` | text(24), nullable | master catalog id |
| `product_name` | text(500) | loaded as UTF-8 |
| `image_url` | text(1000) | product image link |
| `etl_loaded_at` | timestamp | load time |

- Supplier + SKU together are unique
- Why this grain: it is the only key present in orders, bills, cart and price cache
- `product_id` is empty for products seen only on a bill
- `product_id` is not unique, the dictionary does not say if one catalog id can map to several SKUs
- One source name has a broken micron sign, fixed by reading as UTF-8

---

### Step 18: `fact_order`

- **Purpose:** one checkout with its processing lifecycle
- **Grain:** one row per checkout
- **Source:** `orders` header, sections 3.1 and 3.2

| Column | Type | Meaning |
|---|---|---|
| `order_key` | big integer, PK | surrogate key |
| `order_id` | text(24), unique | `orders._id` |
| `user_key` | FK `dim_user` | `user_id`, probably the account owner |
| `ordered_by_user_key` | FK `dim_user` | `order_by`, the person who submitted |
| `group_key` | FK `dim_group` | group on the order |
| `order_status_key` | FK `dim_order_status` | checkout status |
| `created_date_key` | FK `dim_date` | creation day |
| `created_at`, `queued_at`, `started_processing_at`, `processed_at` | timestamps | lifecycle, UTC |
| `processing_duration_sec` | numeric(10,3) | processed minus queued |
| `reported_total_amt` | numeric(12,2) | `total_amt`, includes the failed supplier |
| `savings_total_amt` | numeric(12,2) | formula cannot be checked |
| `shipping_address_text` | text | address lines joined |
| `source_updated_at`, `etl_loaded_at` | timestamps | lineage |

Not stored, rebuilt from `fact_supplier_po`:

- supplier count, success count, error count
- web, email and direct counts
- supplier lists, payment methods, supplier order ids

---

### Step 19: `fact_supplier_po`

- **Purpose:** the purchase order sent to one supplier inside a checkout
- **Grain:** one row per checkout + supplier
- **Source:** `orders.meta_data.<SUP>` plus `payment_methods`, `supplier_order_ids`, success and error supplier lists

| Group | Column | Source / meaning |
|---|---|---|
| Keys | `supplier_po_key` | surrogate key, PK |
| Keys | `order_key` | FK `fact_order` |
| Keys | `po_num` | PO number we generated, for example HS381901 |
| Dimensions | `supplier_key` | FK `dim_supplier`, via the ORDERS alias |
| Dimensions | `user_key`, `group_key`, `order_date_key` | FKs to user, group, date |
| Dimensions | `order_channel_key` | FK `dim_order_channel` |
| Dimensions | `payment_method_key` | FK `dim_payment_method` |
| Dimensions | `shipping_method_key` | FK `dim_shipping_method` |
| Supplier order id | `supplier_order_id_raw` | exactly as stored |
| Supplier order id | `supplier_order_id` | empty when blank or a sentence |
| Status | `is_success` | supplier is in the success list, not the error list |
| Measures | `sub_total_amt` | before tax and shipping |
| Measures | `tax_amt` | tax on the whole PO |
| Measures | `shipping_amt` | `shipping_cost` |
| Measures | `discount_amt` | always null in the sample |
| Measures | `total_amt` | sub total + tax + shipping |
| Measures | `successful_spend_amt` | calculated: `total_amt` if success, else 0 |
| Email receipt | `order_email` | address the PO was sent to |
| Email receipt | `email_delivery_status`, `email_event_type` | delivery result, event type |
| Email receipt | `email_processed_at` | when the provider processed it |

- Checkout + supplier together are unique
- PATTERSON stores a sentence instead of an id, so the raw text is kept and the clean id is empty
- Email provider technical fields (ip, tls, count, message ids, seconds timestamp) stay in the raw layer

---

### Step 20: `fact_invoice`

- **Purpose:** a supplier invoice read from a PDF by OCR and pushed through approval
- **Grain:** one row per invoice
- **Source:** `bills`, sections 4.1, 4.2, 4.5, 4.6

| Group | Column | Source / meaning |
|---|---|---|
| Keys | `invoice_key`, `invoice_doc_id` | surrogate key, `bills._id` (unique) |
| Keys | `invoice_num` | number printed by the supplier |
| Dimensions | `supplier_key` | FK `dim_supplier`, via the BILLS alias |
| Dimensions | `group_key` | FK `dim_group` |
| Dimensions | `document_type_key`, `invoice_status_key`, `invoice_category_key` | FKs to the small lookups |
| Dimensions | `invoice_date_key`, `due_date_key`, `ingested_date_key` | FKs to `dim_date` |
| Weak link | `supplier_po_key` | nullable FK to `fact_supplier_po` |
| As printed | `po_num_raw`, `customer_num`, `account_id` | identifiers on the invoice |
| As printed | `location_name`, `bill_to_address`, `ship_to_address` | free text |
| Measures | `sub_total_amt`, `shipping_amt`, `tax_amt`, `discount_amt` | amounts |
| Measures | `total_amt` | total as printed, never overwritten |
| Measures | `paid_amt` | amount already paid |
| Measures | `recon_diff_amt` | calculated: total minus (sub total + shipping + tax - discount) |
| Flags | `is_due_date_defaulted` | due date was not on the document |
| Flags | `requires_all_approvers` | `approval_condition_is_and`, meaning assumed |
| Flags | `is_workflow_active` | `invoice_workflow_active` |
| Ingestion | `inbox_id`, `inv_source`, `original_filename` | how it arrived |
| Ingestion | `page_from`, `page_to` | pages in the PDF |
| Lineage | `source_created_at`, `source_updated_at`, `source_tlm_at` | tie-breaker for the latest version |

- The sample invoice is off by 4.00 (38.66 + 7.50 = 46.16, printed 50.16), visible in `recon_diff_amt`
- The link to a PO stays empty unless `po_num` really matches one

---

### Step 21: `bridge_invoice_approver`

- **Purpose:** people who must approve an invoice
- **Grain:** one row per invoice + approver
- **Source:** `bills.approval_info[]`

| Column | Type | Meaning |
|---|---|---|
| `invoice_key` | big integer, PK part 1, FK | points to `fact_invoice` |
| `approver_seq` | small integer, PK part 2 | position in the source array |
| `approver_user_key` | FK `dim_user` | `-1` when the id is not found |
| `approver_email`, `approver_name` | text | as printed |

Note: `requires_all_approvers` on the invoice says whether all or any one must approve.

---

### Step 22: `fact_order_line`

- **Purpose:** one product line of a checkout, with price comparison and GL coding
- **Grain:** one row per checkout + line position
- **Source:** `orders.products[]`, section 3.3

| Group | Column | Source / meaning |
|---|---|---|
| Keys | `order_line_key` | surrogate key, PK |
| Keys | `order_key`, `line_num` | checkout and position, together unique |
| Keys | `supplier_po_key` | PO of the line's supplier in this checkout |
| Dimensions | `product_key`, `supplier_key` | FKs to product and supplier |
| Dimensions | `user_key`, `group_key`, `order_date_key` | FKs to user, group, date |
| Dimensions | `gl_coding_key` | FK `dim_gl_coding` |
| Measures | `qty` | packs ordered |
| Measures | `unit_price_amt` | price of one pack |
| Measures | `line_amt` | qty times unit price, rounded to 2 decimals |
| Flag | `is_backorder` | missing in the source means false |
| Comparison | `lowest_price_product_key` | the supplier + SKU key found inside `lowest_price` |
| Comparison | `lowest_price_amt` | the lowest price found |
| Comparison | `pack_qty_normalised` | `lowest_price.qty`, a count or fluid ounces depending on the item |

- `dim_product` is used twice: the product bought and the cheapest alternative
- The dynamic key inside `lowest_price` is unpivoted into the two comparison columns
- Group here is the group on the order, which may differ from the user's current group

---

### Step 23: `fact_invoice_line`

- **Purpose:** one printed line of a supplier invoice
- **Grain:** one row per invoice + line position
- **Source:** `bills.line_items[]`, section 4.3

| Column | Type | Meaning |
|---|---|---|
| `invoice_line_key` | big integer, PK | surrogate key |
| `invoice_key`, `line_num` | FK, small integer | invoice and position, together unique |
| `product_key` | FK `dim_product` | `-1` when no match |
| `supplier_key`, `group_key`, `invoice_date_key` | FKs | supplier, group, invoice day |
| `supplier_product_id_raw` | text(100) | SKU as printed |
| `item_name_raw` | text(500) | description as printed |
| `qty` | integer | quantity billed |
| `unit_price_amt` | numeric(12,4) | price per unit |
| `line_amt` | numeric(12,2) | line total |

- Bill lines have no `product_id`
- Matching is on supplier + SKU, exact match only
- The sample SKU has a size appended ("7265 12 M"), so some lines will not match and fall back to `-1`

---

### Step 24: `fact_cart_item`

- **Purpose:** what sits in a user's cart when the load runs
- **Grain:** one row per snapshot day + user + product
- **Type:** periodic snapshot, because the cart is current state
- **Source:** `users.cart.items` and `users.cart.additional_info`

| Column | Type | Meaning |
|---|---|---|
| `cart_item_key` | big integer, PK | surrogate key |
| `snapshot_date_key` | FK `dim_date` | day of the snapshot |
| `user_key` | FK `dim_user` | cart owner |
| `product_key` | FK `dim_product` | from the supplier + SKU key |
| `qty` | integer | cart line quantity |
| `incr_qty` | integer | quantity step, assumed |
| `max_price_amt` | numeric(12,4) | highest price found |
| `lowest_price_product_key` | FK `dim_product` | `-1` when nothing was chosen |
| `lowest_price_amt` | numeric(12,4) | chosen lowest price |
| `cart_updated_at` | timestamp | last cart change |

Left out on purpose:

- keys in `additional_info` that are not in `cart.items` (leftovers of removed items)
- the per-supplier price list inside `additional_info` (covered by `fact_price_snapshot`)
- `cart.noteId` (note text is unknown)

---

### Step 25: `fact_price_snapshot`

- **Purpose:** cached supplier price and stock status per user
- **Grain:** one row per snapshot day + user + product + price break
- **Type:** periodic snapshot
- **Source:** `users.priceMap.<SUP>.<SKU>`
- **Why last:** expected to be the biggest table in the model

| Column | Type | Meaning |
|---|---|---|
| `price_snapshot_key` | big integer, PK | surrogate key |
| `snapshot_date_key` | FK `dim_date` | day of the snapshot |
| `user_key` | FK `dim_user` | cache owner |
| `supplier_key` | FK `dim_supplier` | supplier |
| `product_key` | FK `dim_product` | supplier SKU |
| `fetched_date_key` | FK `dim_date` | day the price was fetched |
| `price_break_qty` | small integer | the `"1"` key, only tier 1 appears |
| `unit_price_amt` | numeric(12,4) | price, cast because some source prices are strings |
| `fetched_at` | timestamp | exact fetch time |
| `is_available_on_call` | boolean | phone-only item |
| `availability_status` | text(50) | stock status |

Note: fetch ages differ a lot (2023 versus 2025), so always check `fetched_date_key` before trusting a price.

---

## 6. How the facts connect to dimensions

| Fact | Dimensions used |
|---|---|
| `fact_order` | user (twice: owner, submitter), group, order status, date |
| `fact_supplier_po` | supplier, user, group, date, order channel, payment method, shipping method, parent `fact_order` |
| `fact_order_line` | product (twice: bought, cheapest), supplier, user, group, date, GL coding, parent `fact_order`, parent `fact_supplier_po` |
| `fact_invoice` | supplier, group, document type, invoice status, invoice category, date (three roles), optional `fact_supplier_po` |
| `fact_invoice_line` | product, supplier, group, date, parent `fact_invoice` |
| `fact_cart_item` | user, product (twice: item, cheapest), date |
| `fact_price_snapshot` | user, supplier, product, date (two roles) |

Bridges:

- `dim_user` to subgroup, role, org department
- `fact_invoice` to approver (`dim_user`)

Crosswalk:

- `dim_supplier` to both supplier code systems

---

