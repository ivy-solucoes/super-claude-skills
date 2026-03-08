---
name: frappe
description: >
  Use this skill whenever the user wants to develop applications with the Frappe Framework or ERPNext.
  Triggers include: creating DocTypes, writing controllers, defining hooks, building Frappe apps,
  customizing ERPNext, writing whitelisted API methods, form scripts (JS), bench commands,
  migrations, permissions, reports, REST API, background jobs, or any task involving Frappe/ERPNext
  development. Use this skill even when the user just mentions "frappe", "erpnext", "doctype",
  "bench", "frappe.whitelist", or "frappe.get_doc" — even casually. Always consult this skill
  before writing any Frappe-related code.
---

# Frappe Framework Development Skill

Frappe is a full-stack, batteries-included Python + JavaScript web framework built on MariaDB.
It is **metadata-first**: you define DocTypes (data models), and Frappe auto-generates DB tables,
REST endpoints, form UIs, list views, and permissions. ERPNext is the flagship app built on Frappe.

**Documentation**: https://docs.frappe.io/framework/user/en/introduction
When in doubt about any API, read the relevant docs page before writing code.

---

## Core Concepts

| Concept | Description |
|---|---|
| **Bench** | Workspace that hosts apps and sites. CLI: `bench` |
| **App** | Python package inside `apps/`. Created with `bench new-app` |
| **Site** | An isolated DB + config under `sites/`. Created with `bench new-site` |
| **DocType** | The fundamental building block — defines a DB table, form, and behavior |
| **Document** | An instance of a DocType (a single DB row) |
| **Controller** | Python class (`Document` subclass) that holds business logic for a DocType |
| **Hooks** | `hooks.py` — registers app-level event listeners, overrides, scheduled tasks |
| **Desk** | The browser-based admin UI (list views, form views, reports) |

---

## Project Structure

```
frappe-bench/
├── apps/
│   └── my_app/
│       ├── my_app/
│       │   ├── hooks.py               ← App hooks (events, overrides, schedulers)
│       │   ├── my_module/
│       │   │   └── doctype/
│       │   │       └── my_doctype/
│       │   │           ├── my_doctype.json    ← DocType definition
│       │   │           ├── my_doctype.py      ← Controller (server-side logic)
│       │   │           └── my_doctype.js      ← Form script (client-side logic)
│       └── setup.py
└── sites/
    └── mysite.localhost/
        └── site_config.json
```

---

## DocTypes

### Creating a DocType
Use the Desk UI (Developer Mode) or define JSON + Python manually.

**DocType JSON** (`my_doctype.json`) — key fields:
```json
{
  "name": "My DocType",
  "module": "My Module",
  "doctype": "DocType",
  "fields": [
    {"fieldname": "title", "fieldtype": "Data", "label": "Title", "reqd": 1},
    {"fieldname": "status", "fieldtype": "Select", "options": "Open\nClosed"},
    {"fieldname": "linked_doc", "fieldtype": "Link", "options": "Customer"},
    {"fieldname": "items", "fieldtype": "Table", "options": "My Child DocType"}
  ],
  "permissions": [{"role": "System Manager", "read": 1, "write": 1}]
}
```

### Common Field Types
`Data`, `Int`, `Float`, `Currency`, `Date`, `Datetime`, `Text`, `Long Text`,
`Select`, `Check`, `Link`, `Dynamic Link`, `Table`, `Attach`, `HTML`, `Heading`, `Column Break`, `Section Break`

### DocType Variants
- **Standard**: normal persistent DocType
- **Child/Table**: embedded in a parent via a Table field
- **Single**: only one record exists (like settings); access via `frappe.db.get_single_value()`
- **Virtual**: no DB table; data comes from custom Python source

---

## Controllers (Python)

```python
import frappe
from frappe.model.document import Document

class MyDocType(Document):
    # Auto-generated type hints (v15+) — do not edit manually
    # from frappe.types import DF
    # title: DF.Data
    # status: DF.Literal["Open", "Closed"]

    def validate(self):
        """Called before save. Raise frappe.ValidationError to block."""
        if not self.title:
            frappe.throw("Title is required")

    def before_save(self):
        self.modified_note = "Auto-updated"

    def after_insert(self):
        frappe.msgprint(f"Created: {self.name}")

    def on_submit(self):
        """Called when docstatus changes to 1 (Submitted)."""
        pass

    def on_cancel(self):
        """Called when docstatus changes to 2 (Cancelled)."""
        pass

    def on_trash(self):
        """Called before deletion."""
        pass
```

### Full Controller Hook Order (save flow)
`before_insert` → `validate` → `before_save` → `db_insert/db_update` → `after_insert/after_save` → `on_change`

### Document API (Python)
```python
# Get a document
doc = frappe.get_doc("Customer", "CUST-0001")

# New unsaved document
doc = frappe.new_doc("Sales Order")
doc.customer = "CUST-0001"
doc.insert()

# Save existing
doc = frappe.get_doc("Task", "TASK-0001")
doc.status = "Closed"
doc.save()

# Submit / Cancel
doc.submit()
doc.cancel()

# Delete
frappe.delete_doc("Task", "TASK-0001")

# Querying
results = frappe.get_all(
    "Task",
    filters={"status": "Open", "priority": ["in", ["High", "Medium"]]},
    fields=["name", "subject", "due_date"],
    order_by="due_date asc",
    limit=20
)

# Single value
owner = frappe.db.get_value("Task", "TASK-0001", "owner")

# Direct DB query (use sparingly)
rows = frappe.db.sql("SELECT name FROM `tabTask` WHERE status='Open'", as_dict=True)
```

---

## Hooks (`hooks.py`)

```python
# hooks.py

app_name = "my_app"
app_title = "My App"

# Document event hooks (fires on any app's DocType events)
doc_events = {
    "Sales Order": {
        "on_submit": "my_app.events.sales_order.on_submit",
        "on_cancel": "my_app.events.sales_order.on_cancel",
    },
    "*": {
        "after_insert": "my_app.events.all_docs.log_creation"
    }
}

# Scheduled tasks
scheduler_events = {
    "daily": ["my_app.tasks.daily_sync"],
    "hourly": ["my_app.tasks.check_overdue"],
    "cron": {
        "0 9 * * 1-5": ["my_app.tasks.morning_report"]  # weekdays at 9am
    }
}

# Override a standard class
override_doctype_class = {
    "Sales Invoice": "my_app.overrides.CustomSalesInvoice"
}

# Fixtures (auto-export/import these DocTypes with app)
fixtures = ["Custom Field", "Property Setter", {"dt": "Role", "filters": [["name", "like", "My %"]]}]
```

---

## Whitelisted API Methods

Server-side functions callable from the client via `frappe.call()`:

```python
# Python
import frappe

@frappe.whitelist()
def get_customer_orders(customer):
    """Returns open orders for a customer."""
    return frappe.get_all(
        "Sales Order",
        filters={"customer": customer, "status": "To Deliver and Bill"},
        fields=["name", "grand_total", "transaction_date"]
    )

@frappe.whitelist(allow_guest=True)  # No login required
def public_status():
    return {"status": "ok"}
```

```javascript
// JavaScript (Form Script or page)
frappe.call({
    method: "my_app.api.get_customer_orders",
    args: { customer: frm.doc.customer },
    callback: (r) => {
        if (r.message) console.log(r.message);
    }
});
```

---

## Form Scripts (JavaScript)

```javascript
// my_doctype.js
frappe.ui.form.on("My DocType", {
    // Fires when form loads
    onload(frm) {
        frm.set_query("linked_customer", () => ({
            filters: { disabled: 0 }
        }));
    },

    // Fires when form is refreshed (including after save)
    refresh(frm) {
        if (frm.doc.status === "Open") {
            frm.add_custom_button("Mark Closed", () => {
                frm.set_value("status", "Closed");
                frm.save();
            });
        }
    },

    // Fires when field value changes
    amount(frm) {
        frm.set_value("tax", frm.doc.amount * 0.1);
    },

    // Child table row added
    "items_add"(frm, cdt, cdn) {
        let row = locals[cdt][cdn];
        row.qty = 1;
        frm.refresh_field("items");
    }
});
```

---

## REST API

Frappe auto-exposes REST endpoints for all DocTypes:

```
GET    /api/resource/{DocType}           → list
POST   /api/resource/{DocType}           → create
GET    /api/resource/{DocType}/{name}    → read
PUT    /api/resource/{DocType}/{name}    → update
DELETE /api/resource/{DocType}/{name}    → delete

POST   /api/method/{dotted.python.path} → call whitelisted method
```

Authentication: API Key + API Secret via `Authorization: token <key>:<secret>` header.

---

## Bench CLI Reference

```bash
bench init frappe-bench --frappe-branch version-15
bench new-app my_app
bench new-site mysite.localhost --install-app my_app
bench --site mysite.localhost install-app my_app
bench --site mysite.localhost migrate          # apply schema changes after DocType edits
bench --site mysite.localhost console          # Python REPL with frappe context
bench start                                    # start dev server
bench build                                    # build JS/CSS assets
bench --site mysite.localhost set-config developer_mode 1
bench --site mysite.localhost enable-scheduler
bench --site mysite.localhost run-patch my_app.patches.v1_0.my_patch
```

---

## Permissions

Permissions are defined per role in the DocType JSON:
```json
{"role": "Sales User", "read": 1, "write": 1, "create": 1, "delete": 0, "submit": 0}
```

Check permissions in code:
```python
frappe.has_permission("Sales Order", "write", doc=doc, throw=True)
```

---

## Background Jobs

```python
# Enqueue a job (runs in Redis queue via RQ)
frappe.enqueue(
    "my_app.tasks.heavy_computation",
    queue="long",       # "default", "short", "long"
    timeout=600,
    job_id="unique-job-id",   # optional, prevents duplicates
    customer="CUST-0001"      # kwargs passed to the function
)

# The function itself
def heavy_computation(customer):
    # runs in worker process
    pass
```

---

## Common Patterns & Gotchas

- **Always call `doc.save()` or `doc.insert()` to persist** — setting attributes alone doesn't write to DB.
- **Use `frappe.db.commit()`** only when bypassing the ORM (e.g., after raw SQL). ORM auto-commits.
- **`frappe.throw()`** raises a user-visible exception; **`frappe.log_error()`** logs silently.
- **Child table rows** are accessed via `doc.items` (list of dicts/Documents). Add with `doc.append("items", {...})`.
- **`bench migrate`** must be run after any DocType JSON changes to update the DB schema.
- **Developer Mode** must be on to save DocType changes to JSON files on disk.
- **Fixtures** are the correct way to ship configuration (Custom Fields, etc.) with your app.
- **`frappe.get_doc` vs `frappe.get_all`**: `get_doc` returns a full Document object; `get_all` returns a list of dicts.
- Avoid raw `frappe.db.sql()` for CRUD — use the ORM or Query Builder for safety and permission checks.

---

## Reference Files

For deeper API details, read the relevant reference files:
- `references/python-api.md` — Document API, Database API, utility functions
- `references/js-api.md` — Form scripts, controls, dialog API, server calls
- `references/hooks.md` — Complete hooks reference

When building complex features, check the official docs:
https://docs.frappe.io/framework/user/en/introduction
