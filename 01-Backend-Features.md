# 🚗 Ride Book — Backend Features Document

> **Version:** 1.0 (MVP)  
> **Last Updated:** April 24, 2026  
> **Tech Stack:** Node.js / Express.js + MySQL/PostgreSQL + Socket.IO

---

## 1. Overview

Ride Book backend 3 main modules handle karega:

| Module | Platform | Users |
|--------|----------|-------|
| **Rider Module** | Android App | Passengers who book rides |
| **Driver Module** | Android App | Drivers who accept & complete rides |
| **Admin Module** | Web Dashboard | Admin who monitors everything |

---

## 2. Authentication & Authorization

### 2.1 User Registration & Login
- Phone number + OTP based login (via SMS Gateway)
- JWT token based session management
- Role-based access: `RIDER`, `DRIVER`, `ADMIN`

### 2.2 API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/auth/send-otp` | Send OTP to phone number |
| POST | `/api/auth/verify-otp` | Verify OTP & get JWT token |
| POST | `/api/auth/register` | Register new user (rider/driver) |
| POST | `/api/auth/refresh-token` | Refresh expired JWT token |
| POST | `/api/auth/logout` | Invalidate session |

### 2.3 Driver Verification
- Document upload (License, RC, Aadhar, PAN)
- Admin manually approves/rejects driver
- Driver status: `PENDING` → `APPROVED` / `REJECTED`

---

## 3. Rider Features (Android App — User Side)

### 3.1 Profile Management
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/rider/profile` | Get rider profile |
| PUT | `/api/rider/profile` | Update rider profile |
| POST | `/api/rider/profile/photo` | Upload profile photo |

### 3.2 Ride Booking Flow
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/ride/estimate` | Get fare estimate (pickup → drop) |
| POST | `/api/ride/book` | Create a new ride request |
| GET | `/api/ride/:id` | Get ride details |
| POST | `/api/ride/:id/cancel` | Cancel a ride |
| GET | `/api/ride/history` | Get rider's ride history |
| GET | `/api/ride/:id/track` | Real-time driver location tracking |

### 3.3 Ride Booking Logic
```
1. Rider enters Pickup & Drop location
2. Backend calculates distance (Google Maps API)
3. Backend returns fare estimate
4. Rider confirms booking
5. Backend finds nearby available drivers (within 5km radius)
6. Send ride request to nearest driver via Socket.IO
7. If driver doesn't respond in 30 sec → send to next driver
8. Driver accepts → Ride confirmed
9. Real-time tracking starts
10. Driver arrives → Trip starts → Trip ends
11. Fare calculated & ride completed
```

### 3.4 Rating & Review
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/ride/:id/rate` | Rate driver after ride (1-5 stars) |

---

## 4. Driver Features (Android App — Driver Side)

### 4.1 Profile & Documents
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/driver/profile` | Get driver profile |
| PUT | `/api/driver/profile` | Update driver profile |
| POST | `/api/driver/documents` | Upload verification documents |
| GET | `/api/driver/documents/status` | Check verification status |

### 4.2 Availability & Ride Management
| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/api/driver/go-online` | Mark driver as available |
| POST | `/api/driver/go-offline` | Mark driver as unavailable |
| POST | `/api/driver/location` | Update current GPS location |
| GET | `/api/driver/ride/current` | Get current assigned ride |
| POST | `/api/driver/ride/:id/accept` | Accept a ride request |
| POST | `/api/driver/ride/:id/reject` | Reject a ride request |
| POST | `/api/driver/ride/:id/arrived` | Mark arrived at pickup |
| POST | `/api/driver/ride/:id/start` | Start the trip |
| POST | `/api/driver/ride/:id/complete` | Complete the trip |
| GET | `/api/driver/ride/history` | Get ride history |
| GET | `/api/driver/earnings` | Get earnings summary |

### 4.3 Driver Location Updates
- Driver app sends GPS coordinates every **5 seconds** via Socket.IO
- Backend stores latest location in **Redis** for fast lookup
- Used for: finding nearby drivers + real-time tracking

---

## 5. Admin Features (Web Dashboard)

### 5.1 Dashboard Overview
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/dashboard` | Dashboard stats (total rides, users, revenue) |
| GET | `/api/admin/dashboard/live` | Live active rides count |

### 5.2 User Management
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/riders` | List all riders (paginated) |
| GET | `/api/admin/riders/:id` | Get rider details |
| PUT | `/api/admin/riders/:id/block` | Block/Unblock a rider |
| GET | `/api/admin/drivers` | List all drivers (paginated) |
| GET | `/api/admin/drivers/:id` | Get driver details |
| PUT | `/api/admin/drivers/:id/verify` | Approve/Reject driver |
| PUT | `/api/admin/drivers/:id/block` | Block/Unblock a driver |

### 5.3 Ride Management
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/rides` | List all rides (with filters) |
| GET | `/api/admin/rides/:id` | Get ride full details |
| GET | `/api/admin/rides/active` | Get all currently active rides |

### 5.4 Reports & Analytics
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/reports/rides` | Ride reports (daily/weekly/monthly) |
| GET | `/api/admin/reports/revenue` | Revenue reports |
| GET | `/api/admin/reports/drivers` | Driver performance reports |

### 5.5 Settings
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/admin/settings` | Get app settings |
| PUT | `/api/admin/settings` | Update settings (fare/km, commission %) |
| GET | `/api/admin/vehicle-types` | List vehicle types |
| POST | `/api/admin/vehicle-types` | Add vehicle type |
| PUT | `/api/admin/vehicle-types/:id` | Update vehicle type & pricing |

---

## 6. Real-Time Features (Socket.IO)

| Event | Direction | Description |
|-------|-----------|-------------|
| `driver:location-update` | Driver → Server | Driver sends GPS location |
| `ride:new-request` | Server → Driver | New ride request for driver |
| `ride:accepted` | Server → Rider | Driver accepted the ride |
| `ride:driver-location` | Server → Rider | Real-time driver location |
| `ride:arrived` | Server → Rider | Driver arrived at pickup |
| `ride:started` | Server → Rider | Trip started |
| `ride:completed` | Server → Rider | Trip completed with fare |
| `ride:cancelled` | Server → Both | Ride was cancelled |

---

## 7. Fare Calculation Logic

```
Base Fare = ₹30 (configurable by admin)
Per KM Rate = ₹12/km (configurable by admin)
Per Minute Rate = ₹2/min (configurable by admin)
Minimum Fare = ₹50

Total Fare = Base Fare + (Distance × Per KM Rate) + (Duration × Per Minute Rate)
if Total Fare < Minimum Fare → Total Fare = Minimum Fare

Platform Commission = Total Fare × Commission% (e.g., 20%)
Driver Earnings = Total Fare - Platform Commission
```

---

## 8. Middleware & Security

| Feature | Implementation |
|---------|---------------|
| Authentication | JWT Bearer Token |
| Rate Limiting | Express Rate Limit (100 req/min) |
| Input Validation | Joi / express-validator |
| CORS | Configured for allowed origins |
| Helmet | HTTP security headers |
| File Upload | Multer with size limits |
| Error Handling | Centralized error handler |
| Logging | Winston logger |

---

## 9. Folder Structure (Backend)

```
ride-book-backend/
├── src/
│   ├── config/          # DB, Redis, env configs
│   ├── controllers/     # Route handlers
│   │   ├── auth.controller.js
│   │   ├── rider.controller.js
│   │   ├── driver.controller.js
│   │   ├── ride.controller.js
│   │   └── admin.controller.js
│   ├── middleware/       # Auth, validation, error handling
│   ├── models/          # Database models (Sequelize/Prisma)
│   ├── routes/          # API route definitions
│   ├── services/        # Business logic
│   │   ├── auth.service.js
│   │   ├── ride.service.js
│   │   ├── location.service.js
│   │   ├── fare.service.js
│   │   ├── notification.service.js
│   │   └── maps.service.js
│   ├── sockets/         # Socket.IO event handlers
│   ├── utils/           # Helper functions
│   └── app.js           # Express app setup
├── .env
├── package.json
└── README.md
```

---
