# ERPNext Fleet Management Guide for Government Vehicles

## Overview
This guide covers configuring ERPNext for government fleet management, aligned with Kenya's Government Vehicle Management System (GVMS) requirements.

---

## Part 1: Fleet Management Module Setup

### 1.1 Enable Fleet Management App

ERPNext has a built-in Fleet Management module. Install it:

```bash
bench get-app fleet_management
bench --site your-site.com install-app fleet_management
```

### 1.2 Alternative: Custom Fleet Module

If using custom module, create doctypes:

**Vehicle** doctype:
- Vehicle Name (Data)
- Registration Number (Data)
- Make (Data)
- Model (Data)
- Year (Int)
- Color (Data)
- Engine Number (Data)
- Chassis Number (Data)
- Fuel Type (Select: Petrol, Diesel, Electric, Hybrid)
- Seating Capacity (Int)
- Department (Link: Department)
- Assigned To (Link: Employee)
- Status (Select: Available, Assigned, Under Maintenance, Retired)
- Insurance Expiry (Date)
- Tracking Device ID (Data)

**Vehicle Assignment** doctype:
- Vehicle (Link)
- Assigned To (Link: Employee)
- Start Date (Date)
- End Date (Date)
- Purpose (Text)
- Approved By (Link: User)

**Fuel Log** doctype:
- Vehicle (Link)
- Date (Date)
- Odometer Reading (Float)
- Fuel Quantity (Float)
- Fuel Cost (Currency)
- Station (Data)
- Receipt Number (Data)

**Maintenance Record** doctype:
- Vehicle (Link)
- Maintenance Type (Select: Service, Repair, Inspection)
- Date (Date)
- Description (Text)
- Cost (Currency)
- Vendor (Link: Supplier)
- Next Service Date (Date)
- Next Service Odometer (Float)

**Driver** doctype:
- Employee (Link)
- License Number (Data)
- License Expiry (Date)
- License Class (Data)
- Assigned Vehicles (Table: Vehicle Assignment)

---

## Part 2: Vehicle Master Configuration

### 2.1 Vehicle Categories (Kenya Government)

| Category | Description | Engine Capacity |
|----------|-------------|-----------------|
| Executive | CS, PS vehicles | 3000cc+ |
| Official | Senior officers | 2500-3000cc |
| General | Pool vehicles | 1800-2500cc |
| Utility | Pickup trucks | 2500cc+ |
| Security | Police, military | Varies |
| Ambulance | Medical services | Varies |
| Specialized | Construction, etc. | Varies |

### 2.2 Vehicle Allocation (Per Government Policy)

Per Government Transport Policy 2024:

| Official | Allocation |
|----------|------------|
| Cabinet Secretary | 2 vehicles |
| Principal Secretary | 1 vehicle |
| Head of Parastatal | 1 vehicle |
| Chairperson State Corp | 1 vehicle |
| Senior Officers | Pool vehicle |
| Commissioners | Private (reimbursement) |

### 2.3 Create Vehicle Records

Go to **Fleet** → **Vehicle** → **Add New**:

```
Vehicle Name: GRN 123A
Registration Number: GK A123B
Make: Toyota
Model: Land Cruiser V8
Year: 2024
Color: White
Engine Number: XXXXX
Chassis Number: XXXXX
Fuel Type: Diesel
Seating Capacity: 7
Department: Office of the President
Status: Assigned
Insurance Expiry: 2027-08-07
Tracking Device ID: TK-12345
```

---

## Part 3: GVMS Integration

### 3.1 Understanding GVMS

The Government Vehicle Management System (GVMS):
- Centralized digital platform for real-time monitoring
- Tracks authorized drivers, fuel consumption, mileage, maintenance
- Mandatory GPS tracking for all government vehicles
- Integrated with other government systems

### 3.2 Create GVMS Integration App

```bash
bench new-app erpnext_gvms_integration
bench --site your-site.com install-app erpnext_gvms_integration
```

### 3.3 GVMS DocType

Create **GVMS Configuration** doctype:
- Base URL (Data): GVMS API endpoint
- API Key (Data, Password)
- Organization Code (Data)
- Sync Frequency (Select: Real-time, Hourly, Daily)

### 3.4 GVMS Integration Script

```python
# erpnext_gvms_integration/api.py

import frappe
import requests
from frappe.utils import now_datetime, getdate

@frappe.whitelist()
def sync_vehicle_to_gvms(vehicle_name):
    """Sync vehicle data to GVMS"""
    vehicle = frappe.get_doc("Vehicle", vehicle_name)
    
    config = frappe.get_single("GVMS Configuration")
    
    payload = {
        "registration_number": vehicle.registration_number,
        "make": vehicle.make,
        "model": vehicle.model,
        "year": vehicle.year,
        "fuel_type": vehicle.fuel_type,
        "department": vehicle.department,
        "assigned_to": vehicle.assigned_to,
        "engine_number": vehicle.engine_number,
        "chassis_number": vehicle.chassis_number,
        "tracking_device_id": vehicle.tracking_device_id,
        "status": vehicle.status
    }
    
    response = requests.post(
        f"{config.base_url}/vehicles/sync",
        json=payload,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()

@frappe.whitelist()
def sync_fuel_log_to_gvms(fuel_log_name):
    """Sync fuel consumption to GVMS"""
    fuel_log = frappe.get_doc("Fuel Log", fuel_log_name)
    
    config = frappe.get_single("GVMS Configuration")
    
    payload = {
        "vehicle_registration": fuel_log.vehicle,
        "date": str(fuel_log.date),
        "odometer_reading": fuel_log.odometer_reading,
        "fuel_quantity": fuel_log.fuel_quantity,
        "fuel_cost": fuel_log.fuel_cost,
        "station": fuel_log.station,
        "receipt_number": fuel_log.receipt_number,
        "timestamp": str(now_datetime())
    }
    
    response = requests.post(
        f"{config.base_url}/fuel/log",
        json=payload,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()

@frappe.whitelist()
def sync_vehicle_location(vehicle_name, latitude, longitude):
    """Sync vehicle location from GPS tracker"""
    vehicle = frappe.get_doc("Vehicle", vehicle_name)
    
    config = frappe.get_single("GVMS Configuration")
    
    payload = {
        "registration_number": vehicle.registration_number,
        "latitude": latitude,
        "longitude": longitude,
        "timestamp": str(now_datetime()),
        "speed": 0,  # From GPS if available
        "heading": 0  # From GPS if available
    }
    
    response = requests.post(
        f"{config.base_url}/tracking/location",
        json=payload,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()

@frappe.whitelist()
def fetch_vehicle_from_gvms(registration_number):
    """Fetch vehicle data from GVMS"""
    config = frappe.get_single("GVMS Configuration")
    
    response = requests.get(
        f"{config.base_url}/vehicles/{registration_number}",
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()

@frappe.whitelist()
def sync_maintenance_to_gvms(maintenance_name):
    """Sync maintenance record to GVMS"""
    maintenance = frappe.get_doc("Maintenance Record", maintenance_name)
    
    config = frappe.get_single("GVMS Configuration")
    
    payload = {
        "vehicle_registration": maintenance.vehicle,
        "maintenance_type": maintenance.maintenance_type,
        "date": str(maintenance.date),
        "description": maintenance.description,
        "cost": maintenance.cost,
        "vendor": maintenance.vendor,
        "next_service_date": str(maintenance.next_service_date),
        "next_service_odometer": maintenance.next_service_odometer
    }
    
    response = requests.post(
        f"{config.base_url}/maintenance/sync",
        json=payload,
        headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
    )
    
    return response.json()
```

### 3.5 GVMS Hooks

```python
# erpnext_gvms_integration/hooks.py

doc_events = {
    "Vehicle": {
        "on_update": "erpnext_gvms_integration.api.sync_vehicle_to_gvms",
        "after_insert": "erpnext_gvms_integration.api.sync_vehicle_to_gvms"
    },
    "Fuel Log": {
        "on_submit": "erpnext_gvms_integration.api.sync_fuel_log_to_gvms"
    },
    "Maintenance Record": {
        "on_submit": "erpnext_gvms_integration.api.sync_maintenance_to_gvms"
    }
}

# Scheduled tasks for GPS tracking
scheduler_events = {
    "hourly": [
        "erpnext_gvms_integration.tasks.sync_all_vehicle_locations"
    ]
}
```

### 3.6 GPS Tracker Integration

Create scheduler task for GPS sync:

```python
# erpnext_gvms_integration/tasks.py

import frappe
import requests

def sync_all_vehicle_locations():
    """Fetch and sync GPS locations for all vehicles"""
    vehicles = frappe.get_all("Vehicle", 
        filters={"status": ["!=", "Retired"], "tracking_device_id": ["is", "set"]},
        fields=["name", "registration_number", "tracking_device_id"]
    )
    
    config = frappe.get_single("GVMS Configuration")
    
    for vehicle in vehicles:
        try:
            # Fetch location from GPS provider
            response = requests.get(
                f"{config.base_url}/tracking/{vehicle.tracking_device_id}",
                headers={"Authorization": f"Bearer {config.get_password('api_key')}"}
            )
            
            if response.status_code == 200:
                location_data = response.json()
                
                # Update vehicle location
                frappe.db.set_value("Vehicle", vehicle.name, {
                    "custom_latitude": location_data.get("latitude"),
                    "custom_longitude": location_data.get("longitude"),
                    "custom_last_tracking_update": frappe.utils.now_datetime()
                })
                
        except Exception as e:
            frappe.log_error(f"GPS sync failed for {vehicle.name}: {str(e)}")
```

---

## Part 4: Fuel Management

### 4.1 Fuel Policy Configuration

Per Kenya Government Transport Policy:

1. **Fuel Allocation**
   - Based on vehicle type and usage
   - Monthly limits per department
   - Emergency fuel provisions

2. **Fuel Stations**
   - Authorized fuel stations list
   - Fuel card integration
   - Receipt verification

### 4.2 Fuel Log Workflow

1. **Daily Fuel Entry**
   - Driver enters fuel details
   - Attach receipt photo
   - Submit for approval

2. **Approval Workflow**
   - Fleet Manager reviews
   - Check against allocation
   - Approve or reject

3. **Reconciliation**
   - Match with fuel card statements
   - Flag anomalies
   - Generate reports

### 4.3 Fuel Monitoring Report

Create custom report:

```python
# erpnext_gvms_integration/report/fuel_consumption_report/fuel_consumption_report.py

import frappe
from frappe.utils import flt

def execute(filters=None):
    columns = [
        {"fieldname": "vehicle", "fieldtype": "Link", "options": "Vehicle", "label": "Vehicle"},
        {"fieldname": "department", "fieldtype": "Link", "options": "Department", "label": "Department"},
        {"fieldname": "total_fuel", "fieldtype": "Float", "label": "Total Fuel (L)"},
        {"fieldname": "total_cost", "fieldtype": "Currency", "label": "Total Cost"},
        {"fieldname": "avg_consumption", "fieldtype": "Float", "label": "Avg L/100km"},
        {"fieldname": "mileage", "fieldtype": "Float", "label": "Total Mileage (km)"}
    ]
    
    data = frappe.db.sql("""
        SELECT 
            fl.vehicle,
            v.department,
            SUM(fl.fuel_quantity) as total_fuel,
            SUM(fl.fuel_cost) as total_cost,
            AVG(fl.fuel_quantity / NULLIF(fl.odometer_reading - LAG(fl.odometer_reading) OVER (PARTITION BY fl.vehicle ORDER BY fl.date), 0) * 100) as avg_consumption,
            MAX(fl.odometer_reading) - MIN(fl.odometer_reading) as mileage
        FROM `tabFuel Log` fl
        JOIN `tabVehicle` v ON fl.vehicle = v.name
        WHERE fl.date BETWEEN %(from_date)s AND %(to_date)s
        GROUP BY fl.vehicle, v.department
    """, filters, as_dict=1)
    
    return columns, data
```

---

## Part 5: Maintenance Management

### 5.1 Preventive Maintenance Schedule

Create maintenance schedules:

| Vehicle Type | Service Interval | Oil Change | Tire Rotation | Major Service |
|--------------|------------------|------------|---------------|---------------|
| Executive | 5,000 km | 5,000 km | 10,000 km | 30,000 km |
| Official | 5,000 km | 5,000 km | 10,000 km | 30,000 km |
| General | 5,000 km | 5,000 km | 10,000 km | 25,000 km |
| Utility | 5,000 km | 5,000 km | 10,000 km | 20,000 km |

### 5.2 Maintenance Workflow

1. **Service Reminder**
   - System generates alerts based on odometer/date
   - Notify fleet manager and assigned driver

2. **Service Authorization**
   - Fleet Manager creates service request
   - Get quotes from authorized vendors
   - Approve service order

3. **Service Execution**
   - Vehicle taken to service center
   - Update maintenance record
   - Attach service receipts

4. **Cost Tracking**
   - Track maintenance costs per vehicle
   - Calculate cost per kilometer
   - Generate maintenance reports

### 5.3 Authorized Service Centers

Create supplier master for service centers:
- Name, Address, Contact
- Specialization (Engine, Body, Electrical)
- Rates, Payment Terms
- Authorization Status

---

## Part 6: Driver Management

### 6.1 Driver Requirements (Kenya)

Per Government Transport Policy:

1. **Qualifications**
   - Valid driving license (appropriate class)
   - Defensive driving certificate
   - First aid certificate
   - Clean driving record

2. **License Classes**
   - Class B: Light vehicles
   - Class C: Medium vehicles
   - Class D: Heavy vehicles
   - Class A: Motorcycles

### 6.2 Driver Master

Create **Driver** doctype:

- Employee (Link)
- License Number (Data)
- License Class (Select)
- License Expiry (Date)
- Issue Date (Date)
- Issuing Authority (Data)
- Endorsements (Table)
- Driving Record (Table: Driving Incident)

### 6.3 Driver Assignment

1. **Eligibility Check**
   - Valid license
   - Appropriate license class
   - No active incidents
   - Defensive driving certified

2. **Assignment Process**
   - Employee requests vehicle
   - Manager approves
   - Fleet Manager assigns vehicle
   - Update vehicle status

### 6.4 Driver Monitoring

Track driver performance:
- Fuel efficiency
- Maintenance incidents
- Accidents
- Compliance with policies

---

## Part 7: Insurance Management

### 7.1 Insurance Types

Government vehicle insurance:
- Comprehensive (Executive/Security vehicles)
- Third Party (General pool vehicles)
- Specialized (Ambulances, Construction)

### 7.2 Insurance Policy Management

Create **Insurance Policy** doctype:
- Policy Number (Data)
- Insurance Company (Link: Supplier)
- Vehicle (Link)
- Policy Type (Select)
- Start Date (Date)
- End Date (Date)
- Premium Amount (Currency)
- Coverage Amount (Currency)
- Status (Select: Active, Expired, Cancelled)

### 7.3 Insurance Alerts

Configure automated alerts:
- 30 days before expiry
- 7 days before expiry
- On expiry date
- Renewal reminders

---

## Part 8: Accident Management

### 8.1 Accident Reporting

Create **Accident Report** doctype:
- Vehicle (Link)
- Driver (Link)
- Date (Date)
- Location (Data)
- Description (Text)
- Injuries (Table)
- Damage Assessment (Table)
- Police Report Number (Data)
- Insurance Claim Number (Data)
- Status (Select: Reported, Under Investigation, Closed)

### 8.2 Accident Workflow

1. **Immediate Response**
   - Report to fleet manager
   - File police report
   - Notify insurance

2. **Investigation**
   - Investigating officer assigned
   - Collect evidence
   - Determine cause

3. **Resolution**
   - Repair authorization
   - Insurance claim
   - Driver action (if required)
   - Update records

### 8.3 Accident Statistics

Create report for accident analysis:
- Accidents per vehicle
- Accidents per driver
- Cost of accidents
- Trend analysis

---

## Part 9: Disposal Management

### 9.1 Disposal Criteria (Kenya Government)

Per Government Transport Policy:

1. **Age-Based Disposal**
   - Executive vehicles: 10 years
   - Official vehicles: 8 years
   - General vehicles: 7 years
   - Utility vehicles: 10 years

2. **Condition-Based Disposal**
   - Beyond economical repair
   - Failed inspection
   - Safety concerns

3. **Donor-Funded Vehicles**
   - Account for after project closure
   - Return to government pool
   - Dispose per policy

### 9.2 Disposal Process

1. **Inspection**
   - Technical inspection committee
   - Condition assessment
   - Valuation

2. **Approval**
   - Fleet Manager recommendation
   - Accounting Officer approval
   - National Treasury (for high value)

3. **Disposal Method**
   - Public auction
   - Transfer to other MDAC
   - Scrapping

4. **Documentation**
   - Disposal certificate
   - Transfer documents
   - Update GVMS

### 9.3 Disposal DocType

Create **Vehicle Disposal** doctype:
- Vehicle (Link)
- Disposal Reason (Select)
- Inspection Date (Date)
- Valuation Amount (Currency)
- Disposal Method (Select)
- Buyer (Data)
- Disposal Amount (Currency)
- Certificate Number (Data)
- Status (Select: Proposed, Approved, Completed)

---

## Part 10: Reporting

### 10.1 Fleet Reports

1. **Vehicle Register**
   - All vehicles with details
   - Filter by department, status, age

2. **Fleet Utilization**
   - Vehicle usage statistics
   - Idle time analysis

3. **Cost Analysis**
   - Cost per vehicle
   - Cost per kilometer
   - Department-wise costs

### 10.2 Compliance Reports

1. **Insurance Compliance**
   - Vehicles with active insurance
   - Expired policies

2. **Maintenance Compliance**
   - Vehicles overdue for service
   - Maintenance history

3. **License Compliance**
   - Driver license validity
   - Expired licenses

### 10.3 GVMS Reports

1. **Tracking Summary**
   - Vehicle locations
   - Movement patterns

2. **Fuel Report**
   - Consumption trends
   - Anomaly detection

3. **Performance Report**
   - Vehicle performance metrics
   - Driver performance

---

## Part 11: Dashboard Configuration

### 11.1 Fleet Dashboard

Create custom dashboard:

```python
# erpnext_gvms_integration/dashboard/fleet_dashboard.py

import frappe

def get_data():
    return {
        "heatmap": None,
        "transactions": [
            {"label": "Vehicle", "type": "Link", "options": "Vehicle"},
            {"label": "Fuel Log", "type": "Link", "options": "Fuel Log"},
            {"label": "Maintenance", "type": "Link", "options": "Maintenance Record"},
            {"label": "Accidents", "type": "Link", "options": "Accident Report"}
        ],
        "graphs": [
            {
                "name": "Fleet Summary",
                "chart_name": "Fleet Summary",
                "chart_type": "Donut",
                "is_custom": True,
                "filters": [],
                "values": get_fleet_summary()
            }
        ]
    }

def get_fleet_summary():
    data = frappe.db.sql("""
        SELECT status, COUNT(*) as count
        FROM tabVehicle
        GROUP BY status
    """, as_dict=1)
    
    return [{"title": d.status, "value": d.count} for d in data]
```

### 11.2 GVMS Dashboard

Real-time dashboard showing:
- Vehicle locations on map
- Current status of all vehicles
- Alerts and notifications
- Key metrics

---

## Part 12: Mobile App Features

### 12.1 Driver Mobile App

Features for drivers:
- View assigned vehicle
- Submit fuel logs
- Report maintenance issues
- View driving history

### 12.2 Fleet Manager Mobile App

Features for fleet managers:
- View fleet status
- Approve requests
- View GPS tracking
- Generate reports

---

## Part 13: Security & Access Control

### 13.1 Roles

| Role | Access |
|------|--------|
| Fleet Manager | Full access |
| Driver | View assigned vehicle, submit logs |
| Department Head | View department vehicles, approve requests |
| Finance | View costs, approve payments |
| Admin | Configuration, reports |

### 13.2 Permissions

- Vehicle: Fleet Manager (Full), Driver (Read own)
- Fuel Log: Driver (Create), Fleet Manager (Full)
- Maintenance: Fleet Manager (Full), Driver (Read)
- Accident Report: Fleet Manager (Full), Driver (Create)

---

## Part 14: Deployment Checklist

### Pre-Deployment
- [ ] Configure Fleet Management module
- [ ] Setup GVMS integration
- [ ] Configure GPS tracking providers
- [ ] Setup fuel card integration

### Data Migration
- [ ] Import vehicle register
- [ ] Import driver records
- [ ] Import insurance policies
- [ ] Import maintenance history

### Testing
- [ ] Test vehicle assignment workflow
- [ ] Test fuel log submission
- [ ] Test maintenance workflow
- [ ] Test GPS tracking sync
- [ ] Test GVMS integration

### Go-Live
- [ ] Train fleet managers
- [ ] Train drivers
- [ ] Monitor system
- [ ] Address issues

---

## Appendix: Useful Commands

```bash
# Backup fleet data
bench --site your-site.com backup --with-files

# Sync all vehicles to GVMS
bench --site your-site.com execute erpnext_gvms_integration.api.sync_all_vehicles

# Generate fleet report
bench --site your-site.com execute erpnext_gvms_integration.report.generate_fleet_report

# Check vehicle status
bench --site your-site.com mariadb -e "SELECT name, status FROM tabVehicle"
```

---

## References

1. Government Transport Policy 2024 (Draft)
2. Government Vehicle Management System (GVMS) - https://gvms.go.ke
3. Kenya e-GP - https://egpkenya.go.ke
4. Public Procurement and Asset Disposal Act 2015
5. PPADA Regulations 2020
