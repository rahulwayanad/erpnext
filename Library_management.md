# Install and Setup Bench

Bench is the command line tool to manage Frappe apps and sites.

## Installation

If you haven't installed Bench, follow the Installation guide. After installation, you should be able to run commands that start with `bench`.

Run the following command to test your installation:

```bash
$ bench --version
```

## Create the frappe-bench directory

Let's create our project folder which will contain our apps and sites. Run the following command:

```bash
$ bench init frappe-bench
```

This will create a directory named `frappe-bench` in your current working directory. It will do the following:

- Create a python virtual environment under `env` directory.
- Fetch and install the `frappe` app as a python package.
- Install node modules of frappe.
- Build static assets.

### Directory Structure

```
.
├── Procfile
├── apps
│   └── frappe
├── config
│   ├── pids
│   ├── redis_cache.conf
│   ├── redis_queue.conf
│   └── redis_socketio.conf
├── env
│   ├── bin
│   ├── include
│   ├── lib
│   └── share
├── logs
│   ├── backup.log
│   └── bench.log
└── sites
    ├── apps.txt
    ├── assets
    └── common_site_config.json
```

### Start the Bench Server

Now that we have created our `frappe-bench` directory, we can start the Frappe web server by running the following command:

```bash
$ cd frappe-bench
$ bench start
```

---

## Create App for Library Management

To create our Library Management app, run the `new-app` command:

```bash
$ cd frappe-bench
$ bench new-app library_management
```

## Create a new site

To create a new site, run the following command from the `frappe-bench` directory:

```bash
$ bench new-site library.test
```

This will map `library.test` to localhost. Bench has a convenient command to do just that:

```bash
$ bench --site library.test add-to-hosts
```

## Install app on site

```bash
$ bench --site library.test install-app library_management
```

To confirm if the app was installed, run the following command:

```bash
$ bench --site library.test list-apps
frappe
library_management
```

## Login to Desk

To create DocTypes in our app, we must log in to Desk. Go to [http://library.test:8000](http://library.test:8000) and it should show you a login page.

Enter `Administrator` as the user and password that you set while creating the site.

---

## Create a DocType

DocType is analogous to a Model in other frameworks. Apart from defining properties, it also defines the behavior of the Model.

### Enable Developer Mode

Before we can create DocTypes, we need to enable developer mode on our bench. This will enable boilerplate creation when we create doctypes and we can track them into version control with our app.

Go to your terminal and quit the bench server if it's already running then from the `frappe-bench` directory, run the following command:

```bash
$ bench set-config -g developer_mode true
$ bench start
```

### Creating a DocType for Article

While in Desk, navigate to the **DocType List** using the Awesomebar. This list will include DocTypes bundled with the framework, those that are a part of the installed Frappe apps and custom ones, which you can create specific to each site.

The first doctype we will create is **Article**. To create it, click on **New**.

- Enter **Name** as `Article`
- Select `Library Management` in **Module**
- Add the following fields in the **Fields** table:

| Field | Type | Notes |
|---|---|---|
| Article Name | Data | Mandatory |
| Image | Attach Image | |
| Author | Data | |
| Description | Text Editor | |
| ISBN | Data | |
| Status | Select | Options: `Issued` and `Available` |
| Publisher | Data | |

After adding the fields, click on **Save**.

You will see a **Go to Article List** button at the top right of the form. Click on it to go to the Article List. Here you will see a blank list with no records because the table has no records.

---

## Controller Methods

Controller methods allow you to write business logic during the lifecycle of a document.

Let's create our second doctype: **Library Member**. It will have the following fields:

| Field | Type | Notes |
|---|---|---|
| First Name | Data | Mandatory |
| Last Name | Data | |
| Full Name | Data | Read Only |
| Email Address | Data | |
| Phone | Data | |

After you have created the doctype, go to **Library Member** list, clear the cache from **Settings > Reload** and create a new Library Member.

If you notice, the **Full Name** field is not shown in the form. This is because we set it as Read Only. It will be shown only when it has some value.

Let's write code in our python controller class such that Full Name is computed automatically from First Name and Last Name.

Open your code editor and open the file `library_member.py` and make the following changes:

**`library_member.py`**

```python
class LibraryMember(Document):
    # this method will run every time a document is saved
    def before_save(self):
        self.full_name = f'{self.first_name} {self.last_name or ""}'
```

> **Note:** If the above snippet doesn't work for you, make sure server side scripts are enabled, and then restart bench:
> ```bash
> $ bench --site set-config server_script_enabled true
> ```

---

## Types of DocType

Let's learn about the different types of doctype in the framework by creating more doctypes.

### Library Membership

Let's create another doctype: **Library Membership**. It will have the following fields:

| Field | Type | Notes |
|---|---|---|
| Library Member | Link | Mandatory |
| Full Name | Data | Read Only |
| From Date | Date | |
| To Date | Date | |
| Paid | Check | |

It will have **Is Submittable** enabled. It will have **Naming** set as `LMS.#####` and restricted to **Librarian** role. Also, the **Title Field** should be set to `full_name` in the View Settings section.

The **Link** field `Library Member` is similar to a Foreign Key column in other frameworks. It will let you link the value to a record in another DocType. In this case, it links to a record of **Library Member** DocType.

The **Full Name** field is a Read Only field that will be automatically fetched from the `full_name` field in the linked record **Library Member**.

#### Submittable DocTypes

When you enable **Is Submittable** in a DocType it becomes a Submittable DocType. A Submittable doctype can have 3 states: **Draft**, **Submitted** and **Cancelled**.

#### Controller Validation for Membership

Now, let's write code that will make sure whenever a Library Membership is created, there is no active membership for the Member.

**`library_membership.py`**

```python
import frappe
from frappe.model.document import Document
from frappe.model.docstatus import DocStatus


class LibraryMembership(Document):
    # check before submitting this document
    def before_submit(self):
        exists = frappe.db.exists(
            "Library Membership",
            {
                "library_member": self.library_member,
                "docstatus": DocStatus.submitted(),
                # check if the membership's end date is later than this membership's start date
                "to_date": (">", self.from_date),
            },
        )
        if exists:
            frappe.throw("There is an active membership for this member")
```

### Library Transaction

Let's create a DocType to record an Issue or Return of an Article by a Library Member who has an active membership.

This doctype will be called **Library Transaction** and will have the following fields:

| Field | Type | Notes |
|---|---|---|
| Article | Link | Link to Article |
| Library Member | Link | Link to Library Member |
| Type | Select | Options: `Issue` and `Return` |
| Date | Date | Date of Transaction |

This doctype will also be a **Submittable** doctype.

#### Validation for Transaction

When an Article is issued, we should verify whether the Library Member has an active membership. We should also check whether the Article is available for Issue. Let's write the code for these validations.

**`library_transaction.py`**

```python
import frappe
from frappe.model.document import Document
from frappe.model.docstatus import DocStatus


class LibraryTransaction(Document):
    def before_submit(self):
        if self.type == "Issue":
            self.validate_issue()
            # set the article status to be Issued
            article = frappe.get_doc("Article", self.article)
            article.status = "Issued"
            article.save()

        elif self.type == "Return":
            self.validate_return()
            # set the article status to be Available
            article = frappe.get_doc("Article", self.article)
            article.status = "Available"
            article.save()

    def validate_issue(self):
        self.validate_membership()
        article = frappe.get_doc("Article", self.article)
        # article cannot be issued if it is already issued
        if article.status == "Issued":
            frappe.throw("Article is already issued by another member")

    def validate_return(self):
        article = frappe.get_doc("Article", self.article)
        # article cannot be returned if it is not issued first
        if article.status == "Available":
            frappe.throw("Article cannot be returned without being issued first")

    def validate_membership(self):
        # check if a valid membership exist for this library member
        valid_membership = frappe.db.exists(
            "Library Membership",
            {
                "library_member": self.library_member,
                "docstatus": DocStatus.submitted(),
                "from_date": ("<", self.date),
                "to_date": (">", self.date),
            },
        )
        if not valid_membership:
            frappe.throw("The member does not have a valid membership")
```

> There is a lot of code here but it should be self explanatory. There are inline code comments for more explanation.

### Library Settings

Let's create the last doctype for our app: **Library Settings**. It will have the following fields:

| Field | Type | Notes |
|---|---|---|
| Loan Period | Int | Will define the loan period in number of days |
| Maximum Number of Issued Articles | Int | Restrict the maximum number of articles that can be issued by a single member |

Since we don't need to have multiple records for these settings, we will enable **Is Single** for this doctype.

After creating the doctype, click on **Go to Library Settings**, to go to the form and set the values for **Loan Period** and **Maximum Number of Issued Articles**.

#### Single DocTypes

When a DocType has **Is Single** enabled, it will become a **Single DocType**. A single doctype is similar to singleton records in other frameworks. It does not create a new database table. Instead all single values are stored in a single table called `tabSingles`. It is used usually for storing global settings.

#### Validation for Library Settings

Let's make the change in Library Membership such that, the **To Date** is automatically computed based on the **Loan Period** and the **From Date**.

**`library_membership.py`**

```python
import frappe
from frappe.model.document import Document
from frappe.model.docstatus import DocStatus


class LibraryMembership(Document):
    # check before submitting this document
    def before_submit(self):
        exists = frappe.db.exists(
            "Library Membership",
            {
                "library_member": self.library_member,
                "docstatus": DocStatus.submitted(),
                # check if the membership's end date is later than this membership's start date
                "to_date": (">", self.from_date),
            },
        )
        if exists:
            frappe.throw("There is an active membership for this member")

        # get loan period and compute to_date by adding loan_period to from_date
        loan_period = frappe.db.get_single_value("Library Settings", "loan_period")
        self.to_date = frappe.utils.add_days(self.from_date, loan_period or 30)
```

We have used `frappe.db.get_single_value` method to get the value of `loan_period` from the **Library Settings** doctype.

Now, let's make the change in Library Transaction such that when an Article is Issued, it checks whether the maximum limit is reached.

**`library_transaction.py`**

```python
import frappe
from frappe.model.document import Document
from frappe.model.docstatus import DocStatus


class LibraryTransaction(Document):
    def before_submit(self):
        if self.type == "Issue":
            self.validate_issue()
            self.validate_maximum_limit()
            # set the article status to be Issued
            article = frappe.get_doc("Article", self.article)
            article.status = "Issued"
            article.save()

        elif self.type == "Return":
            self.validate_return()
            # set the article status to be Available
            article = frappe.get_doc("Article", self.article)
            article.status = "Available"
            article.save()

    def validate_issue(self):
        self.validate_membership()
        article = frappe.get_doc("Article", self.article)
        # article cannot be issued if it is already issued
        if article.status == "Issued":
            frappe.throw("Article is already issued by another member")

    def validate_return(self):
        article = frappe.get_doc("Article", self.article)
        # article cannot be returned if it is not issued first
        if article.status == "Available":
            frappe.throw("Article cannot be returned without being issued first")

    def validate_maximum_limit(self):
        max_articles = frappe.db.get_single_value("Library Settings", "max_articles")
        count = frappe.db.count(
            "Library Transaction",
            {"library_member": self.library_member, "type": "Issue", "docstatus": DocStatus.submitted()},
        )
        if count >= max_articles:
            frappe.throw("Maximum limit reached for issuing articles")

    def validate_membership(self):
        # check if a valid membership exist for this library member
        valid_membership = frappe.db.exists(
            "Library Membership",
            {
                "library_member": self.library_member,
                "docstatus": DocStatus.submitted(),
                "from_date": ("<", self.date),
                "to_date": (">", self.date),
            },
        )
        if not valid_membership:
            frappe.throw("The member does not have a valid membership")
```
