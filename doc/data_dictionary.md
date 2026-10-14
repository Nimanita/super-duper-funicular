# Data Dictionary: users, orders, bills (MongoDB sample documents)

This file explains **every field** in the three sample documents, including the ones where I had to guess. Read it before looking at any schema.

## Conventions

- `<SUP>` means a supplier code used as a JSON key, for example `PATTERSON`, `HS`, `MKSN`.
- `<SKU>` means a supplier product id, for example `5701-091`.
- All ids named `*_id`, `roleId`, `departmentId` etc. are 24-character MongoDB ObjectId strings unless stated otherwise.
- Where a meaning is a guess, the text says so ("probably", "looks like", "assumed"). Section 8 lists the guesses to confirm.

---

## 1. What each collection is

| Collection | One document = | Grain | Source of the document |
|---|---|---|---|
| `users` | One user account of a dental clinic, with the user's current cart and a cache of supplier prices embedded inside it. | one user | Application profile service. |
| `orders` | One checkout. The checkout is split into one purchase order (PO) per supplier, and every supplier PO has its own amounts. | one checkout, containing many supplier POs, containing many product lines | Application order service. |
| `bills` | One supplier invoice, read from an emailed PDF by OCR and then pushed through an approval workflow. | one invoice | Invoice ingestion service. |

**Business story in one line:** a clinic user fills a cart from several suppliers, checks out (`orders`), each supplier later sends an invoice (`bills`), and the invoice is approved or disputed.

---

## 2. `users` collection

### 2.1 Identity and contact

| Field | Type | Meaning |
|---|---|---|
| `_id.$oid` | string | MongoDB primary key of the user document. |
| `seq` | int | Sequence/serial number of the record. Looks like a per-document counter. |
| `email` | string | Contact email. |
| `username` | string | Login name. Equal to `email` in the sample. |
| `fname` | string | First name. |
| `lname` | string | Last name. |
| `phone_no[]` | array of string | Phone numbers. Probably the older field. |
| `phone_nos[]` | array of string | Phone numbers. Probably the newer field; it holds a different value from `phone_no` in the sample, so both are needed. |
| `password` | string | Credential. Redacted in the sample. |

### 2.2 Clinic, location and organisation

| Field | Type | Meaning |
|---|---|---|
| `clinic_name` | string | Name of the clinic the user works for. |
| `street` | string | Street address of the clinic. |
| `city` | string | City. |
| `state` | string | State. In the sample it is `Alabama` while the city is Hyderabad, so the sample address is fake test data. |
| `zip_code` | string | Postal code. `500089` is an Indian PIN, again inconsistent with the state. Store as text, never as a number. |
| `group_id` | string | Id of the group (parent organisation, for example a dental chain) the clinic belongs to. |
| `subgroup_id` | string | One subgroup (a division or location of the group). Equals the only element of `subgroup_ids` in the sample, so it is probably the primary subgroup. |
| `subgroup_ids[]` | array of string | All subgroups the user belongs to. Array means many-to-many. |
| `roleId[]` | array of string | Ids of the user's roles. The role master collection is not in the sample, so the names are unknown. |
| `departmentId[]` | array of string | Ids of the user's departments (organisational departments). **Not the same thing** as the accounting `department` code in orders. |
| `user_type[]` | array of string | Tags describing the account. `clinic` = clinic account, `grp_admin` = administrator of the group, `test` = test account. (assumed) |

### 2.3 Feature flags and onboarding state

| Field | Type | Meaning |
|---|---|---|
| `has_formulary` | bool | The account has a formulary, meaning a saved list of the clinic's approved or usual products. (assumed) |
| `edit_formulary` | bool | The user may edit the formulary. (assumed) |
| `isSourcingEnabled` | bool | The price-comparison ("sourcing") feature is on for this account. (assumed) |
| `is_group_enabled` | bool | The multi-location group feature is on. (assumed) |
| `order_email_disabled` | bool | Do not send order emails to this user. |
| `approval_email_disabled` | bool | Do not send approval emails to this user. |
| `email_verified` | bool | The user confirmed their email address. |
| `invite_accepted` | bool | The user accepted the invitation to join. |
| `verification_status` | string | Account verification state. |
| `addToList` | bool | UI setting about adding items to a list. Exact meaning unknown. |
| `isVisited` | int | Counter or state about visits to a screen (onboarding). Exact meaning unknown. |

### 2.4 Marketing and engagement

| Field | Type | Meaning |
|---|---|---|
| `referral_type` | string | How the user found the product (acquisition channel). |
| `shortRefUrl` | string | Short referral link of this user. |
| `offer_popup_seen_count` | int | How many times the offer pop-up was shown. |
| `supplier_login_count` | int | Number of logins to supplier accounts made through the app. (assumed) |

### 2.5 Timestamps in `users`

Formats are mixed: `{"$date": <epoch ms>}`, plain epoch ms numbers, and (in other collections) ISO strings.

| Field | Format | Meaning |
|---|---|---|
| `created_at` | `$date` ms | Account creation time. |
| `last_login_at` | `$date` ms | Last login. |
| `flink` | `$date` ms | Probably "first link", the first time a supplier account was linked. Unclear. |
| `supplier_linked_at` | `$date` ms | Last time a supplier account was linked. (assumed) |
| `price_synced_at` | `$date` ms | Last time the user's supplier prices were refreshed. (assumed) |
| `master_updated_at` | `$date` ms | Last update of the user's product master data (formulary). (assumed) |
| `purchase_list_updated_at` | `$date` ms | Last update of the purchase list. Equals `master_updated_at` in the sample. (assumed) |
| `inv_first_viewed_at` | `$date` ms | First time the user opened the invoices area. (assumed) |
| `pass_reset_req_at` | plain epoch ms | Last password-reset request. |
| `cart_updated_millis` | plain epoch ms | Last change to the cart. |

### 2.6 `cart` (the current shopping cart)

| Field | Type | Meaning |
|---|---|---|
| `cart.items` | map | Cart lines. Key is `"<SUP>:<SKU>"`, value is the quantity ordered. |
| `cart.noteId` | string (UUID) | Id of a note attached to the cart. Note content is not in the sample. (assumed) |
| `cart.additional_info` | map | Price-comparison data per cart item, keyed by `"<SUP>:<SKU>"`. See below. It also contains keys that are **not** in `cart.items` (`DARBY:9515480`, `BENCO:5680-739`), which are probably removed items left behind. |
| `additional_info.<key>.price` | map | Price of the **same item** at several suppliers. |
| `price.DARBY`, `price.BENCO`, `price.DENTALCITY`, `price.TDSC`, `price.DC` | number or string | Price at that supplier. `TDSC` and `DC` are supplier codes whose full names are not in the data. `DC` is often lower than the others and is not equal to `DENTALCITY`, so it is a separate source. (assumed) |
| `price.max` | number | Highest price found for the item across suppliers. It can be higher than every listed supplier (for `DENTALCITY:76-201`, `max` is 143.85 while the listed prices top out at 44.49), so it probably covers suppliers not shown. |
| value type quirk | string vs number | The price of the supplier that owns the cart line is a **string** (`"DARBY": "54.22"` on `DARBY:5251516`); all others are numbers. |
| `additional_info.<key>.lowest_price` | map | The cheapest price chosen for the item, as `{"<SUP>:<SKU>": price}`. Empty `{}` when nothing was chosen. For `DARBY:9541206` the chosen price is 11.99 although `DC` shows 11.15, so `lowest_price` only considers suppliers the user has linked and `DC` is not one of them. (assumed) |
| `additional_info.<key>.incr_qty` | int | Quantity step or increment for the item. (assumed) |

### 2.7 `priceMap` (per-user price and stock cache)

Structure: `priceMap.<SUP>.<SKU> = { "1": price, fetched_at, available_on_call, availability_status }`. One key per supplier code (17 supplier keys exist; most are empty `{}` in the sample).

| Field | Type | Meaning |
|---|---|---|
| `priceMap.<SUP>` | map | Cached prices for one supplier. |
| `priceMap.<SUP>.<SKU>` | map | One supplier item. |
| `"1"` (inside the SKU map) | number | Price at quantity tier 1. The key looks like a quantity break; only tier `"1"` appears. |
| `fetched_at` | epoch ms | When this price was last fetched from the supplier. Different items have very different ages (2023 versus 2025). |
| `available_on_call` | bool | Item is only available by phone. |
| `availability_status` | string | Stock status. |

Supplier keys present in `priceMap`: DC, HS, TDSC, BENCO, DARBY, SAFCO, PEARSON, WBMASON, AODIRECT, DENTSPLY, BRASSELER, PATTERSON, DENTALCITY, BIOHORIZONS.

### 2.8 `_meta` (change-tracking envelope)

The same envelope appears in all three collections.

| Field | Type | Meaning |
|---|---|---|
| `_meta.at` | epoch ms or ISO | Time of the change event. Use it to pick the latest version of a document. |
| `_meta.by` | string or null | User who made the change. `null` = system. |
| `_meta.src` | string | Service that wrote the change. |
| `_meta.traceId` | string | Request trace id for debugging. |

### 2.9 Internal sync flags (all bool)

These show whether background jobs have computed statistics for the user. They carry no business meaning, and two of them are spelled both ways (`synced` and `synched`).

Fields: `is_stats_synched`, `is_item_stats_synced`, `is_item_stats_synched`, `is_supplier_stats_synced`, `is_supplier_stats_synched`, `order_frequency_data_synced`, `is_category_order_stats_synced`, `is_supplier_order_stats_synced`, `is_expense_code_order_stats_synced`, `is_supp_product_order_stats_synced`, `useOrderStatsPipeline`.

All are bool flags. Assumed to be background-job bookkeeping with no business meaning.

---

## 3. `orders` collection

### 3.1 Header and lifecycle

| Field | Type | Meaning |
|---|---|---|
| `_id.$oid` | string | Primary key of the checkout. |
| `status` | string | Overall status of the checkout. `ERRORED` here because one supplier failed (`MKSN`) while the other 8 succeeded, so it means "at least one supplier failed", not "everything failed". (assumed) |
| `user_id` | string | The user (account) the order belongs to. |
| `order_by` | string | The person who actually placed the order. Differs from `user_id` in the sample, so `user_id` is probably the account owner and `order_by` the submitter. |
| `group_id` | string | Group the order belongs to. Differs from the group in the users sample (sample data not linked). |
| `shipping_address[]` | array of string | Delivery address, one string per address line. |
| `created_at` | `$date` ms | Order created. |
| `queued_at` | `$date` ms | Order placed in the processing queue. |
| `started_processing_at` | `$date` ms | Worker started sending the order to suppliers. |
| `processed_at` | `$date` ms | All suppliers handled. |
| `processing_duration` | number | Seconds from queue to completion. `processed_at - queued_at` = 18.707 s, exactly the stored value. |

### 3.2 Rollups on the order (all derivable from the children)

| Field | Type | Meaning |
|---|---|---|
| `total_amt` | number | Sum of every supplier PO `total`, **including the failed supplier**. The nine PO totals add up to exactly 3643.09. |
| `savings_total` | number | Money saved by choosing cheaper suppliers than the comparison price. The formula cannot be checked from the sample. |
| `suppliers[]` | array | List of supplier codes in the order. |
| `suppliers_count` | int | Number of suppliers. Equals `success_count + error_count` (8 + 1). |
| `success_count` / `error_count` | int | Suppliers that succeeded or failed. |
| `success_suppliers[]` / `error_suppliers[]` | array | The supplier codes behind those counts. Error list is `["MKSN"]`. |
| `web_count` | int | Number of suppliers ordered through the **web** channel. Matches the three suppliers that return an external order id (`HS`, `NOBEL`, `ULTRADENT`). (assumed) |
| `email_count` | int | Number of suppliers ordered by **email**. Matches the two suppliers with an email delivery event (`ONEREP`, `STERICYCLE`). (assumed) |
| `direct_count` | int | Number of suppliers ordered through a **direct integration** (the other four: `STAPLES`, `STRSDIMD`, `PATTERSON`, `MKSN`). 3 + 2 + 4 = 9 suppliers. (assumed) |
| `payment_methods` | map | Payment method per supplier. All are `Bill On Account`, meaning pay later against an invoice, which is why bills arrive afterwards. |
| `supplier_order_ids` | map | The supplier's own order number per supplier. Three patterns: a real external id (`HS`, `NOBEL`, `ULTRADENT`); our own `po_num` repeated (`ONEREP`, `STAPLES`, `STRSDIMD`, `STERICYCLE`); empty string for the failed supplier (`MKSN`). **`PATTERSON` holds a sentence** ("Your order has been submitted and an email will be sent...") instead of an id. |

### 3.3 `products[]` (order lines)

| Field | Type | Meaning |
|---|---|---|
| `product_id` | string | Id of the product in the app's master catalog. |
| `supplier` | string | Supplier code that fulfils this line. |
| `supplier_product_id` | string | The supplier's SKU or catalog number. |
| `name` | string | Product name as shown in the catalog. One name contains a broken character (`ÔøΩ`, an encoding error for the micron sign), so read the data as UTF-8. |
| `image_url` | string | Product image link. |
| `qty` | int | Number of packs ordered. |
| `unit_price` | number | Price of one pack. |
| `price` | number | Line total. `price = qty x unit_price` (for example 3 x 61.58 = 184.74). The raw values carry float noise (`184.74000000000000909`), so round to 2 decimals. |
| `is_backorder` | bool | Item is on backorder. Present only on one line (blades), so a missing field means `false`. |
| `lowest_price` | map | Price comparison for the product. (assumed) |
| `lowest_price.qty` | number or string | Normalised pack quantity used to compute a per-unit price. It is a count for countable items (250 gloves per box, 160 wipes per tub) and a **fluid-ounce volume** for liquids (500 ml = 16.91 fl oz; 18 x 500 ml = 304.33 fl oz). It is sometimes a string and sometimes a number. (assumed) |
| `lowest_price."<SUP>:<SKU>"` | number | Lowest price found for that product (equal to `unit_price` in the sample). The key name is dynamic, so it must be unpivoted. (assumed) |
| `account_code_values` | map | Accounting (general ledger) coding of the line. |
| `account_code_values.department` | string | Accounting department code. **Not** the same as the user's `departmentId`. |
| `account_code_values.location_id` | string | Accounting location code. |
| `account_code_values.expense_code` | string | Expense category (GL account). Values seen: 52015, 52005, 52035, 52040, 52050, 62401. |

### 3.4 `meta_data.<SUP>` (one purchase order per supplier)

The key under `meta_data` is the supplier code. Each block is a PO.

| Field | Type | Meaning |
|---|---|---|
| `po_num` | string | Purchase order number we generated for that supplier. The prefix looks like the supplier (`HS381901`, `PTU381902`). |
| `sub_total` | number | Sum of the PO's line amounts, before tax and shipping. |
| `tax` | number | Tax on the whole PO (not per line). |
| `shipping_cost` | number | Shipping charge. |
| `discount` | number or null | Discount. Always `null` in the sample. |
| `total` | number | `sub_total + tax + shipping_cost` (for `STRSDIMD`: 21.25 + 1.70 + 9.00 = 31.95). |
| `shipping_method` | string | Delivery speed or method. |
| `order_email` | string | Email address the PO was sent to. Present only for email-channel suppliers. |
| `email_delivery_status` | string | Result of the email send. |
| `email_delivery_event` | map | The delivery receipt from the email provider. The `sg_` prefixes point to **SendGrid**. (assumed) |
| `email_delivery_event.ip` | string | IP address in the event. (assumed) |
| `email_delivery_event.tls` | int | Message was sent over TLS (1 = yes). (assumed) |
| `email_delivery_event.count` | int | Event count or attempt number. (assumed) |
| `email_delivery_event.email` | string | Recipient address. |
| `email_delivery_event.event` / `event_name` | string | Event type (`delivered`). The two fields repeat each other. |
| `email_delivery_event.smtp-id` | string | SMTP message id. Redacted. |
| `email_delivery_event.response` | string | Remote server's reply. Redacted. |
| `email_delivery_event.processed` | `$date` ms | When the provider processed the email. |
| `email_delivery_event.timestamp` | epoch **seconds** | Same moment as `processed`, but in seconds, not milliseconds. |
| `email_delivery_event.sg_event_id` / `sg_message_id` | string | SendGrid ids for the event and the message. (assumed) |

### 3.5 `_meta` in `orders`

Same four fields as in section 2.8. In the orders sample `_meta.by` holds a user email (`abc@example.com`), so it records who triggered the change.

---

## 4. `bills` collection

### 4.1 Invoice header

| Field | Type | Meaning |
|---|---|---|
| `_id.$oid` | string | Primary key of the invoice document. |
| `inv_num` | string | Invoice number printed by the supplier. |
| `inv_type` | string | Type of the invoice record. Same value as `document_type` in the sample. Other values (for example credit memo) probably exist. |
| `document_type` | string | Type of the scanned document. Duplicate of `inv_type`. (assumed) |
| `inv_date` | `$date` ISO | Invoice date. |
| `due_date` | `$date` ISO | Payment due date. Here it is exactly `inv_date + 30 days` and `is_due_date_defaulted` is `true`, so the due date was **not on the invoice**; the system filled in a default. |
| `po_num` | string | Purchase-order reference found on the invoice. In the sample it equals `invoice_extra_info.account_id` (`UN638655`), so it is a customer account number, **not** an order PO from the `orders` collection. |
| `customer_num` | string | Our customer number at the supplier. |
| `status` | string | Invoice workflow status. Values like `DISPUTED` suggest others (approved, paid, pending). (assumed) |
| `invoice_category` | string | Spend category assigned to the invoice. |
| `group_id` | string | Group the invoice belongs to. |
| `location_name` | string | Location name, empty in the sample. |
| `supplier` | string | Supplier code. It is a different namespace from the codes in orders (`PATTERSON`), so it needs a mapping table. The `UC` prefix may mean "user created" (a supplier not in the catalog). |
| `bill_to` | string | Billing address as one free-text string. |
| `ship_to` | string | Shipping address as one free-text string. |

### 4.2 Amounts

| Field | Type | Meaning |
|---|---|---|
| `sub_total` | number | Sum of the line items (18.70 + 19.96 = 38.66). |
| `shipping` | number | Shipping charge. |
| `taxes` | number | Tax. |
| `discount_amount` | number | Discount. |
| `total` | number | Amount billed as printed. **Does not reconcile**: 38.66 + 7.50 + 0 = 46.16, but `total` is 50.16, so 4.00 is unexplained (probably an extra fee the OCR did not capture). Keep the printed total and store the difference as `recon_diff`. |
| `paid_amount` | number | Amount already paid. |

### 4.3 `line_items[]`

| Field | Type | Meaning |
|---|---|---|
| `supplier_product_id` | string | Supplier SKU, with the size appended in the sample (`7265 12 M`). (assumed) |
| `name` | string | Item description as printed. |
| `qty` | int | Quantity billed. |
| `unit_price` | number | Price per unit. |
| `price` | number | Line total. `qty x unit_price` (2 x 9.35 = 18.70). |

Note: invoice lines have **no `product_id`**, so they can only be linked to the catalog through `supplier` + `supplier_product_id`.

### 4.4 Supplier details (`supplier_extra_info`)

| Field | Type | Meaning |
|---|---|---|
| `name` | string | Supplier legal name. |
| `address` | string | Supplier address. |
| `website` | string | Supplier website, empty here. |
| `remit_to` | string | Address where payment must be sent. |

### 4.5 Approval workflow

| Field | Type | Meaning |
|---|---|---|
| `approval_info[]` | array of map | People who must approve the invoice. Each entry has `email`, `user_id` and `user_name`. |
| `approval_info[].user_id` | string | The approver's user id. |
| `approval_info[].email` / `user_name` | string | Approver contact details. |
| `approval_condition_is_and` | bool | `true` = **all** approvers must approve; `false` = **any one** approver is enough. (assumed) |
| `invoice_workflow_active` | bool | The approval workflow is switched on for this invoice. |

### 4.6 Ingestion (how the invoice entered the system)

| Field | Type | Meaning |
|---|---|---|
| `inbox_id` | string | Id of the receiving inbox (the mailbox the PDF arrived in). (assumed) |
| `inv_source` | string | Channel and sender: emailed, from the given address. |
| `file_name` | string | Name of the stored PDF. Redacted. |
| `original_filename` | string | Name of the file as uploaded or received. |
| `invoice_extra_info.page_from` / `page_to` | int | Pages of the PDF that hold this invoice, so one PDF may contain several invoices. (assumed) |
| `invoice_extra_info.account_id` | string | Customer account number read from the invoice. |
| `invoice_extra_info.is_due_date_defaulted` | bool | The due date was not found on the document and a default was applied. |
| `created_at` | `$date` ISO | Document created in the database. |
| `updated_at` | `$date` ISO | Last update. |
| `tlm` | `$date` ISO | "Time last modified". It is 138 ms after `updated_at`, so it is a separate system stamp. Useful as a tie-breaker when choosing the latest version. (assumed) |
| `_meta` | map | Same envelope as in section 2.8. |

---

## 5. Checks I ran on the sample (so you can trust these notes)

| Check | Result |
|---|---|
| Sum of the nine `meta_data.*.total` equals `total_amt` | 3643.09 = 3643.09, matches |
| `suppliers_count` = `success_count` + `error_count` | 9 = 8 + 1, matches |
| `web_count` + `email_count` + `direct_count` = `suppliers_count` | 3 + 2 + 4 = 9, matches |
| `email_count` equals number of suppliers with an email delivery event | 2 (`ONEREP`, `STERICYCLE`), matches |
| `web_count` equals number of suppliers with an external order id | 3 (`HS`, `NOBEL`, `ULTRADENT`), matches |
| `processing_duration` = `processed_at` - `queued_at` | 18.707 s, matches |
| Order line `price` = `qty` x `unit_price` | matches on the lines I checked |
| PO `total` = `sub_total` + `tax` + `shipping_cost` | matches (for example `STRSDIMD` 31.95) |
| Invoice `sub_total` = sum of `line_items.price` | 38.66, matches |
| Invoice `total` = `sub_total` + `shipping` + `taxes` - `discount_amount` | **Does not match**: 46.16 versus 50.16 |
| `due_date` = `inv_date` + 30 days and `is_due_date_defaulted` = `true` | matches |
| Liquid `lowest_price.qty` equals volume in fluid ounces | 500 ml = 16.91, 18 x 500 ml = 304.33, matches |

---

## 6. Data-quality notes

| Problem | Where | What to do |
|---|---|---|
| Money stored as strings and with float noise | `cart.additional_info`, `priceMap`, order lines | Cast to `NUMERIC(12,2)`; use `(12,4)` for unit prices. |
| `lowest_price.qty` is sometimes a string, sometimes a number | `orders.products[]` | Cast to `NUMERIC`. |
| Three date formats | all collections | Convert to UTC `timestamptz`. Remember `timestamp` in the email event is in seconds. |
| Dirty supplier order id | `supplier_order_ids.PATTERSON` | Keep the raw text; store a null in the clean id column. |
| Failed supplier counted in the total | `total_amt`, `meta_data.MKSN` | Compute successful spend from successful supplier POs only. |
| Encoding error `ÔøΩ` in a product name | `orders.products[].name` | Force UTF-8 on ingestion. |
| Two supplier code systems | orders use `PATTERSON`, bills use `UC100111` | Build a supplier alias (crosswalk) table. |
| Three meanings of "department" | users `departmentId` (org department id), orders `department` (GL code) | Never join them without a mapping. |
| Duplicate or similar fields | `phone_no` and `phone_nos`; `inv_type` and `document_type`; `event` and `event_name`; `synced` and `synched` | Pick one, or keep both with a documented rule. |
| Test data | `user_type` contains `test`; city, state and zip of the sample user disagree | Filter test users out of analytics. |
| Sample ids do not link across files | `group_id` differs in all three samples | Real data should link; do not infer relationships from the sample ids. |

---

## 7. How the three collections connect

| From | To | Through | Reliability |
|---|---|---|---|
| `orders.user_id` / `orders.order_by` | `users._id` | ObjectId | Reliable in real data |
| `orders.group_id`, `users.group_id`, `bills.group_id` | one group | ObjectId | Reliable in real data |
| `orders.products[].product_id` | product catalog | ObjectId | Reliable (catalog collection not supplied) |
| `bills.line_items[]` | order lines | `supplier` + `supplier_product_id` | Indirect, only after the supplier alias mapping |
| `bills` | `orders` | `po_num` + supplier | **Weak**: the sample bill's `po_num` is an account number, so this link must be nullable |
| `bills.approval_info[].user_id` | `users._id` | ObjectId | Reliable |
| `users.cart.items` / `priceMap` keys | order lines | `"<SUP>:<SKU>"` | Reliable (same key pattern as `lowest_price`) |

---
