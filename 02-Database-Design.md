# 🗄️ Ride Book — Database Design Document

> **Version:** 1.0 (MVP)  
> **Last Updated:** April 24, 2026  
> **Database:** MySQL / PostgreSQL

---

## 1. Entity Relationship Overview

```
┌──────────┐       ┌──────────┐       ┌──────────────┐
│  USERS   │──1:1──│ RIDERS   │──1:N──│    RIDES     │
│          │       └──────────┘       │              │
│          │       ┌──────────┐       │              │
│          │──1:1──│ DRIVERS  │──1:N──│              │
│          │       │          │       └──────┬───────┘
└──────────┘       │          │──1:N──┌──────┴───────┐
                   └──────────┘       │ RIDE_TRACKING│
                                      └──────────────┘
```

---

## 2. All Tables Summary

| # | Table Name | Description |
|---|-----------|-------------|
| 1 | `users` | All users (riders, drivers, admins) |
| 2 | `riders` | Rider-specific profile info |
| 3 | `drivers` | Driver-specific profile & documents |
| 4 | `driver_documents` | Uploaded documents for verification |
| 5 | `vehicle_types` | Vehicle categories & pricing |
| 6 | `vehicles` | Vehicles registered by drivers |
| 7 | `rides` | All ride bookings |
| 8 | `ride_tracking` | GPS tracking data during rides |
| 9 | `ratings` | Rider ↔ Driver ratings |
| 10 | `otp_verifications` | OTP records for login |
| 11 | `admin_settings` | App-wide configurations |
| 12 | `notifications` | Push notification logs |

---

## 3. Detailed Table Schemas

### 3.1 `users` — Master User Table

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Unique user ID |
| phone | VARCHAR(15) | UNIQUE, NOT NULL | Phone number |
| name | VARCHAR(100) | NOT NULL | Full name |
| email | VARCHAR(150) | NULLABLE, UNIQUE | Email address |
| profile_photo | VARCHAR(255) | NULLABLE | Profile photo URL |
| role | ENUM('RIDER','DRIVER','ADMIN') | NOT NULL | User role |
| status | ENUM('ACTIVE','BLOCKED','INACTIVE') | DEFAULT 'ACTIVE' | Account status |
| created_at | TIMESTAMP | DEFAULT NOW() | Registration date |
| updated_at | TIMESTAMP | ON UPDATE NOW() | Last update |

**Indexes:** `phone` (UNIQUE), `role`, `status`

---

### 3.2 `riders` — Rider Profile

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Rider ID |
| user_id | BIGINT | FK → users.id, UNIQUE | Linked user |
| home_address | TEXT | NULLABLE | Saved home address |
| home_lat | DECIMAL(10,8) | NULLABLE | Home latitude |
| home_lng | DECIMAL(11,8) | NULLABLE | Home longitude |
| work_address | TEXT | NULLABLE | Saved work address |
| work_lat | DECIMAL(10,8) | NULLABLE | Work latitude |
| work_lng | DECIMAL(11,8) | NULLABLE | Work longitude |
| total_rides | INT | DEFAULT 0 | Total rides taken |
| avg_rating | DECIMAL(3,2) | DEFAULT 5.00 | Average rating |
| created_at | TIMESTAMP | DEFAULT NOW() | — |

---

### 3.3 `drivers` — Driver Profile

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Driver ID |
| user_id | BIGINT | FK → users.id, UNIQUE | Linked user |
| license_number | VARCHAR(50) | NOT NULL | Driving license |
| is_verified | ENUM('PENDING','APPROVED','REJECTED') | DEFAULT 'PENDING' | Verification status |
| is_online | BOOLEAN | DEFAULT FALSE | Currently available? |
| current_lat | DECIMAL(10,8) | NULLABLE | Current latitude |
| current_lng | DECIMAL(11,8) | NULLABLE | Current longitude |
| current_location_updated_at | TIMESTAMP | NULLABLE | Last location update |
| vehicle_id | BIGINT | FK → vehicles.id, NULLABLE | Assigned vehicle |
| total_rides | INT | DEFAULT 0 | Total rides completed |
| avg_rating | DECIMAL(3,2) | DEFAULT 5.00 | Average rating |
| total_earnings | DECIMAL(12,2) | DEFAULT 0.00 | Lifetime earnings |
| created_at | TIMESTAMP | DEFAULT NOW() | — |

**Indexes:** `is_online`, `is_verified`, `current_lat + current_lng` (SPATIAL/COMPOSITE)

---

### 3.4 `driver_documents` — Verification Documents

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Document ID |
| driver_id | BIGINT | FK → drivers.id | Linked driver |
| document_type | ENUM('AADHAR','PAN','LICENSE','RC','INSURANCE','PHOTO') | NOT NULL | Doc type |
| document_url | VARCHAR(255) | NOT NULL | Uploaded file URL |
| status | ENUM('PENDING','APPROVED','REJECTED') | DEFAULT 'PENDING' | Verification status |
| admin_remarks | TEXT | NULLABLE | Admin's notes |
| uploaded_at | TIMESTAMP | DEFAULT NOW() | Upload time |
| verified_at | TIMESTAMP | NULLABLE | Verification time |

---

### 3.5 `vehicle_types` — Vehicle Categories

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | ID |
| name | VARCHAR(50) | NOT NULL | e.g., "Mini", "Sedan", "SUV", "Auto" |
| description | VARCHAR(200) | NULLABLE | Category description |
| icon_url | VARCHAR(255) | NULLABLE | Category icon |
| base_fare | DECIMAL(8,2) | NOT NULL | Base fare (₹) |
| per_km_rate | DECIMAL(8,2) | NOT NULL | Rate per kilometer |
| per_min_rate | DECIMAL(8,2) | NOT NULL | Rate per minute |
| min_fare | DECIMAL(8,2) | NOT NULL | Minimum fare |
| commission_percent | DECIMAL(5,2) | NOT NULL | Platform commission % |
| is_active | BOOLEAN | DEFAULT TRUE | Is this type available? |
| created_at | TIMESTAMP | DEFAULT NOW() | — |

---

### 3.6 `vehicles` — Registered Vehicles

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Vehicle ID |
| driver_id | BIGINT | FK → drivers.id | Owner driver |
| vehicle_type_id | BIGINT | FK → vehicle_types.id | Vehicle category |
| make | VARCHAR(50) | NOT NULL | e.g., "Maruti" |
| model | VARCHAR(50) | NOT NULL | e.g., "Swift Dzire" |
| year | YEAR | NOT NULL | Manufacturing year |
| color | VARCHAR(30) | NOT NULL | Vehicle color |
| plate_number | VARCHAR(20) | UNIQUE, NOT NULL | Registration number |
| created_at | TIMESTAMP | DEFAULT NOW() | — |

---

### 3.7 `rides` — Ride Bookings (Core Table)

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Ride ID |
| rider_id | BIGINT | FK → riders.id | Who booked |
| driver_id | BIGINT | FK → drivers.id, NULLABLE | Who accepted |
| vehicle_type_id | BIGINT | FK → vehicle_types.id | Requested vehicle type |
| status | ENUM (see below) | NOT NULL | Current ride status |
| pickup_address | TEXT | NOT NULL | Pickup location text |
| pickup_lat | DECIMAL(10,8) | NOT NULL | Pickup latitude |
| pickup_lng | DECIMAL(11,8) | NOT NULL | Pickup longitude |
| drop_address | TEXT | NOT NULL | Drop location text |
| drop_lat | DECIMAL(10,8) | NOT NULL | Drop latitude |
| drop_lng | DECIMAL(11,8) | NOT NULL | Drop longitude |
| estimated_distance_km | DECIMAL(8,2) | NULLABLE | Estimated distance |
| actual_distance_km | DECIMAL(8,2) | NULLABLE | Actual distance traveled |
| estimated_duration_min | INT | NULLABLE | Estimated time |
| actual_duration_min | INT | NULLABLE | Actual time |
| estimated_fare | DECIMAL(10,2) | NULLABLE | Fare estimate shown to rider |
| actual_fare | DECIMAL(10,2) | NULLABLE | Final fare charged |
| commission_amount | DECIMAL(10,2) | NULLABLE | Platform commission |
| driver_earnings | DECIMAL(10,2) | NULLABLE | Driver's earning |
| payment_method | ENUM('CASH','ONLINE') | DEFAULT 'CASH' | Payment mode |
| cancelled_by | ENUM('RIDER','DRIVER','SYSTEM') | NULLABLE | Who cancelled |
| cancel_reason | TEXT | NULLABLE | Cancellation reason |
| requested_at | TIMESTAMP | DEFAULT NOW() | Booking time |
| accepted_at | TIMESTAMP | NULLABLE | Driver accepted time |
| arrived_at | TIMESTAMP | NULLABLE | Driver arrived time |
| started_at | TIMESTAMP | NULLABLE | Trip started time |
| completed_at | TIMESTAMP | NULLABLE | Trip completed time |
| cancelled_at | TIMESTAMP | NULLABLE | Cancellation time |

**Ride Status Values:**
```
REQUESTED → ACCEPTED → DRIVER_ARRIVED → IN_PROGRESS → COMPLETED
    ↓           ↓                                        
 CANCELLED   CANCELLED                                   
```

**Indexes:** `rider_id`, `driver_id`, `status`, `requested_at`, `pickup_lat + pickup_lng`

---

### 3.8 `ride_tracking` — Real-Time GPS Tracking

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Track ID |
| ride_id | BIGINT | FK → rides.id | Linked ride |
| driver_id | BIGINT | FK → drivers.id | Driver |
| lat | DECIMAL(10,8) | NOT NULL | Latitude |
| lng | DECIMAL(11,8) | NOT NULL | Longitude |
| speed | DECIMAL(5,2) | NULLABLE | Speed in km/h |
| recorded_at | TIMESTAMP | DEFAULT NOW() | Timestamp |

**Index:** `ride_id + recorded_at` (COMPOSITE)

> **Note:** High-frequency data. Consider archiving old records monthly.

---

### 3.9 `ratings` — Rider ↔ Driver Ratings

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Rating ID |
| ride_id | BIGINT | FK → rides.id, UNIQUE | One rating per ride |
| rider_id | BIGINT | FK → riders.id | Rider who rated |
| driver_id | BIGINT | FK → drivers.id | Driver who was rated |
| rating | TINYINT | NOT NULL (1-5) | Star rating |
| comment | TEXT | NULLABLE | Optional review |
| created_at | TIMESTAMP | DEFAULT NOW() | — |

---

### 3.10 `otp_verifications` — OTP Records

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | OTP ID |
| phone | VARCHAR(15) | NOT NULL | Phone number |
| otp_code | VARCHAR(6) | NOT NULL | Hashed OTP code |
| purpose | ENUM('LOGIN','REGISTER') | NOT NULL | OTP purpose |
| is_used | BOOLEAN | DEFAULT FALSE | Already used? |
| expires_at | TIMESTAMP | NOT NULL | Expiry time (5 min) |
| created_at | TIMESTAMP | DEFAULT NOW() | — |

**Index:** `phone + is_used + expires_at`

---

### 3.11 `admin_settings` — App Configuration

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Setting ID |
| setting_key | VARCHAR(100) | UNIQUE, NOT NULL | Setting name |
| setting_value | TEXT | NOT NULL | Setting value |
| description | VARCHAR(255) | NULLABLE | What this setting does |
| updated_at | TIMESTAMP | ON UPDATE NOW() | Last modified |

**Default Settings:**
```
driver_search_radius_km = 5
ride_request_timeout_sec = 30
max_cancel_per_day = 3
support_phone = "+91XXXXXXXXXX"
support_email = "support@ridebook.com"
```

---

### 3.12 `notifications` — Notification Log

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | Notification ID |
| user_id | BIGINT | FK → users.id | Recipient |
| title | VARCHAR(200) | NOT NULL | Notification title |
| body | TEXT | NOT NULL | Notification message |
| type | ENUM('RIDE','PROMO','SYSTEM') | NOT NULL | Category |
| is_read | BOOLEAN | DEFAULT FALSE | Read status |
| created_at | TIMESTAMP | DEFAULT NOW() | — |

---

## 4. Key Relationships Summary

```
users (1) ──── (1) riders
users (1) ──── (1) drivers
drivers (1) ──── (N) driver_documents
drivers (1) ──── (1) vehicles
vehicles (N) ──── (1) vehicle_types
riders (1) ──── (N) rides
drivers (1) ──── (N) rides
rides (1) ──── (N) ride_tracking
rides (1) ──── (1) ratings
users (1) ──── (N) notifications
```

---

## 5. Data Volume Estimates (Monthly)

| Table | Expected Rows/Month | Storage Strategy |
|-------|---------------------|-----------------|
| users | ~5,000 new | Standard |
| rides | ~50,000 new | Index heavily |
| ride_tracking | ~5,000,000+ | Archive monthly, partition by date |
| ratings | ~30,000 new | Standard |
| otp_verifications | ~100,000 | Auto-delete expired (cron job) |
| notifications | ~200,000 | Archive quarterly |

---
