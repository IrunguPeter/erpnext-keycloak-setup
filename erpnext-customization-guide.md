# ERPNext Customization Guide for Kenya Government Operations

## Overview
This guide covers customizing ERPNext for:
- Ticketing System
- HR Management (Employees, Leave, Promotions)
- Integration with Kenya's HRIS-K (Human Resource Information System - Kenya)
- Accounting (Receivables, Payables, Budgeting)
- Procurement with EGP (Electronic Government Procurement) Integration

---

## Part 1: Ticketing System Configuration

### 1.1 Enable Built-in Issue Tracking

ERPNext includes a built-in Issue doctype for ticketing:

1. Go to **Setup** → **Modules** → **Support**
2. Enable **Issue** module
3. Configure Issue Types:
   - Go to **Support** → **Settings** → **Issue Type**
   - Add types: `HR Request`, `IT Support`, `Finance Query`, `Procurement`

### 1.2 Create Custom Ticket Doctypes

Go to **Setup** → **Customize** → **DocType**

Create doctypes:
- **Ticket** (custom doctype)
  - Subject (Data)
  - Description (Text Editor)
  - Priority (Select: Low, Medium, High, Urgent)
  - Status (Select: Open, In Progress, Resolved, Closed)
  - Assigned To (Link: User)
  - Department (Link: Department)

### 1.3 Ticket Workflow

Go to **Setup** → **Workflow** → **Workflow**:
1. Create workflow for Ticket
2. Add states: Draft → Open → In Progress → Resolved → Closed
3. Add transitions with conditions

---

## Part 2: HR Management Setup

### 2.1 Install HRMS App

```bash
bench get-app --branch version-16 hrms
bench --site your-site.com install-app hrms
```

### 2.2 Employee Configuration

Go to **HR** → **Employee**:

1. **Create Employee Master**
   - Employee Name, Date of Birth, Gender
   - Date of Joining, Department, Designation
   - Employment Type (Permanent, Contract, Casual)
   - **Kenya Specific Fields**:
     - National ID Number
     - KRA PIN (Kenya Revenue Authority)
     - NHIF Number (National Hospital Insurance Fund)
     - NSSF Number (National Social Security Fund)

2. **Custom Fields for Kenya**

Go to **Setup** → **Customize** → **Custom Field**:

Add to Employee doctype:
- `national_id` (Data, Label: National ID)
- `kra_pin` (Data, Label: KRA PIN)
- `nhif_number` (Data, Label: NHIF Number)
- `nssf_number` (Data, Label: NSSF Number)
- `upn_number` (Data, Label: Unified Payroll Number)

### 2.3 Leave Management

Go to **HR** → **Leave and Attendance**:

1. **Leave Types** (Kenya statutory leaves):
   - Annual Leave (21 days minimum)
   - Sick Leave (7 days with medical certificate)
   - Maternity Leave (90 days)
   - Paternity Leave (14 days)
   - Compassionate Leave (7 days)
   - Public Holidays

2. **Leave Policy**
   - Create leave policies per employment type
   - Configure accrual rules
   - Set carry-forward limits

3. **Leave Application Workflow**
   - Employee applies → Manager approves → HR processes

### 2.4 Promotions Module

Go to **HR** → **Promotion**:

1. Create **Promotion** doctype:
   - Employee (Link)
   - Current Designation (Data)
   - New Designation (Data)
   - Current Grade (Data)
   - New Grade (Data)
   - Effective Date (Date)
   - Promotion Type (Select: Regular, Acting, Upgrade)
   - Approval Status (Select)

2. **Promotion Workflow**
   - HR Initiate → Department Head Approve → Director Approve → Effective

---

## Part 3: Integration with Kenya HRIS-K

### 3.1 Understanding HRIS-K

The Human Resource Information System - Kenya (HRIS-K) is the government's centralized HR system:
- Website: https://uhr.kenya.go.ke/
- Replaces GHRIS and IPPD systems
- Manages payroll, leave, pensions for public service

### 3.2 API Integration Setup

Create a custom app for HRIS-K integration:

```bash
bench new-app hrms_kenya_integration
bench --site your-site.com install-app hrms_kenya_integration
```

### 3.3 Create Integration DocType

Go to **Setup** → **DocType** → Create:

**HRIS-K Configuration**:
- Base URL (Data): `https://uhr.kenya.go.ke/api`
- API Key (Data, Password)
- Organization Code (Data)
- Sync Frequency (Select: Hourly, Daily)

### 3.4 Sync Employee Data

Create Python script for syncing:

```python
# hrms_kenya_integration/api.py

import frappe
import requests

@frappe.whitelist()
def sync_employee_to_hris(employee_id):
    """Sync employee data to HRIS-K"""
    employee = frappe.get_doc("Employee", employee_id)
    
    payload = {
        "national_id": employee.custom_national_id,
        "kra_pin": employee.custom_kra_pin,
        "nhif_number": employee.custom_nhif_number,
        "nssf_number": employee.custom_nssf_number,
        "upn_number": employee.custom_upn_number,
        "employee_name": employee.employee_name,
        "department": employee.department,
        "designation": employee.designation,
        "date_of_joining": str(employee.date_of_joining)
    }
    
    config = frappe.get_single("HRIS-K Configuration")
    
    response = requests.post(
        f"{config.base_url}/employees/sync",
        json=payload,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()

@frappe.whitelist()
def sync_leave_to_hris(leave_id):
    """Sync leave application to HRIS-K"""
    leave = frappe.get_doc("Leave Application", leave_id)
    
    payload = {
        "employee_upn": frappe.db.get_value("Employee", leave.employee, "custom_upn_number"),
        "leave_type": leave.leave_type,
        "from_date": str(leave.from_date),
        "to_date": str(leave.to_date),
        "total_days": leave.total_leave_days,
        "reason": leave.reason
    }
    
    config = frappe.get_single("HRIS-K Configuration")
    
    response = requests.post(
        f"{config.base_url}/leave/sync",
        json=payload,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()
```

### 3.5 Configure Hooks

```python
# hrms_kenya_integration/hooks.py

doc_events = {
    "Employee": {
        "on_update": "hrms_kenya_integration.api.sync_employee_to_hris",
        "after_insert": "hrms_kenya_integration.api.sync_employee_to_hris"
    },
    "Leave Application": {
        "on_submit": "hrms_kenya_integration.api.sync_leave_to_hris"
    }
}
```

---

## Part 4: Accounting Configuration

### 4.1 Enable Accounting Module

Go to **Setup** → **Modules** → **Accounts**

### 4.2 Accounts Receivable

Go to **Accounts** → **Accounts Receivable**:

1. **Configure Receivable Types**
   - Trade Receivables
   - Other Receivables
   - Advance Receivables

2. **Customer Setup**
   - Create customer master
   - Set payment terms
   - Configure aging buckets

3. **Invoice Workflow**
   - Sales Invoice → Payment Entry → Bank Reconciliation

### 4.3 Accounts Payable

Go to **Accounts** → **Accounts Payable**:

1. **Configure Payable Types**
   - Trade Payables
   - Other Payables
   - Statutory Payables (KRA, NHIF, NSSF)

2. **Supplier Setup**
   - Create supplier master
   - Set payment terms
   - Configure approval workflow

3. **Bill Processing**
   - Purchase Invoice → Payment Entry → Bank Reconciliation

### 4.4 Budgeting

Go to **Accounts** → **Budget**:

1. **Budget Structure**
   - Create Cost Centers
   - Create Budget (per department/project)
   - Set budget periods (monthly/quarterly/annual)

2. **Budget Allocation**
   - Allocate by department
   - Allocate by project
   - Allocate by expense head

3. **Budget Control**
   - Enable budget checking on Journal Entry
   - Enable budget checking on Purchase Invoice
   - Set approval workflow for over-budget

### 4.5 Kenya Tax Configuration

Go to **Accounts** → **Tax**:

1. **VAT Setup**
   - VAT 16% (standard rate)
   - VAT 0% (exempt goods)
   - VAT Exemption Certificate

2. **Withholding Tax**
   - WHT on payments to suppliers
   - WHT certificates

3. **KRA Integration**
   - Configure iTax API
   - Auto-generate withholding tax certificates
   - Submit VAT returns

---

## Part 5: Procurement with EGP Integration

### 5.1 Understanding EGP Kenya

The Electronic Government Procurement (e-GP) system:
- Website: https://egpkenya.go.ke/
- Mandated by PPADA 2015 and PPADR 2020
- All government procurement must go through e-GP

### 5.2 Enable Procurement Module

Go to **Setup** → **Modules** → **Buying**

### 5.3 Procurement Workflow

1. **Purchase Requisition**
   - Employee creates request
   - Department Head approves
   - Procurement Officer processes

2. **Purchase Order**
   - Create PO from PR
   - Send to supplier
   - Track delivery

3. **Goods Receipt**
   - Receive goods
   - Quality check
   - Update inventory

### 5.4 EGP Integration Setup

Create custom app for EGP integration:

```bash
bench new-app erpnext_egp_integration
bench --site your-site.com install-app erpnext_egp_integration
```

### 5.5 EGP API DocType

Go to **Setup** → **DocType** → Create:

**EGP Configuration**:
- Base URL (Data): `https://egpkenya.go.ke/api`
- Entity Code (Data)
- API Key (Data, Password)
- Procuring Entity ID (Data)

### 5.6 EGP Integration Script

```python
# erpnext_egp_integration/api.py

import frappe
import requests
from frappe.utils import getdate

@frappe.whitelist()
def create_egp_tender(purchase_requisition):
    """Create tender on e-GP platform"""
    pr = frappe.get_doc("Purchase Requisition", purchase_requisition)
    
    config = frappe.get_single("EGP Configuration")
    
    # Prepare tender data
    tender_data = {
        "procuring_entity_id": config.procuring_entity_id,
        "tender_reference": pr.name,
        "description": pr.items[0].description,
        "estimated_value": pr.total,
        "currency": "KES",
        "procurement_method": determine_procurement_method(pr),
        "opening_date": str(pr.transaction_date),
        "closing_date": str(add_months(pr.transaction_date, 1)),
        "category": determine_procurement_category(pr)
    }
    
    response = requests.post(
        f"{config.base_url}/tenders/create",
        json=tender_data,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    if response.status_code == 200:
        tender_id = response.json().get("tender_id")
        frappe.db.set_value("Purchase Requisition", pr.name, "custom_egp_tender_id", tender_id)
        frappe.msgprint(f"Tender created on e-GP: {tender_id}")
    
    return response.json()

@frappe.whitelist()
def submit_bid_to_egp(purchase_order):
    """Submit bid to e-GP platform"""
    po = frappe.get_doc("Purchase Order", purchase_order)
    config = frappe.get_single("EGP Configuration")
    
    bid_data = {
        "tender_id": po.custom_egp_tender_id,
        "supplier_tin": frappe.db.get_value("Supplier", po.supplier, "tax_id"),
        "bid_amount": po.grand_total,
        "currency": "KES",
        "items": [{
            "item_code": item.item_code,
            "description": item.description,
            "quantity": item.qty,
            "unit_price": item.rate,
            "amount": item.amount
        } for item in po.items]
    }
    
    response = requests.post(
        f"{config.base_url}/bids/submit",
        json=bid_data,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()

@frappe.whitelist()
def sync_annual_procurement_plan(department):
    """Sync Annual Procurement Plan to e-GP"""
    config = frappe.get_single("EGP Configuration")
    
    # Get budget data
    budget = frappe.get_doc("Budget", {"department": department})
    
    plan_data = {
        "procuring_entity_id": config.procuring_entity_id,
        "financial_year": budget.fiscal_year,
        "department": department,
        "items": [{
            "item_description": item.description,
            "estimated_value": item.amount,
            "procurement_method": item.custom_procurement_method,
            "timeline": item.custom_timeline
        } for item in budget.items]
    }
    
    response = requests.post(
        f"{config.base_url}/annual-plan/sync",
        json=plan_data,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()

def determine_procurement_method(pr):
    """Determine procurement method based on value"""
    if pr.total <= 500000:
        return "Request for Quotation"
    elif pr.total <= 2000000:
        return "Request for Quotation"
    elif pr.total <= 10000000:
        return "Open Tender"
    else:
        return "National Open Tender"

def determine_procurement_category(pr):
    """Determine procurement category"""
    category_map = {
        "Goods": "Goods",
        "Works": "Works",
        "Services": "Services",
        "Consultancy": "Consultancy Services"
    }
    return category_map.get(pr.items[0].item_group, "Goods")

def add_months(date, months):
    """Add months to a date"""
    from dateutil.relativedelta import relativedelta
    return getdate(date) + relativedelta(months=months)
```

### 5.7 Custom Fields for EGP

Add to Purchase Requisition:
- `custom_egp_tender_id` (Data, Label: e-GP Tender ID)
- `custom_egp_status` (Select, Label: e-GP Status)
- `custom_procurement_method` (Select, Label: Procurement Method)

Add to Purchase Order:
- `custom_egp_tender_id` (Data)
- `custom_egp_bid_id` (Data)
- `custom_contract_id` (Data)

### 5.8 Hooks Configuration

```python
# erpnext_egp_integration/hooks.py

doc_events = {
    "Purchase Requisition": {
        "on_submit": "erpnext_egp_integration.api.create_egp_tender"
    },
    "Purchase Order": {
        "on_submit": "erpnext_egp_integration.api.submit_bid_to_egp"
    }
}
```

---

## Part 6: Custom Reports

### 6.1 HR Reports

Create custom reports:

1. **Employee Master Report**
   - All employees with Kenya-specific fields
   - Filter by department, designation, employment type

2. **Leave Balance Report**
   - Annual leave, sick leave balances
   - Carry-forward amounts

3. **Payroll Summary Report**
   - Gross salary, deductions, net pay
   - KRA tax calculations

### 6.2 Financial Reports

1. **Budget Variance Report**
   - Budget vs Actual
   - Variance analysis

2. **Accounts Receivable Aging**
   - Outstanding by customer
   - Aging buckets (30, 60, 90, 120+ days)

3. **Accounts Payable Aging**
   - Outstanding by supplier
   - Payment schedule

### 6.3 Procurement Reports

1. **Procurement Summary**
   - Total procurement value
   - By department, category

2. **EGP Tender Status**
   - Tenders on e-GP platform
   - Award status

---

## Part 7: Security & Permissions

### 7.1 Role Configuration

Create custom roles:
- **HR Manager**: Full HR access
- **Finance Manager**: Full accounting access
- **Procurement Officer**: Full procurement access
- **Department Head**: Approve requests for department
- **Employee**: Self-service access

### 7.2 Permission Rules

Go to **Setup** → **Roles and Permissions**:

1. **Employee**
   - HR Manager: Read, Write, Create, Delete
   - Department Head: Read, Write (department only)
   - Employee: Read (own record only)

2. **Leave Application**
   - Employee: Create, Read (own)
   - Department Head: Read, Write, Approve (department)
   - HR Manager: Full access

3. **Purchase Requisition**
   - Employee: Create, Read (own)
   - Department Head: Read, Write, Approve
   - Procurement Officer: Full access

---

## Part 8: Deployment Checklist

### Pre-Deployment
- [ ] Configure MariaDB for production
- [ ] Setup SSL certificates
- [ ] Configure backup strategy
- [ ] Setup monitoring

### Module Installation
- [ ] Install ERPNext
- [ ] Install HRMS
- [ ] Install custom apps (hrms_kenya_integration, erpnext_egp_integration)
- [ ] Configure modules

### Configuration
- [ ] Setup Users and Roles
- [ ] Configure Leave Types
- [ ] Configure Account Heads
- [ ] Configure Cost Centers
- [ ] Setup Tax Rules
- [ ] Configure EGP Integration
- [ ] Configure HRIS-K Integration

### Testing
- [ ] Test Employee creation
- [ ] Test Leave application workflow
- [ ] Test Purchase Requisition workflow
- [ ] Test accounting entries
- [ ] Test EGP integration
- [ ] Test HRIS-K sync

### Go-Live
- [ ] Import existing data
- [ ] Train users
- [ ] Monitor system
- [ ] Address issues

---

## Appendix: Useful Commands

```bash
# Backup site
bench --site your-site.com backup

# Restore backup
bench --site your-site.com restore /path/to/backup.sql.gz

# Clear cache
bench --site your-site.com clear-cache

# Run custom script
bench --site your-site.com execute hrms_kenya_integration.api.sync_all_employees

# Check site status
bench doctor

# Update ERPNext
bench update
```

---

## Support

For issues or questions:
- ERPNext: https://discuss.frappe.io/
- Kenya e-GP: https://support.egpkenya.go.ke/
- HRIS-K: hris.kenya@psyg.go.ke
