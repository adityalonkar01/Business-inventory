# Inventory Management System

## Run the desktop application

From the project directory, run:

```powershell
.\.venv\Scripts\python.exe login.py
```

After signing in as an administrator, choose **Products** to view and search
active products, add or edit product details, or safely deactivate a product.
Product records are managed through the existing SQLite product service.

Managers open a dedicated dashboard with live inventory, sales, purchase, and
stock-movement summaries. They can search and view active products, manage
inventory, review aggregate sales, create and receive purchases, access
business reports, and change their own password. Product catalog editing,
categories, supplier administration, settings, and user administration remain
administrator-only.

Salespeople open a dedicated dashboard with their own daily, weekly, and
monthly sales, invoice and item counts, recent transactions, and read-only
low-stock alerts. **Sales & Billing** provides the searchable in-stock product
catalogue and checkout, **Sales History** filters their own invoices by number
and date, **Reports** provides date-range summaries and CSV export of their
sales only, and **Profile** allows them to change their own password.

Choose **Inventory** as an administrator to view stock levels, receive stock,
make signed stock adjustments, and review movement history. Managers have
inventory viewing and stock-management access. Salespeople have read-only
inventory and movement-history access.

Administrators and salespeople can use **Sales & Billing** to search active
products, build invoices, set GST rates per bill line, apply an invoice-level
discount, complete sales, and search their sales history. The default GST rate
is copied to newly added items, and can be overridden separately for each item.
Salespeople can view only their own invoices; administrators can view all
invoices. Managers can view aggregate sales totals and business reports, but
cannot create sales or access individual invoices.

Completed invoices preserve product name, SKU, quantity, selling price,
discount allocation, tax rate, tax amount, and post-discount line total at the
time of sale. GST is calculated once on each item's post-discount amount. Stock
reductions and explicit sale movements are committed with the invoice in a
single transaction. To print an invoice, open it from the completion prompt or
Sales History, choose **Save / Print Invoice**, and use the browser's print
command on the generated HTML invoice.

The inventory movement and sales tables are added using idempotent, additive
migrations when their services are first used (or when the database setup
script is run). Existing sales headers and movement records are retained.
Legacy sale items whose per-product tax rate was never recorded remain marked
as tax-not-recorded instead of being displayed as zero-tax items.

## Administrator modules

Administrators also have working **Categories**, **Suppliers**, **Purchases**,
**Reports**, **User Management**, and **Settings** pages.

- Categories are seeded from the existing product category values. Renaming a
  category updates the text stored on matching products. A category in use
  cannot be deactivated.
- Suppliers can be created, edited, and activated or deactivated. Existing
  purchase references remain linked to the supplier record.
- Purchases are saved as drafts with line-item snapshots. Stock is not changed
  until an administrator or manager confirms receipt. Receipt updates stock,
  updates the latest purchase price, records inventory movement, and marks the
  purchase received in one transaction; a received purchase cannot be received
  a second time.
- Reports include daily sales, top-selling products, low stock, stock valuation,
  purchase activity, and category counts. Date ranges can be selected as today,
  this week, this month, or entered directly. Administrators and managers can
  export authorized business reports as CSV; managers do not receive user
  administration or settings data.
- User administration preserves the existing PBKDF2-HMAC-SHA256 hash format.
  Administrators can create users, change roles, reset passwords, and activate
  or deactivate accounts. The last active administrator cannot be demoted or
  deactivated.
- Settings persist business contact details, currency symbol, default GST,
  invoice prefix, and default minimum stock. Business details and currency are
  copied into each newly completed invoice so later settings changes do not
  rewrite historical invoices. Currency cannot be changed once sales have
  been recorded, which keeps aggregate totals in one consistent denomination.
- Managers can work with purchase drafts and confirm stock receipts using
  active suppliers. Supplier and category administration, user management, and
  settings remain administrator-only.
- Salesperson reports and invoice searches are restricted to the authenticated
  salesperson's user ID. The existing invoice snapshots, pricing, GST,
  discount, stock checks, and atomic stock movements remain part of the
  existing sales service. Checkout refreshes the catalog before confirmation
  and the service rechecks price and stock inside its transaction.

These tables and invoice fields are introduced with additive `CREATE TABLE IF
NOT EXISTS`/`ALTER TABLE` migrations. The existing database file, users,
products, sales, and stock records are not reset or replaced. CSV exports are
created only at the location chosen by the user performing the authorized
export.

## Manually test inventory

1. Sign in using an existing admin or manager account and open **Inventory**.
2. Search for a product by name or SKU. Products at or below their minimum
   stock level are marked **LOW STOCK**.
3. Use **Receive** to add a positive quantity, or **Adjust** to enter a signed
   quantity (for example, `-2` to remove two units).
4. Open **Stock Movement History** to inspect the recorded actor, quantity,
   stock before and after, timestamp, and notes.
5. Sign in as a salesperson to verify inventory is visible but stock actions
   are not available. Managers can open **Sales Summary** for aggregate totals.

## Manually test manager modules

1. Run `.\.venv\Scripts\python.exe login.py` and sign in with an existing,
   active manager account.
2. Confirm the Manager Dashboard shows live product, stock, low-stock,
   inventory valuation, today's/month's sales, recent transactions, purchases,
   and stock movements.
3. Search **Products** by name and SKU and confirm price, stock, and minimum
   stock details are visible; managers cannot edit catalog records.
4. Open **Purchases**, create a draft using an active supplier, then receive it
   once and confirm stock history records the manager as the actor.
5. Review **Reports** with a date range and export an authorized report to CSV.
   Confirm **Sales Summary** shows aggregate totals only, not invoice details.
6. In **Profile**, enter the current password and a matching new password and
   confirm the change succeeds. A wrong current password must be rejected.
7. Confirm categories, supplier administration, settings, and user management
   are not available to the manager. Sign in as an administrator and verify
   administrator modules remain available.

## Manually test salesperson modules

1. Run `.\.venv\Scripts\python.exe login.py` and sign in with an existing,
   active salesperson account. Confirm the salesperson dashboard opens with
   their own sales and read-only low-stock information.
2. In **Sales & Billing**, search by product name or SKU, add a product, change
   its quantity, review GST/discount and totals, then confirm checkout.
3. Try a quantity above current stock. Checkout should refresh stock and reject
   insufficient quantity; the sale service also verifies stock and current
   selling prices atomically before committing.
4. Confirm a valid sale and verify the saved invoice, item snapshots, stock
   deduction, and sale movement. Open **Sales History**, filter by invoice and
   date, and view/print the saved invoice.
5. In **Reports**, choose a date range and export daily sales, best sellers, or
   invoice history. Confirm another salesperson's records are not included.
6. In **Profile**, verify a wrong current password is rejected and a matching
   confirmed password change succeeds.

Customer records and return/refund workflows are not currently part of the
database or sales service. Sales continue to support the existing no-customer
invoice workflow. Returns are intentionally not simulated or implemented
without an auditable return/refund schema and policy.

## Manually test sales and billing

1. Sign in as an administrator or salesperson and open **Sales & Billing**.
2. Search for an active product, add it to the bill, and adjust its quantity.
   The displayed selling price is read from the product record.
3. Set a default GST rate (0–100%) before adding products, then review or
   override the GST rate on each bill line. Apply an invoice discount (0–100%)
   and verify the displayed tax and final total.
4. Complete the sale and inspect the generated invoice. The product stock
   quantity decreases and Inventory > Stock Movement History records a Sale.
5. Open **Sales History**, search by invoice number, and open the invoice to
   save a print-ready HTML document.
6. Try a quantity above available stock; the sale must be rejected without
   changing stock or creating an invoice.
7. Sign in as a salesperson and confirm only that salesperson's invoices are
   visible. Managers do not have Sales & Billing access.

## Run product service tests

```powershell
.\.venv\Scripts\python.exe test_product_service_unit.py
```

These tests use an isolated temporary SQLite database and do not modify
`database\inventory.db`.

Run the complete automated suite with:

```powershell
.\.venv\Scripts\python.exe -m unittest discover -v
```

The automated inventory tests use temporary SQLite databases; the interactive
console scripts are guarded so test discovery does not prompt for input.
Sales service tests also use isolated temporary SQLite databases and exercise
transaction rollback, per-item tax calculations, salesperson-specific
dashboard/report data, date-filtered invoice history, CSV export, role
permissions, historical invoice snapshots, and safe migration of pre-existing
sales and inventory movement records. `test_admin_modules.py` covers account authorization and
password hashing, category migration, supplier and purchase receiving,
transaction rollback, persistent settings, report calculations, CSV export,
and report authorization; it also uses an isolated temporary database.
