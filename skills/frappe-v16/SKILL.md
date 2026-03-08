---
name: frappe-framework
description: >
  Use this skill whenever the user wants to develop applications with the Frappe Framework or ERPNext.
  Triggers include: creating DocTypes, writing controllers, defining hooks, building Frappe apps,
  customizing ERPNext, writing whitelisted API methods, form scripts (JS), bench commands,
  migrations, permissions, reports (Script/Query/Report Builder), REST API, background jobs,
  portal pages, web forms, realtime events (socket.io), Query Builder (frappe.qb), caching,
  virtual doctypes, server scripts, client scripts, testing, translations, print formats,
  webhooks, OAuth2, or any task involving Frappe/ERPNext development.
  Use this skill even when the user just mentions "frappe", "erpnext", "doctype", "bench",
  "frappe.whitelist", "frappe.get_doc", "frappe.qb", "frm.refresh", "hooks.py", or any
  Frappe/ERPNext-related term — even casually. Always consult this skill before writing
  any Frappe-related code.
---

# Frappe Framework Development Skill

Frappe is a full-stack, batteries-included Python + JavaScript web framework built on MariaDB (or PostgreSQL).
It is **metadata-first**: define DocTypes → Frappe auto-generates DB tables, REST endpoints, form UIs, list views, and permissions. ERPNext is the flagship app built on Frappe.

**Docs**: https://docs.frappe.io/framework/user/en/introduction — read before writing unfamiliar APIs.

---

## Core Concepts

| Concept | Description |
|---|---|
| **Bench** | Workspace hosting multiple apps and sites. CLI: `bench` |
| **App** | Python package inside `apps/`. Created with `bench new-app` |
| **Site** | Isolated DB + config under `sites/`. Created with `bench new-site` |
| **DocType** | Fundamental building block — defines a DB table, form UI, and behavior |
| **Document** | An instance of a DocType (a single DB row) |
| **Controller** | Python class (`Document` subclass) with business logic |
| **Hooks** | `hooks.py` — app-level event listeners, overrides, scheduled tasks |
| **Desk** | Browser admin UI (list views, form views, reports, workspaces) |
| **Portal** | Public-facing web pages served at root URL |

---

## Project Directory Structure

```
frappe-bench/
├── apps/
│   ├── frappe/                         ← Core framework
│   └── my_app/
│       ├── my_app/
│       │   ├── hooks.py                ← App hooks
│       │   ├── patches.txt             ← Migration patches list
│       │   ├── public/
│       │   │   ├── js/                 ← JS assets bundled into desk
│       │   │   └── css/
│       │   ├── templates/
│       │   │   ├── pages/              ← Portal page templates
│       │   │   └── generators/
│       │   ├── www/                    ← Portal pages (route = filename)
│       │   │   ├── orders.html         ← /orders
│       │   │   └── orders.py           ← context for orders.html
│       │   └── my_module/
│       │       └── doctype/
│       │           └── my_doctype/
│       │               ├── my_doctype.json    ← DocType definition
│       │               ├── my_doctype.py      ← Controller
│       │               ├── my_doctype.js      ← Form script
│       │               └── test_my_doctype.py ← Unit tests
│       ├── setup.py
│       └── requirements.txt
└── sites/
    └── mysite.localhost/
        └── site_config.json
```

---

## DocTypes

### DocType Variants

| Variant | Description | Key Traits |
|---|---|---|
| **Standard** | Normal persistent DocType | DB table `tab{DocType}` |
| **Child / Table** | Embedded in parent via Table field | `istable=1`, no standalone list view |
| **Single** | One global record (settings) | No `name` column; use `frappe.db.get_single_value()` |
| **Virtual** | No DB table; custom data source | Requires `get_list`, `get_doc`, `get_count` in controller |
| **Submittable** | Can be submitted/cancelled | `is_submittable=1`; docstatus 0/1/2 |

### Docstatus Values
- `0` = Draft (default, editable)
- `1` = Submitted (locked)
- `2` = Cancelled

### DocType JSON Structure

```json
{
  "name": "My DocType",
  "module": "My Module",
  "doctype": "DocType",
  "is_submittable": 0,
  "track_changes": 1,
  "autoname": "naming_series:",
  "fields": [
    {"fieldname": "title", "fieldtype": "Data", "label": "Title", "reqd": 1, "in_list_view": 1},
    {"fieldname": "status", "fieldtype": "Select", "options": "Open\nIn Progress\nClosed", "default": "Open"},
    {"fieldname": "linked_customer", "fieldtype": "Link", "options": "Customer"},
    {"fieldname": "amount", "fieldtype": "Currency", "precision": 2},
    {"fieldname": "sec_items", "fieldtype": "Section Break", "label": "Items"},
    {"fieldname": "items", "fieldtype": "Table", "options": "My Child DocType"}
  ],
  "permissions": [
    {"role": "System Manager", "read": 1, "write": 1, "create": 1, "delete": 1},
    {"role": "Sales User", "read": 1, "write": 1, "create": 1}
  ]
}
```

### All Field Types

| Category | Types |
|---|---|
| **Text** | `Data`, `Small Text`, `Text`, `Long Text`, `Text Editor`, `Markdown Editor`, `HTML Editor`, `Code`, `Password` |
| **Number** | `Int`, `Float`, `Currency`, `Percent`, `Rating` |
| **Date/Time** | `Date`, `Time`, `Datetime`, `Duration` |
| **Choice** | `Select`, `Check`, `Autocomplete` |
| **Relations** | `Link`, `Dynamic Link`, `Table`, `Table MultiSelect` |
| **Media** | `Attach`, `Attach Image`, `Image`, `Signature`, `Geolocation` |
| **Layout** | `Section Break`, `Column Break`, `Tab Break`, `Fold` |
| **Info** | `HTML`, `Heading`, `Read Only` |
| **Special** | `Color`, `Icon`, `Phone`, `Name`, `Barcode`, `JSON` |

### Important DocField Properties

```
reqd                  1 = mandatory
read_only             1 = display only
hidden                1 = hidden (still in DB)
in_list_view          1 = show as list column
in_standard_filter    1 = add as filter
in_global_search      1 = include in global search
search_index          1 = add DB index
unique                1 = unique constraint
default               static or dynamic ("Today", "Now")
depends_on            conditional display: eval:doc.status=="Open"
mandatory_depends_on  conditional mandatory
read_only_depends_on  conditional read-only
fetch_from            auto-fill from link: "customer.customer_name"
fetch_if_empty        1 = only fetch when field is blank
allow_on_submit       1 = editable after submit
precision             decimal places for Float/Currency
```

### Naming Strategies (`autoname`)

```python
"autoname": "naming_series:"            # Uses Naming Series field
"autoname": "field:title"               # Uses value of `title`
"autoname": "format:{customer}-{##}"    # Pattern with padding
"autoname": "hash"                      # Random hash

# Custom — override autoname() in controller:
def autoname(self):
    self.name = f"CUSTOM-{self.customer}-{frappe.utils.nowdate()}"
```

### Virtual DocField

Field visible in form but not stored in DB. Computed on the fly:

```python
class MyDocType(Document):
    @property
    def full_name(self):
        return f"{self.first_name} {self.last_name}"
```

In JSON: `"fieldtype": "Data", "is_virtual": 1`

---

## Controllers (Python)

```python
import frappe
from frappe import _
from frappe.model.document import Document

class MyDocType(Document):

    def before_insert(self):
        """Before first save. doc.name may not be set yet."""
        pass

    def validate(self):
        """Before every save (insert + update). Raise to block."""
        if self.amount < 0:
            frappe.throw(_("Amount cannot be negative"))
        self.total = self.amount + (self.tax or 0)

    def before_save(self):
        """Just before DB write."""
        self.last_updated = frappe.utils.now_datetime()

    def after_insert(self):
        """Only after first insert."""
        frappe.sendmail(recipients=[self.email], subject="Created", message=self.name)

    def after_save(self):
        """After every save."""
        pass

    def on_submit(self):
        """docstatus 0→1. Create downstream entries here."""
        self.create_ledger_entries()

    def on_cancel(self):
        """docstatus 1→2. Reverse on_submit actions."""
        self.reverse_ledger_entries()

    def on_trash(self):
        """Before deletion. Raise to prevent delete."""
        if self.status == "Active":
            frappe.throw(_("Cannot delete an Active record."))

    def after_delete(self):
        pass

    def on_change(self):
        """After any change. Avoid heavy logic here."""
        pass

    def before_submit(self):
        """Validation before submit."""
        pass

    def before_cancel(self):
        """Validation before cancel."""
        pass

    def before_update_after_submit(self):
        """Before saving allow_on_submit fields."""
        pass
```

### Full Hook Execution Order

```
Insert:  before_insert → validate → before_save → db_insert → after_insert → after_save → on_change
Update:  validate → before_save → db_update → after_save → on_change
Submit:  before_submit → validate → on_submit → after_save → on_change
Cancel:  before_cancel → on_cancel → after_save → on_change
Delete:  on_trash → db_delete → after_delete
```

---

## Document API (Python)

### Fetch / Create

```python
import frappe

# Full Document object (permission check by default)
doc = frappe.get_doc("Customer", "CUST-0001")
doc = frappe.get_doc("Customer", "CUST-0001", ignore_permissions=True)

# Cached read (do NOT modify — shared reference)
doc = frappe.get_cached_doc("Customer", "CUST-0001")

# New unsaved document
doc = frappe.new_doc("Sales Order")
doc.customer = "CUST-0001"
doc.insert()                          # triggers before_insert, validate, after_insert
doc.insert(ignore_permissions=True)

# Save existing
doc = frappe.get_doc("Task", "TASK-0001")
doc.status = "Closed"
doc.save()

# Submit / Cancel
doc.submit()
doc.cancel()

# Delete
frappe.delete_doc("Task", "TASK-0001", ignore_permissions=True)

# Single DocType
site = frappe.get_single("Website Settings")
val = frappe.db.get_single_value("Website Settings", "home_page")
```

### Query Lists

```python
results = frappe.get_all(
    "Task",
    filters={
        "status": "Open",
        "priority": ["in", ["High", "Medium"]],
        "creation": [">", "2024-01-01"],
        "name": ["like", "TASK-%"],
        "owner": ["is", "set"],
    },
    or_filters={"assigned_to": frappe.session.user, "owner": frappe.session.user},
    fields=["name", "subject", "due_date"],
    order_by="due_date asc",
    limit=50,
    start=0,           # offset for pagination
    as_list=False,
    ignore_permissions=False,
)

# Filter operators: =, !=, <, >, <=, >=, like, not like, in, not in, is, is not, between
count = frappe.db.count("Task", {"status": "Open"})
```

### Database Operations

```python
# Read single value
value = frappe.db.get_value("Customer", "CUST-001", "customer_name")
# Multiple values as dict
data  = frappe.db.get_value("Customer", "CUST-001", ["customer_name", "email_id"], as_dict=True)
# With filter
name  = frappe.db.get_value("Customer", {"email_id": "x@example.com"}, "name")

# Set value directly (fast, bypasses hooks)
frappe.db.set_value("Task", "TASK-001", "status", "Closed")
frappe.db.set_value("Task", "TASK-001", {"status": "Closed", "completed_on": frappe.utils.today()})

# Exists check
if frappe.db.exists("Customer", "CUST-001"):
    pass
if frappe.db.exists("Task", {"subject": "My Task"}):
    pass

# Raw SQL (bypasses hooks and permissions — use sparingly)
rows = frappe.db.sql(
    "SELECT name, status FROM `tabTask` WHERE owner = %(user)s",
    {"user": frappe.session.user},
    as_dict=True
)
frappe.db.commit()   # Required after raw DML SQL

# Delete
frappe.db.delete("Task", {"status": "Cancelled"})
```

### Query Builder (frappe.qb)

Safe, composable query building without raw SQL:

```python
from frappe.query_builder import DocType
from frappe.query_builder.functions import Count, Sum

Task     = DocType("Task")
Customer = DocType("Customer")

# Simple select
results = (
    frappe.qb.from_(Task)
    .select(Task.name, Task.status, Task.subject)
    .where(Task.status == "Open")
    .orderby(Task.creation, order=frappe.qb.desc)
    .limit(20)
    .run(as_dict=True)
)

# Join
results = (
    frappe.qb.from_(Task)
    .join(Customer).on(Task.customer == Customer.name)
    .select(Task.name, Task.subject, Customer.customer_name)
    .where((Task.status == "Open") & (Customer.disabled == 0))
    .run(as_dict=True)
)

# Aggregation
results = (
    frappe.qb.from_(Task)
    .select(Task.status, Count(Task.name).as_("count"), Sum(Task.estimated_time).as_("total_hrs"))
    .groupby(Task.status)
    .run(as_dict=True)
)

# Update via qb
frappe.qb.update(Task).set(Task.status, "Closed").where(Task.due_date < "2024-01-01").run()
```

### Child Tables

```python
doc = frappe.get_doc("Sales Order", "SO-0001")

# Iterate rows
for item in doc.items:
    print(item.item_code, item.qty, item.rate)

# Append a row
doc.append("items", {"item_code": "ITEM-001", "qty": 5, "rate": 100, "uom": "Nos"})
doc.save()

# Remove rows by condition
doc.items = [row for row in doc.items if row.item_code != "ITEM-001"]
doc.save()

# In child table event (controller or hooks):
# cdt = child doctype name, cdn = row name
def validate(self):
    for row in self.items:
        row.amount = row.qty * row.rate
```

---

## Hooks (`hooks.py`)

```python
app_name      = "my_app"
app_title     = "My App"
app_publisher = "My Company"
app_version   = "0.0.1"

# ─── Document Events ──────────────────────────────────────────────────────────
doc_events = {
    "Sales Order": {
        "on_submit":   "my_app.events.sales_order.on_submit",
        "on_cancel":   "my_app.events.sales_order.on_cancel",
        "validate":    "my_app.events.sales_order.validate",
        "after_insert":"my_app.events.sales_order.after_insert",
    },
    "*": {   # All DocTypes
        "on_trash": "my_app.events.audit.log_deletion"
    }
}

# ─── Scheduled Tasks ──────────────────────────────────────────────────────────
scheduler_events = {
    "all":          ["my_app.tasks.every_minute"],
    "hourly":       ["my_app.tasks.hourly_sync"],
    "daily":        ["my_app.tasks.daily_cleanup"],
    "weekly":       ["my_app.tasks.weekly_report"],
    "monthly":      ["my_app.tasks.monthly_invoice"],
    "hourly_long":  ["my_app.tasks.long_hourly_job"],
    "daily_long":   ["my_app.tasks.long_daily_job"],
    "cron": {
        "0 9 * * 1-5":  ["my_app.tasks.morning_standup"],
        "*/15 * * * *": ["my_app.tasks.every_15_min"],
    }
}

# ─── Class Overrides ──────────────────────────────────────────────────────────
override_doctype_class = {
    "Sales Invoice": "my_app.overrides.CustomSalesInvoice"
}
override_whitelisted_methods = {
    "frappe.client.get_list": "my_app.api.custom_get_list"
}

# ─── Fixtures ─────────────────────────────────────────────────────────────────
fixtures = [
    "Custom Field",
    "Property Setter",
    "Print Format",
    {"dt": "Role",     "filters": [["name", "like", "My App %"]]},
    {"dt": "Workflow", "filters": [["document_type", "=", "Sales Order"]]},
]

# ─── Assets ───────────────────────────────────────────────────────────────────
app_include_js  = ["/assets/my_app/js/my_app.min.js"]
app_include_css = ["/assets/my_app/css/my_app.min.css"]
doctype_js      = {"Sales Order": "public/js/sales_order.js"}
doctype_list_js = {"Customer":    "public/js/customer_list.js"}

# ─── Portal ───────────────────────────────────────────────────────────────────
website_route_rules = [{"from_route": "/orders/<n>", "to_route": "order-detail"}]
website_generators  = ["Blog Post", "Web Page"]

# ─── Permissions ──────────────────────────────────────────────────────────────
permission_query_conditions = {
    "Task": "my_app.permissions.get_task_permission_query"
}
has_permission = {
    "Task": "my_app.permissions.has_task_permission"
}

# ─── Session ──────────────────────────────────────────────────────────────────
boot_session         = "my_app.boot.boot_session"
on_session_creation  = "my_app.auth.on_session_creation"
on_logout            = "my_app.auth.on_logout"

# ─── Jinja Globals ────────────────────────────────────────────────────────────
jinja = {
    "methods": ["my_app.utils.jinja.format_currency"],
    "filters": ["my_app.utils.jinja.upper_case"],
}

# ─── CORS ─────────────────────────────────────────────────────────────────────
allow_cors = ["https://my-frontend.example.com"]
```

---

## Whitelisted API Methods

```python
import frappe
from frappe import _

@frappe.whitelist()
def get_customer_data(customer):
    """Requires login."""
    frappe.has_permission("Customer", "read", throw=True)
    doc = frappe.get_doc("Customer", customer)
    return {"name": doc.name, "customer_name": doc.customer_name}

@frappe.whitelist(allow_guest=True)
def public_api():
    """No login required."""
    return {"version": frappe.__version__}

@frappe.whitelist(methods=["POST"])
def create_record(data):
    """POST only. data auto-parsed from JSON."""
    doc = frappe.get_doc(frappe.parse_json(data))
    doc.insert(ignore_permissions=True)
    return doc.name
```

### Calling from JS

```javascript
// frappe.call (callback style)
frappe.call({
    method: "my_app.api.get_customer_data",
    args: { customer: frm.doc.customer },
    freeze: true,
    freeze_message: __("Loading..."),
    callback(r) {
        if (r.message) frm.set_value("customer_name", r.message.customer_name);
    }
});

// frappe.xcall (Promise style)
frappe.xcall("my_app.api.get_customer_data", { customer: "CUST-001" })
    .then(data => console.log(data))
    .catch(err => frappe.msgprint(err));
```

---

## Form Scripts (JavaScript)

```javascript
frappe.ui.form.on("My DocType", {

    setup(frm) {
        // Runs once before data loads
    },

    onload(frm) {
        // set_query for link filters
        frm.set_query("linked_customer", () => ({
            filters: { disabled: 0 }
        }));
        // In child table:
        frm.set_query("item_code", "items", (doc, cdt, cdn) => ({
            filters: { is_stock_item: 1 }
        }));
    },

    refresh(frm) {
        if (frm.doc.docstatus === 1) {
            frm.add_custom_button(__("Create Invoice"), () => {
                frappe.call({
                    method: "my_app.api.create_invoice",
                    args: { source: frm.docname },
                    callback(r) { frappe.set_route("Form", "Sales Invoice", r.message); }
                });
            }, __("Actions")); // optional group
        }
    },

    // Field change event
    amount(frm) {
        frm.set_value("tax", flt(frm.doc.amount) * 0.1);
        frm.refresh_field("tax");
    },

    before_save(frm) {
        if (!frm.doc.items || !frm.doc.items.length) {
            frappe.throw(__("Add at least one item."));
        }
    },

    after_save(frm) {
        frappe.show_alert({ message: __("Saved!"), indicator: "green" });
    },

    on_submit(frm) {},
    after_cancel(frm) {}
});

// Child table events
frappe.ui.form.on("My Child DocType", {
    items_add(frm, cdt, cdn) {
        frappe.model.set_value(cdt, cdn, "qty", 1);
    },
    qty(frm, cdt, cdn) {
        let row = locals[cdt][cdn];
        frappe.model.set_value(cdt, cdn, "amount", row.qty * row.rate);
        frm.refresh_field("items");
    },
    items_remove(frm, cdt, cdn) {
        frm.refresh_field("items");
    }
});
```

### frm API Quick Reference

```javascript
// Values
frm.set_value("fieldname", value);
frm.get_field("fieldname");
frm.doc.fieldname;

// Display properties
frm.set_df_property("fieldname", "hidden", 1);
frm.set_df_property("fieldname", "reqd", 1);
frm.set_df_property("fieldname", "read_only", 1);
frm.set_df_property("fieldname", "options", "A\nB\nC"); // Select
frm.toggle_display("fieldname", false);
frm.toggle_enable("fieldname", false);
frm.toggle_reqd("fieldname", true);
frm.refresh_field("fieldname");
frm.refresh_fields(["f1", "f2"]);

// State
frm.docname;
frm.doctype;
frm.is_new();
frm.is_dirty();
frm.dirty();          // mark unsaved

// Actions
frm.save();
frm.save("Submit");
frm.save("Cancel");
frm.scroll_to_field("fieldname");
frm.reload_doc();

// Dashboard links
frm.dashboard.add_transactions([{label: "Invoice", items: [frm.docname]}]);
```

---

## Dialog API (JavaScript)

```javascript
// Confirm
frappe.confirm("Are you sure?", () => { /* yes */ }, () => { /* no */ });

// Prompt (single field)
frappe.prompt(
    {fieldname: "reason", fieldtype: "Small Text", label: "Reason", reqd: 1},
    (values) => console.log(values.reason),
    "Enter Reason",
    "Submit"
);

// Full dialog
let d = new frappe.ui.Dialog({
    title: "My Dialog",
    fields: [
        {fieldname: "customer", fieldtype: "Link", options: "Customer", label: "Customer", reqd: 1},
        {fieldname: "col_break", fieldtype: "Column Break"},
        {fieldname: "date", fieldtype: "Date", label: "Date", default: frappe.datetime.get_today()},
    ],
    primary_action_label: "Submit",
    primary_action(values) {
        console.log(values.customer);
        d.hide();
    }
});
d.show();
d.set_value("customer", "CUST-001");
let val = d.get_value("customer");

// Message/Alert
frappe.msgprint({ title: __("Done"), indicator: "green", message: __("Completed.") });
frappe.show_alert({ message: __("Saved!"), indicator: "green" }, 3);
```

---

## REST API

### Endpoints

```
GET    /api/resource/{DocType}                 list docs
POST   /api/resource/{DocType}                 create doc
GET    /api/resource/{DocType}/{name}          get doc
PUT    /api/resource/{DocType}/{name}          update doc
DELETE /api/resource/{DocType}/{name}          delete doc
POST   /api/method/{dotted.python.path}        call whitelisted method
```

### Authentication

```bash
# 1. API Key + Secret (token auth) — preferred for integrations
curl -H "Authorization: token <api_key>:<api_secret>" \
     https://site.example.com/api/resource/Customer

# 2. Basic Auth
curl -u "admin:password" https://site.example.com/api/resource/Customer

# 3. OAuth2 Bearer
curl -H "Authorization: Bearer <access_token>" \
     https://site.example.com/api/resource/Customer
```

### Filtering via URL

```
/api/resource/Task?filters=[["status","=","Open"],["priority","=","High"]]
                  &fields=["name","subject","due_date"]
                  &limit=20&order_by=due_date asc
```

### CRUD Examples

```bash
# Create
curl -X POST https://site.example.com/api/resource/Customer \
  -H "Authorization: token key:secret" -H "Content-Type: application/json" \
  -d '{"customer_name":"New Co","customer_type":"Company"}'

# Update
curl -X PUT https://site.example.com/api/resource/Customer/CUST-001 \
  -H "Authorization: token key:secret" -H "Content-Type: application/json" \
  -d '{"phone":"+1234567890"}'

# Call whitelisted method
curl -X POST https://site.example.com/api/method/my_app.api.get_customer_data \
  -H "Authorization: token key:secret" -H "Content-Type: application/json" \
  -d '{"customer":"CUST-001"}'
```

---

## Background Jobs

```python
import frappe

# Enqueue a job (runs in Redis/RQ worker)
frappe.enqueue(
    "my_app.tasks.heavy_computation",
    queue="long",           # "default"(5min), "short"(5min), "long"(1hr)
    timeout=3600,
    job_id="unique-job-id", # prevents duplicate enqueueing
    enqueue_after_commit=True,
    # kwargs:
    campaign="CAMP-001",
    batch_size=100,
)

# Enqueue at specific time
frappe.enqueue(
    "my_app.tasks.send_reminder",
    queue="default",
    eta=frappe.utils.add_days(frappe.utils.now_datetime(), 1),
    doc_name="TASK-001"
)

# Enqueue a method on the current doc
def on_submit(self):
    frappe.enqueue_doc(self.doctype, self.name, "process_after_submit", queue="long")

def process_after_submit(self):
    pass  # runs in background worker

# Realtime progress updates from background job
def long_task(total):
    for i, item in enumerate(get_items()):
        process(item)
        frappe.publish_progress(
            percent=(i + 1) / total * 100,
            title="Processing",
            description=f"{i+1}/{total}"
        )
```

---

## Realtime (Socket.io)

```python
# Server → Client push
frappe.publish_realtime(
    event="my_event",
    message={"status": "done", "name": self.name},
    user=frappe.session.user,
    doctype="My DocType",
    docname=self.name,
    after_commit=True,
)
```

```javascript
// Client — subscribe
frappe.realtime.on("my_event", (data) => {
    console.log(data.status);
    frm.reload_doc();
});

// Subscribe to a doc room (receives all doc-level events)
frappe.realtime.doc_subscribe("Sales Order", "SO-0001");
frappe.realtime.doc_unsubscribe("Sales Order", "SO-0001");
```

---

## Caching

```python
# Redis (site-level, persistent across requests)
frappe.cache.set_value("my_key", {"data": 123}, expires_in_sec=3600)
value = frappe.cache.get_value("my_key")
frappe.cache.delete_value("my_key")

# Per-user hash
frappe.cache.hset("namespace", frappe.session.user, data)
data = frappe.cache.hget("namespace", frappe.session.user)

# Decorator caching
from frappe.utils.caching import site_cache, redis_cache

@site_cache(ttl=3600)
def get_expensive_data(param):
    return frappe.db.sql("...", as_dict=True)

@redis_cache(ttl=300)
def get_config():
    return frappe.get_single("My Settings")

# Request-level (in-memory, cleared per request)
frappe.local.flags.my_flag = True
```

---

## Permissions (Advanced)

### Role Permissions JSON

```json
{"role": "Sales Manager", "read": 1, "write": 1, "create": 1, "delete": 1,
 "submit": 1, "cancel": 1, "amend": 1, "report": 1, "import": 1, "export": 1,
 "print": 1, "email": 1, "share": 1}
```

### Dynamic Permission Query (hooks.py)

```python
# hooks.py
permission_query_conditions = {
    "Task": "my_app.permissions.get_task_query"
}

# my_app/permissions.py
def get_task_query(user):
    if user == "Administrator":
        return ""
    u = frappe.db.escape(user)
    return f"(`tabTask`.assigned_to = {u} OR `tabTask`.owner = {u})"
```

### `has_permission` Hook

```python
# hooks.py
has_permission = {"Project": "my_app.permissions.has_project_permission"}

def has_project_permission(doc, ptype, user):
    if ptype == "read":
        return doc.status != "Confidential" or user == doc.project_manager
    return None  # fall back to standard check
```

### Code Checks

```python
frappe.has_permission("Sales Order", "write", doc=doc, throw=True)
# throw=True raises frappe.PermissionError if access denied
```

### User Permissions

```python
frappe.permissions.add_user_permission("Customer", "CUST-001", "user@example.com")
frappe.permissions.remove_user_permission("Customer", "CUST-001", "user@example.com")
```

---

## Reports

### Script Report (Python)

```python
# my_report.py
import frappe
from frappe import _

def execute(filters=None):
    columns = [
        {"label": _("Customer"), "fieldname": "customer", "fieldtype": "Link",
         "options": "Customer", "width": 200},
        {"label": _("Total"),    "fieldname": "total",    "fieldtype": "Currency", "width": 120},
    ]
    data = frappe.db.sql("""
        SELECT customer, SUM(grand_total) AS total
        FROM `tabSales Invoice`
        WHERE docstatus=1
          AND posting_date BETWEEN %(from_date)s AND %(to_date)s
        GROUP BY customer ORDER BY total DESC
    """, filters, as_dict=True)

    chart = {
        "data": {
            "labels": [d.customer for d in data],
            "datasets": [{"values": [d.total for d in data]}]
        },
        "type": "bar"
    }
    summary = [{"label": "Grand Total", "value": sum(d.total for d in data),
                "datatype": "Currency", "currency": "USD"}]
    return columns, data, None, chart, summary
```

```json
// my_report.json
{
  "report_name": "My Report",
  "report_type": "Script Report",
  "ref_doctype": "Sales Invoice",
  "is_standard": "Yes",
  "filters": [
    {"fieldname": "from_date", "fieldtype": "Date", "label": "From Date", "reqd": 1},
    {"fieldname": "to_date",   "fieldtype": "Date", "label": "To Date",   "reqd": 1}
  ]
}
```

### Query Report (SQL-only)

```python
def execute(filters=None):
    columns = [
        {"label": "Name",   "fieldname": "name",   "fieldtype": "Link", "options": "Task"},
        {"label": "Status", "fieldname": "status",  "fieldtype": "Data"},
    ]
    data = frappe.db.sql(
        "SELECT name, status FROM `tabTask` WHERE status = %(status)s",
        filters, as_dict=True
    )
    return columns, data
```

---

## Portal Pages

### `www/` Folder Structure

```
my_app/www/
├── orders.html          → /orders
├── orders.py            → context for /orders
└── orders/
    └── index.html       → /orders/ (directory index)
```

```python
# www/orders.py
import frappe

def get_context(context):
    if frappe.session.user == "Guest":
        frappe.local.flags.redirect_location = "/login"
        raise frappe.Redirect
    context.orders = frappe.get_all(
        "Sales Order",
        filters={"customer": frappe.session.user},
        fields=["name", "grand_total", "transaction_date"]
    )
    context.title = "My Orders"
```

```html
{# www/orders.html #}
{% extends "templates/web.html" %}
{% block page_content %}
<h1>{{ title }}</h1>
{% for order in orders %}
<div class="order-row">{{ order.name }} — {{ order.grand_total }}</div>
{% endfor %}
{% endblock %}
```

### Generators (One URL per Record)

Enable "Has Web View" on the DocType and add `route` and `published` fields. Then in controller:

```python
def get_context(self, context):
    context.title = self.title
    context.parents = [{"title": "All Products", "route": "/products"}]
```

---

## Web Forms

Web Forms expose a DocType form to portal/guest users:

```javascript
// Web Form Client Script
frappe.web_form.after_load = function() {
    frappe.web_form.set_df_property("amount", "hidden", 1);
};

frappe.web_form.validate = function() {
    let values = frappe.web_form.get_values();
    if (!values.amount) {
        frappe.throw("Amount is required");
        return false;
    }
};

frappe.web_form.on_submit = function(data) {
    console.log("Submitted:", data);
};
```

---

## Server Scripts (Desk UI, No Deploy)

Types of Server Scripts:

| Type | Trigger | Use Case |
|---|---|---|
| **DocType Event** | Document lifecycle event | Like a controller hook, no deploy needed |
| **Scheduler Event** | Cron-like schedule | Background automation |
| **API** | HTTP call to `/api/method/script_name` | Rapid endpoint creation |
| **Permission Query** | Permission filter | Dynamic row-level security |

```python
# DocType Event script — context: doc, frappe
if doc.amount > 10000:
    doc.needs_approval = 1
    frappe.sendmail(
        recipients=["manager@example.com"],
        subject=f"High value: {doc.name}",
        message=f"Review {doc.name} (amount: {doc.amount})"
    )
```

Client Scripts (Desk UI) replace `.js` for no-deploy customization:

```javascript
// context: frm, frappe
frappe.ui.form.on(frm.doctype, {
    refresh(frm) {
        frm.set_df_property("po_no", "reqd", frm.doc.order_type === "Purchase");
    }
});
```

---

## Bench CLI Reference

```bash
# ─── Setup ────────────────────────────────────────────────────────────────────
bench init frappe-bench --frappe-branch version-15
bench new-app my_app
bench new-site mysite.localhost --install-app frappe --install-app my_app
bench --site mysite.localhost install-app my_app
bench get-app https://github.com/frappe/erpnext --branch version-15

# ─── Development ──────────────────────────────────────────────────────────────
bench start                                          # dev server
bench build                                          # bundle JS/CSS
bench --site mysite.localhost migrate                # apply schema changes
bench --site mysite.localhost console                # Python REPL
bench --site mysite.localhost execute "frappe.db.sql('SELECT 1')"
bench --site mysite.localhost set-config developer_mode 1
bench --site mysite.localhost set-config server_script_enabled 1
bench --site mysite.localhost clear-cache
bench --site mysite.localhost clear-website-cache

# ─── Apps & Sites ─────────────────────────────────────────────────────────────
bench --site mysite.localhost list-apps
bench --site mysite.localhost uninstall-app my_app
bench --site mysite.localhost drop-site --archived

# ─── Backup & Restore ─────────────────────────────────────────────────────────
bench --site mysite.localhost backup
bench --site mysite.localhost backup --with-files
bench --site mysite.localhost restore /path/backup.sql.gz

# ─── Patches ──────────────────────────────────────────────────────────────────
bench --site mysite.localhost run-patch my_app.patches.v2_0.my_patch
bench --site mysite.localhost migrate     # runs all pending patches

# ─── Fixtures ─────────────────────────────────────────────────────────────────
bench --site mysite.localhost export-fixtures --app my_app

# ─── Production ───────────────────────────────────────────────────────────────
sudo bench setup production frappe
bench setup nginx
bench setup supervisor
bench --site mysite.localhost enable-scheduler
bench --site mysite.localhost disable-maintenance-mode

# ─── Translations ─────────────────────────────────────────────────────────────
bench generate-pot-file --app my_app
bench update-po-files --app my_app
bench compile-po-to-mo --app my_app
```

---

## Database Migrations (Patches)

```python
# my_app/patches/v2_0/rename_field.py
import frappe
from frappe.model.utils.rename_field import rename_field

def execute():
    if frappe.db.has_column("Sales Order", "old_field"):
        rename_field("Sales Order", "old_field", "new_field")
    frappe.reload_doc("my_module", "doctype", "sales_order")
```

```
# patches.txt — append only, one patch per line
my_app.patches.v2_0.rename_field
my_app.patches.v2_0.populate_new_field
```

### Schema Helpers

```python
frappe.db.has_column("DocType", "fieldname")
frappe.db.add_column("DocType", "fieldname", "varchar(140)")
frappe.rename_doc("DocType", "old_name", "new_name", ignore_if_missing=True)
frappe.reload_doc("module", "doctype", "my_doctype")
```

---

## Jinja Templates (Print Formats, Emails)

```html
{# Available in all Jinja contexts #}
{{ doc.name }}
{{ doc.customer }}
{{ frappe.utils.fmt_money(doc.grand_total, currency=doc.currency) }}
{{ frappe.db.get_value("Company", doc.company, "company_name") }}

{% if doc.status == "Submitted" %}<b>SUBMITTED</b>{% endif %}

{% for item in doc.items %}
<tr><td>{{ item.item_code }}</td><td>{{ item.qty }}</td><td>{{ item.rate }}</td></tr>
{% endfor %}

{{ _("Invoice") }}   {# translation #}
{{ frappe.utils.format_date(doc.posting_date, "dd MMM yyyy") }}
```

---

## Translations

```python
# Python
from frappe import _
_("Hello World")
_("Hello {0}", args=[user_name])
```

```javascript
// JavaScript
__("Hello World")
__("Hello {0}", [user_name])
```

---

## Utility Functions (Python)

```python
from frappe.utils import (
    nowdate, nowdatetime, now_datetime, today,
    add_days, add_months, date_diff,
    flt, cint, cstr,
    fmt_money, format_date, format_datetime,
    get_url, generate_hash,
)

today()                          # "2024-01-15"
add_days(today(), 7)             # "2024-01-22"
date_diff("2024-02-01", today()) # int (days)
flt("3.14")                      # 3.14
cint("42")                       # 42
cstr(None)                       # "" (never None)
fmt_money(12345.67, currency="USD")  # "$12,345.67"

# Email
frappe.sendmail(
    recipients=["user@example.com"],
    subject="Hello",
    message="<p>Body</p>",
    attachments=[{"fname": "report.pdf", "fcontent": pdf_bytes}]
)
```

---

## Testing

```python
# test_my_doctype.py
import frappe
from frappe.tests.utils import FrappeTestCase

class TestMyDocType(FrappeTestCase):

    def test_basic_creation(self):
        doc = frappe.get_doc({"doctype": "My DocType", "title": "Test", "amount": 100})
        doc.insert()
        self.assertEqual(doc.amount, 100)

    def test_validation_error(self):
        with self.assertRaises(frappe.ValidationError):
            frappe.get_doc({"doctype": "My DocType", "title": "T", "amount": -1}).insert()

    def tearDown(self):
        frappe.db.rollback()
```

```bash
bench --site mysite.localhost run-tests --app my_app
bench --site mysite.localhost run-tests --doctype "My DocType"
bench --site mysite.localhost run-tests --module my_app.module.doctype.my_doctype.test_my_doctype
```

---

## Virtual DocTypes

```python
from frappe.model.document import Document

class MyVirtualDocType(Document):
    @staticmethod
    def get_list(args):
        # Must return list of dicts with at least {"name": ...}
        return [{"name": "VIRT-001", "title": "Virtual Record"}]

    @staticmethod
    def get_count(args):
        return 1

    @staticmethod
    def get_doc(kwargs):
        data = external_api.get(kwargs["name"])
        return {"name": kwargs["name"], "title": data["title"]}

    def db_insert(self, *args, **kwargs):
        external_api.create({"name": self.name, "title": self.title})

    def db_update(self, *args, **kwargs):
        external_api.update(self.name, {"title": self.title})

    def delete(self):
        external_api.delete(self.name)
```

---

## Customize Standard DocTypes (Without Modifying Core)

### Custom Fields

```python
from frappe.custom.doctype.custom_field.custom_field import create_custom_fields

create_custom_fields({
    "Customer": [
        {
            "fieldname": "custom_tier",
            "fieldtype": "Select",
            "label": "Customer Tier",
            "options": "Bronze\nSilver\nGold",
            "insert_after": "customer_name",
        }
    ]
}, ignore_validate=True, update=True)
```

### Property Setters

```python
frappe.make_property_setter({
    "doctype": "Sales Invoice",
    "fieldname": "due_date",
    "property": "reqd",
    "value": "1",
    "property_type": "Check"
})
```

---

## Logging & Debugging

```python
# User-visible
frappe.throw(_("Error message"))            # Raises exception, shows popup
frappe.msgprint(_("Info message"))          # Non-blocking popup
frappe.msgprint(_("Alert!"), alert=True)    # Top-bar alert

# Silent logging
frappe.log_error(frappe.get_traceback(), "Title")
frappe.log_error("message", "Title")

# Console output (visible in `bench start` logs)
print("debug:", value)
frappe.logger("my_app").debug("message")
frappe.logger("my_app").info("Info log")
```

```bash
tail -f logs/web.log
tail -f logs/worker.log
tail -f logs/scheduler.log
```

---

## Site Configuration (`site_config.json`)

```json
{
  "db_name": "mysite",
  "developer_mode": 1,
  "allow_tests": 1,
  "server_script_enabled": 1,
  "disable_website_cache": 0,
  "maintenance_mode": 0,
  "pause_scheduler": 0
}
```

---

## Critical Gotchas

**Save to persist — attributes alone don't write to DB:**
```python
doc.field = value
doc.save()   # or doc.insert() for new docs
```

**`frappe.db.commit()` only after raw SQL.** The ORM auto-commits.

**Exception types:**
```python
frappe.throw("msg")                   # frappe.ValidationError
frappe.throw("msg", frappe.PermissionError)
raise frappe.DoesNotExistError        # 404
raise frappe.DuplicateEntryError
```

**`bench migrate` is required** after any DocType JSON change.

**Developer Mode must be on** to write DocType definitions to disk.

**`frappe.get_doc` vs `frappe.get_all`:**
- `get_doc` → full Document with child tables loaded
- `get_all` / `get_list` → flat list of dicts (faster, no child tables)

**`frappe.get_cached_doc` → DO NOT modify** — shared memory reference.

**Table name convention:** `tab{DocType Name}` — e.g., `tabSales Order`.

**Fixtures = correct way** to distribute Custom Fields, Roles, Workflows, Print Formats. Declare in `hooks.py → fixtures`, run `bench export-fixtures`.

**`ignore_permissions=True`** — use in migrations, patches, background jobs where session user may lack rights.

**`frappe.session.user`** — current logged-in user. In background jobs it's often `"Administrator"`.

**Avoid raw `frappe.db.sql()` for CRUD** — use ORM or Query Builder to keep hooks and permission checks active.

---

## Reference URLs

| Topic | URL |
|---|---|
| Document API | https://docs.frappe.io/framework/user/en/api/document |
| Database API | https://docs.frappe.io/framework/user/en/api/database |
| Query Builder | https://docs.frappe.io/framework/user/en/api/query-builder |
| Hooks | https://docs.frappe.io/framework/user/en/python-api/hooks |
| Background Jobs | https://docs.frappe.io/framework/user/en/api/background_jobs |
| Realtime | https://docs.frappe.io/framework/user/en/api/realtime |
| Form Scripts JS | https://docs.frappe.io/framework/user/en/api/form |
| Dialog API JS | https://docs.frappe.io/framework/user/en/api/dialog |
| Controls JS | https://docs.frappe.io/framework/user/en/api/controls |
| Server Calls JS | https://docs.frappe.io/framework/user/en/api/server-calls |
| Portal Pages | https://docs.frappe.io/framework/user/en/portal-pages |
| Web Forms | https://docs.frappe.io/framework/user/en/web-form |
| Reports | https://docs.frappe.io/framework/user/en/desk/reports |
| Script Report | https://docs.frappe.io/framework/user/en/desk/reports/script-report |
| Query Report | https://docs.frappe.io/framework/user/en/desk/reports/query-report |
| REST API | https://docs.frappe.io/framework/user/en/api/rest |
| Caching | https://docs.frappe.io/framework/user/en/guides/caching |
| Testing | https://docs.frappe.io/framework/user/en/testing |
| Bench Commands | https://docs.frappe.io/framework/user/en/bench/bench-commands |
| Permissions | https://docs.frappe.io/framework/permission-types |
| Virtual DocTypes | https://docs.frappe.io/framework/user/en/basics/doctypes/virtual-doctype |
| Jinja API | https://docs.frappe.io/framework/user/en/api/jinja |
| Migrations | https://docs.frappe.io/framework/user/en/database-migrations |
| Translations | https://docs.frappe.io/framework/user/en/translations |
| Naming | https://docs.frappe.io/framework/user/en/basics/doctypes/naming |
| Field Types | https://docs.frappe.io/framework/user/en/basics/doctypes/fieldtypes |
| Server Scripts | https://docs.frappe.io/framework/user/en/desk/scripting/server-script |
| Notifications | https://docs.frappe.io/framework/notifications |
| Webhooks | https://docs.frappe.io/framework/user/en/guides/integration/webhooks |
