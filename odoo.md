# 🎓 Odoo Custom Addons — Complete Beginner's Guide

---

## 📁 Understanding Your Odoo Setup

```
/home/lt110/Odoo/
├── odoo/                        ← Core Odoo source code
│   ├── odoo.conf                ← Main configuration file
│   ├── addons/                  ← Built-in Odoo modules (don't touch these)
│   └── custom/
│       └── addons/              ← YOUR custom modules go here ✅
├── venv/                        ← Python virtual environment
├── .env                         ← Environment variables
├── start.sh                     ← Script to start Odoo
└── odoo.log                     ← Log file
```

Your `odoo.conf` already has both paths configured:
```ini
addons_path = /home/lt110/Odoo/odoo/addons,/home/lt110/Odoo/odoo/custom/addons
```
This tells Odoo: *"Look for modules in BOTH of these folders."*

---

## 🧩 What is an Odoo Module (Addon)?

A module is a **Python package** (a folder) that adds features to Odoo. Every module follows the **exact same structure**:

```
my_module/                        ← Module folder (name = technical name)
├── __init__.py                   ← Makes it a Python package (REQUIRED)
├── __manifest__.py               ← Module metadata & configuration (REQUIRED)
├── models/                       ← Python files: define database tables & logic
│   ├── __init__.py
│   └── my_model.py
├── views/                        ← XML files: define UI (forms, lists, menus)
│   └── my_views.xml
├── security/                     ← Access control rules
│   ├── ir.model.access.csv       ← Who can read/write/create/delete
│   └── my_security.xml
├── static/                       ← Frontend assets (JS, CSS, images)
│   └── src/
│       ├── js/
│       ├── css/
│       └── xml/
├── controllers/                  ← HTTP route handlers (REST APIs, web pages)
│   ├── __init__.py
│   └── main.py
└── data/                         ← Default/demo data (XML files)
    └── my_data.xml
```

---

## 🔑 The Two Most Important Files

### 1. `__manifest__.py` — The Module's ID Card

```python
{
    # ── Basic Info ──────────────────────────────────────
    'name': 'PeopleConnect Employee',      # Display name in Odoo UI
    'version': '1.1',                      # Your version
    'category': 'Human Resources',         # Groups it in the app list
    'summary': 'Extended version of Employee module',
    'author': 'PearlSoft technologies LLP',

    # ── Dependencies ────────────────────────────────────
    # Modules that MUST be installed BEFORE this one
    'depends': ['base', 'hr', 'hr_contract', 'web'],

    # ── Data Files ──────────────────────────────────────
    # XML/CSV files loaded IN ORDER when the module installs
    'data': [
        'security/ir.model.access.csv',    # ← Load security FIRST
        'views/hr_employee_views.xml',     # ← Then views
    ],

    # ── Frontend Assets ─────────────────────────────────
    'assets': {
        'web.assets_backend': [
            'my_module/static/css/style.css',
            'my_module/static/src/js/script.js',
        ]
    },

    # ── Flags ───────────────────────────────────────────
    'installable': True,      # Can it be installed? Always True
    'application': False,     # True = shows as big App icon; False = just a module
    'auto_install': False,    # True = installs automatically with dependencies
    'license': 'LGPL-3',
}
```

### 2. `__init__.py` — Importing Python Files

```python
# /module/__init__.py  — imports the models subpackage
from . import models

# /module/models/__init__.py — imports each model file
from . import hr_employee
from . import res_config_settings
```

> **Rule:** Every folder that has Python files needs an `__init__.py` that imports them.

---

## 🏗️ Understanding Models (Database Tables)

```python
from odoo import models, fields, api, _

class HrEmployeePublic(models.Model):
    _inherit = 'hr.employee.public'   # ← EXTEND an existing model (no new table)
    # OR: _name = 'my.model'         # ← CREATE a new model (creates new DB table)

    # Fields = columns in the database
    communication_id = fields.Char(string='Communication ID')
    joining_date = fields.Date(string='Joining Date')
    can_edit = fields.Boolean(string='Can Edit', compute='_compute_can_edit')

    # Computed field — calculated dynamically, not stored in DB
    @api.depends_context('uid')
    def _compute_can_edit(self):
        for record in self:
            record.can_edit = self.env.user.has_group('hr.group_hr_user')
```

**Two ways to use models:**

| `_name = 'my.model'` | `_inherit = 'hr.employee'` |
|---|---|
| Creates a **brand new** database table | **Extends** an existing model |
| For completely new features | To add fields/methods to existing Odoo models |

**Common Field Types:**
```python
fields.Char(string='Name')                          # Short text
fields.Text(string='Description')                   # Long text
fields.Integer(string='Count')                      # Whole number
fields.Float(string='Price')                        # Decimal number
fields.Boolean(string='Active')                     # True/False
fields.Date(string='Date')                          # Date only
fields.Datetime(string='Created')                   # Date + time
fields.Many2one('res.partner', string='Partner')    # Foreign key (link to another record)
fields.One2many('sale.order.line', 'order_id', ...) # Reverse of Many2one
fields.Many2many('res.users', string='Users')       # Many-to-many relationship
fields.Selection([('a','A'),('b','B')], ...)        # Dropdown
```

---

## 🖥️ Understanding Views (UI/XML)

Views tell Odoo how to display your data:

```xml
<!-- Form View = detail view of one record -->
<record id="view_employee_form" model="ir.ui.view">
    <field name="name">hr.employee.form</field>
    <field name="model">hr.employee</field>
    <field name="arch" type="xml">
        <form string="Employee">
            <sheet>
                <field name="name"/>
                <field name="communication_id"/>
                <field name="joining_date"/>
            </sheet>
        </form>
    </field>
</record>

<!-- List/Tree View = table of multiple records -->
<record id="view_employee_list" model="ir.ui.view">
    <field name="name">hr.employee.list</field>
    <field name="model">hr.employee</field>
    <field name="arch" type="xml">
        <tree>
            <field name="name"/>
            <field name="joining_date"/>
        </tree>
    </field>
</record>

<!-- Action = what happens when a menu is clicked -->
<record id="action_employees" model="ir.actions.act_window">
    <field name="name">Employees</field>
    <field name="res_model">hr.employee</field>
    <field name="view_mode">list,form</field>
</record>

<!-- Menu item in the sidebar/navbar -->
<menuitem id="menu_employees"
    name="Employees"
    action="action_employees"
    parent="menu_hr_root"/>
```

---

## 🔐 Security (Access Control)

The file `security/ir.model.access.csv` controls **who can do what**:

```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_hr_employee,hr.employee all,model_hr_employee,base.group_user,1,0,0,0
access_hr_employee_manager,hr.employee manager,model_hr_employee,hr.group_hr_manager,1,1,1,1
```

| Column | Meaning |
|---|---|
| `model_id:id` | Which model? format: `model_` + model name with dots replaced by underscores |
| `group_id:id` | Which user group? (empty = everyone) |
| `perm_read` | Can read? 1=yes, 0=no |
| `perm_write` | Can edit? |
| `perm_create` | Can create new records? |
| `perm_unlink` | Can delete? |

---

## 🚀 How to Install a New Module — Step by Step

### Method 1: Install via Odoo Web UI

**Step 1:** Copy the module folder into your custom addons directory:
```bash
cp -r /path/to/new_module /home/lt110/Odoo/odoo/custom/addons/
```

**Step 2:** Restart Odoo (or activate developer mode first):
```bash
cd /home/lt110/Odoo
source venv/bin/activate
python odoo/odoo-bin -c odoo/odoo.conf
```

**Step 3:** In the Odoo Web UI:
- Go to **Settings → Activate developer mode** (bottom of Settings page, or add `?debug=1` to URL)
- Go to **Apps** menu
- Click **Update Apps List** button (only visible in developer mode!)
- Search for your module name
- Click **Install**

### Method 2: Install via Command Line (Faster!)

```bash
cd /home/lt110/Odoo
source venv/bin/activate

# Install a module:
python odoo/odoo-bin \
    -c odoo/odoo.conf \
    -d odoo18 \
    --install-modules=my_module_name

# Update/upgrade an already-installed module:
python odoo/odoo-bin \
    -c odoo/odoo.conf \
    -d odoo18 \
    --update=my_module_name
```

### Method 3: Install via Odoo Shell
```bash
cd /home/lt110/Odoo
source venv/bin/activate
python odoo/odoo-bin shell -c odoo/odoo.conf -d odoo18

# In the shell:
>>> env['ir.module.module'].search([('name','=','my_module')]).button_immediate_install()
>>> exit()
```

---

## 🏭 Creating Your First Module From Scratch

### Minimum required files:

**`__manifest__.py`:**
```python
{
    'name': 'My First Module',
    'version': '1.0',
    'category': 'Custom',
    'summary': 'My first Odoo module',
    'depends': ['base'],
    'data': [
        'security/ir.model.access.csv',
        'views/my_views.xml',
    ],
    'installable': True,
    'application': True,
    'auto_install': False,
    'license': 'LGPL-3',
}
```

**`__init__.py`:**
```python
from . import models
```

**`models/__init__.py`:**
```python
from . import my_model
```

**`models/my_model.py`:**
```python
from odoo import models, fields

class MyTask(models.Model):
    _name = 'my.task'              # Creates table 'my_task' in database
    _description = 'My Task'

    name = fields.Char(string='Task Name', required=True)
    description = fields.Text(string='Description')
    done = fields.Boolean(string='Done', default=False)
    priority = fields.Selection([
        ('low', 'Low'),
        ('medium', 'Medium'),
        ('high', 'High'),
    ], string='Priority', default='medium')
```

**`security/ir.model.access.csv`:**
```csv
id,name,model_id:id,group_id:id,perm_read,perm_write,perm_create,perm_unlink
access_my_task,my.task,model_my_task,,1,1,1,1
```

**`views/my_views.xml`:**
```xml
<?xml version="1.0" encoding="utf-8"?>
<odoo>
    <!-- Form View -->
    <record id="view_my_task_form" model="ir.ui.view">
        <field name="name">my.task.form</field>
        <field name="model">my.task</field>
        <field name="arch" type="xml">
            <form string="Task">
                <sheet>
                    <field name="name"/>
                    <field name="priority"/>
                    <field name="done"/>
                    <field name="description"/>
                </sheet>
            </form>
        </field>
    </record>

    <!-- List View -->
    <record id="view_my_task_list" model="ir.ui.view">
        <field name="name">my.task.list</field>
        <field name="model">my.task</field>
        <field name="arch" type="xml">
            <tree>
                <field name="name"/>
                <field name="priority"/>
                <field name="done"/>
            </tree>
        </field>
    </record>

    <!-- Action -->
    <record id="action_my_task" model="ir.actions.act_window">
        <field name="name">My Tasks</field>
        <field name="res_model">my.task</field>
        <field name="view_mode">list,form</field>
    </record>

    <!-- Menu -->
    <menuitem id="menu_my_module_root"
        name="My Module"
        sequence="100"/>
    <menuitem id="menu_my_tasks"
        name="Tasks"
        parent="menu_my_module_root"
        action="action_my_task"/>
</odoo>
```

---

## 🔄 The Module Lifecycle

```
       Create files
           ↓
    Copy to addons folder
           ↓
      Restart Odoo
           ↓
   Update Apps List (UI)
           ↓
      Find & Install
           ↓
    Odoo creates DB tables
    and loads all XML files
           ↓
      Module is active! 🎉
           ↓
    Make changes to code?
           ↓
    Upgrade module (not reinstall)
    python odoo-bin -u module_name
```

---

## ⚙️ Your `odoo.conf` Explained

```ini
[options]
admin_passwd = admin          # Master password for DB management UI
db_host = localhost           # PostgreSQL server
db_port = 5432                # PostgreSQL port
db_user = lt110               # DB username
db_password = lt110           # DB password
db_name = odoo18              # Database to use

http_port = 8069              # Odoo URL port → http://localhost:8069
longpolling_port = 8072       # For real-time chat/notifications

workers = 0                   # 0 = single threaded (good for development)

# Odoo searches for modules in these paths:
addons_path = /home/lt110/Odoo/odoo/addons,          # built-in modules
             /home/lt110/Odoo/odoo/custom/addons      # your custom modules

logfile = /var/log/odoo/odoo.log   # Where logs are saved
```

---

## 🔍 Your Existing Modules — Quick Reference

| Module | Purpose |
|---|---|
| `zyrahrm_theme` | Custom UI theme with TailwindCSS |
| `peopleconnect_employee` | Extended employee model with extra fields |
| `peopleconnect_attendance` | Attendance management |
| `peopleconnect_leave` | Leave management |
| `peopleconnect_payslip` | Payslip customizations |
| `peopleconnect_recruitment` | Recruitment workflow |
| `peopleconnect_web` | Web/HTTP controllers (REST API endpoints) |
| `peopleconnect_dashboard` | Custom dashboard |
| `accounting_pdf_reports` | PDF report generation |
| `xf_payroll_system` | Full payroll system |

---

## 🐛 Common Mistakes & How to Fix Them

| Problem | Cause | Fix |
|---|---|---|
| Module not showing in Apps list | Forgot to Update Apps List | Apps → Update Apps List |
| `ImportError` on startup | Missing `from . import model_name` in `__init__.py` | Add the import |
| `KeyError: 'ir.model.access'` | Missing or wrong security CSV | Check `security/ir.model.access.csv` |
| Changes not reflected | Didn't upgrade module | Run with `-u module_name` |
| `psycopg2` database error | Field added but DB not updated | Upgrade module to sync DB schema |
| Module installs but menu missing | Missing `<menuitem>` in XML or forgot to add view file to `data:` in manifest | Check manifest `data:` list |

---

## 💡 Key Concepts Summary

| Concept | What it is |
|---|---|
| **Module/Addon** | A folder with `__manifest__.py` that adds features |
| **Model** | A Python class = a database table |
| **`_name`** | Creates a NEW table |
| **`_inherit`** | EXTENDS an existing table/model |
| **View** | XML file that defines how data looks in the browser |
| **Manifest** | The module's "passport" — name, dependencies, files |
| **`depends`** | Modules that must be installed first |
| **`data`** | XML/CSV files loaded on install |
| **`assets`** | JS/CSS files for the frontend |
| **`installable: True`** | Module can be installed from the UI |
