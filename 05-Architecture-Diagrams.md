# 📊 Ride Book — Architecture & Flow Diagrams

> **Version:** 1.0 (MVP)  
> **Last Updated:** April 24, 2026  
> **Note:** All diagrams below are in Mermaid format. View them in any Mermaid-compatible viewer (GitHub, VS Code Mermaid Preview extension, mermaid.live)

---

## 1. System Architecture (High-Level)

```mermaid
graph TB
    subgraph "Mobile Apps (Android)"
        RA[🧑 Rider App<br/>Android / Kotlin]
        DA[🚗 Driver App<br/>Android / Kotlin]
    end
    
    subgraph "Web"
        AW[🖥️ Admin Dashboard<br/>React.js]
    end
    
    subgraph "Backend Server"
        API[⚡ REST API<br/>Node.js + Express]
        WS[🔌 WebSocket<br/>Socket.IO]
    end
    
    subgraph "Database Layer"
        DB[(🗄️ MySQL<br/>Database)]
        RD[(⚡ Redis<br/>Cache + Location)]
    end
    
    subgraph "External Services"
        GM[🗺️ Google Maps<br/>API]
        SMS[📱 MSG91<br/>SMS Gateway]
        FCM[🔔 Firebase<br/>FCM]
        CS[☁️ Cloudinary<br/>Storage]
    end
    
    RA <-->|HTTPS + Socket.IO| API
    RA <-->|Real-time| WS
    DA <-->|HTTPS + Socket.IO| API
    DA <-->|Real-time| WS
    AW <-->|HTTPS| API
    
    API --> DB
    API --> RD
    API --> GM
    API --> SMS
    API --> FCM
    API --> CS
    WS --> RD
```

---

## 2. Ride Booking Flow (Complete Journey)

```mermaid
sequenceDiagram
    participant R as 🧑 Rider
    participant APP as 📱 Rider App
    participant API as ⚡ Backend API
    participant DB as 🗄️ Database
    participant RDS as ⚡ Redis
    participant DAPP as 📱 Driver App
    participant D as 🚗 Driver
    participant GM as 🗺️ Google Maps
    participant SMS as 📱 SMS

    R->>APP: Enter Pickup & Drop Location
    APP->>GM: Autocomplete Search
    GM-->>APP: Location Suggestions
    R->>APP: Confirm Locations
    APP->>API: POST /ride/estimate
    API->>GM: Distance Matrix API
    GM-->>API: Distance & Duration
    API->>API: Calculate Fare
    API-->>APP: Fare Estimate (₹XXX)
    R->>APP: Confirm Booking
    APP->>API: POST /ride/book
    API->>DB: Create Ride (REQUESTED)
    API->>RDS: Find Nearby Drivers (5km)
    RDS-->>API: Available Drivers List
    
    loop For Each Nearby Driver
        API->>DAPP: 🔔 New Ride Request (Socket.IO)
        Note over DAPP: 30 sec timeout
        alt Driver Accepts
            D->>DAPP: Accept Ride
            DAPP->>API: POST /driver/ride/:id/accept
            API->>DB: Update Ride (ACCEPTED)
            API->>APP: 🔔 Ride Accepted! (Socket.IO)
            API->>SMS: Send SMS to Rider
        else Driver Rejects / Timeout
            API->>API: Try Next Driver
        end
    end

    Note over R, D: === DRIVER EN ROUTE ===
    
    loop Every 5 seconds
        D->>DAPP: GPS Update
        DAPP->>API: Location Update (Socket.IO)
        API->>RDS: Store Location
        API->>APP: Driver Location (Socket.IO)
        APP->>R: Show Driver on Map
    end

    D->>DAPP: Arrived at Pickup
    DAPP->>API: POST /ride/:id/arrived
    API->>APP: 🔔 Driver Arrived!
    
    D->>DAPP: Start Trip
    DAPP->>API: POST /ride/:id/start
    API->>DB: Update Ride (IN_PROGRESS)
    API->>APP: 🔔 Trip Started!

    Note over R, D: === TRIP IN PROGRESS ===

    D->>DAPP: Complete Trip
    DAPP->>API: POST /ride/:id/complete
    API->>GM: Get Actual Distance
    API->>API: Calculate Final Fare
    API->>DB: Update Ride (COMPLETED + Fare)
    API->>APP: 🔔 Trip Completed! Fare: ₹XXX
    API->>SMS: Send Ride Receipt SMS
    R->>APP: Rate Driver ⭐⭐⭐⭐⭐
```

---

## 3. User Authentication Flow

```mermaid
flowchart TD
    A[User Opens App] --> B{First Time?}
    B -->|Yes| C[Registration Screen]
    B -->|No| D[Login Screen]
    
    C --> E[Enter Name + Phone]
    D --> F[Enter Phone Number]
    
    E --> G[POST /auth/send-otp]
    F --> G
    
    G --> H[MSG91 Sends OTP via SMS]
    H --> I[User Enters OTP]
    I --> J[POST /auth/verify-otp]
    
    J --> K{OTP Valid?}
    K -->|No| L[Show Error - Retry]
    L --> I
    
    K -->|Yes| M{New User?}
    M -->|Yes| N[Create Account in DB]
    M -->|No| O[Fetch Existing Account]
    
    N --> P[Generate JWT Token]
    O --> P
    
    P --> Q{User Role?}
    Q -->|RIDER| R[🧑 Rider Home Screen]
    Q -->|DRIVER| S{Is Verified?}
    
    S -->|PENDING| T[⏳ Verification Pending Screen]
    S -->|APPROVED| U[🚗 Driver Home Screen]
    S -->|REJECTED| V[❌ Re-upload Documents]
```

---

## 4. Driver Onboarding & Verification Flow

```mermaid
flowchart TD
    A[Driver Downloads App] --> B[Register with Phone + OTP]
    B --> C[Fill Profile Details]
    C --> D[Upload Documents]
    
    D --> D1[📄 Driving License]
    D --> D2[📄 Vehicle RC]
    D --> D3[📄 Aadhar Card]
    D --> D4[📄 PAN Card]
    D --> D5[📸 Profile Photo]
    
    D1 & D2 & D3 & D4 & D5 --> E[Documents Uploaded to Cloud Storage]
    E --> F[Status: PENDING]
    F --> G[⏳ Wait for Admin Review]
    
    G --> H[Admin Reviews on Web Dashboard]
    
    H --> I{All Docs Valid?}
    I -->|Yes| J[✅ Status: APPROVED]
    I -->|No| K[❌ Status: REJECTED + Remarks]
    
    J --> L[🔔 Push Notification: Account Verified!]
    K --> M[🔔 Push Notification: Please Re-upload]
    
    L --> N[🚗 Driver Can Go Online & Accept Rides]
    M --> O[Driver Re-uploads Documents]
    O --> F
```

---

## 5. Fare Calculation Flow

```mermaid
flowchart LR
    A[Pickup Location] --> B[Google Distance Matrix API]
    C[Drop Location] --> B
    
    B --> D[Distance: X km]
    B --> E[Duration: Y min]
    
    D --> F[Fare Calculator]
    E --> F
    
    G[Vehicle Type Config<br/>Base: ₹30<br/>Per KM: ₹12<br/>Per Min: ₹2<br/>Min Fare: ₹50] --> F
    
    F --> H{Calculate}
    
    H --> I["Total = Base(₹30) + Distance×₹12 + Duration×₹2"]
    I --> J{Total < ₹50?}
    J -->|Yes| K[Fare = ₹50 Minimum]
    J -->|No| L[Fare = Calculated Amount]
    
    K --> M[Show Estimate to Rider]
    L --> M
    
    M --> N["Commission (20%) = Platform Earning"]
    M --> O["Driver Earning (80%)"]
```

---

## 6. Database Entity Relationship Diagram

```mermaid
erDiagram
    USERS ||--o| RIDERS : "has profile"
    USERS ||--o| DRIVERS : "has profile"
    DRIVERS ||--o{ DRIVER_DOCUMENTS : "uploads"
    DRIVERS ||--o| VEHICLES : "owns"
    VEHICLES }o--|| VEHICLE_TYPES : "belongs to"
    
    RIDERS ||--o{ RIDES : "books"
    DRIVERS ||--o{ RIDES : "accepts"
    RIDES ||--|| VEHICLE_TYPES : "requested type"
    RIDES ||--o{ RIDE_TRACKING : "tracked by"
    RIDES ||--o| RATINGS : "rated"
    
    USERS ||--o{ NOTIFICATIONS : "receives"
    
    USERS {
        bigint id PK
        varchar phone UK
        varchar name
        enum role
        enum status
    }
    
    RIDERS {
        bigint id PK
        bigint user_id FK
        decimal avg_rating
        int total_rides
    }
    
    DRIVERS {
        bigint id PK
        bigint user_id FK
        boolean is_online
        decimal current_lat
        decimal current_lng
        enum is_verified
    }
    
    RIDES {
        bigint id PK
        bigint rider_id FK
        bigint driver_id FK
        enum status
        decimal pickup_lat
        decimal pickup_lng
        decimal drop_lat
        decimal drop_lng
        decimal actual_fare
        timestamp requested_at
    }
    
    VEHICLE_TYPES {
        bigint id PK
        varchar name
        decimal base_fare
        decimal per_km_rate
        decimal per_min_rate
    }
    
    VEHICLES {
        bigint id PK
        bigint driver_id FK
        varchar plate_number UK
        varchar make
        varchar model
    }
    
    RATINGS {
        bigint id PK
        bigint ride_id FK
        tinyint rating
        text comment
    }
```

---

## 7. Admin Dashboard Modules

```mermaid
graph TB
    subgraph "Admin Web Dashboard"
        DASH[📊 Dashboard<br/>Overview & Stats]
        
        subgraph "Management"
            UM[👥 User Management<br/>Riders List, Block/Unblock]
            DM[🚗 Driver Management<br/>Verify, Block, Documents]
            RM[🛣️ Ride Management<br/>All Rides, Active Rides]
        end
        
        subgraph "Analytics"
            RR[📈 Ride Reports<br/>Daily/Weekly/Monthly]
            REV[💰 Revenue Reports<br/>Earnings & Commission]
            DP[📊 Driver Performance<br/>Ratings & Completion Rate]
        end
        
        subgraph "Settings"
            VT[🚐 Vehicle Types<br/>Add/Edit Types & Pricing]
            AS[⚙️ App Settings<br/>Radius, Timeout, Commission]
        end
    end
    
    DASH --> UM
    DASH --> DM
    DASH --> RM
    DASH --> RR
    DASH --> REV
    DASH --> DP
    DASH --> VT
    DASH --> AS
```

---

## 8. Real-Time Communication Architecture

```mermaid
graph LR
    subgraph "Rider App"
        R1[Rider 1]
        R2[Rider 2]
        R3[Rider 3]
    end
    
    subgraph "Socket.IO Server"
        SIO[Socket.IO<br/>Event Hub]
        
        subgraph "Rooms"
            RR1[ride:101]
            RR2[ride:102]
            RR3[ride:103]
        end
    end
    
    subgraph "Driver App"
        D1[Driver A]
        D2[Driver B]
        D3[Driver C]
    end
    
    subgraph "Redis"
        RL[Driver Locations<br/>GeoHash Index]
    end
    
    R1 <--> SIO
    R2 <--> SIO
    R3 <--> SIO
    D1 <--> SIO
    D2 <--> SIO
    D3 <--> SIO
    
    SIO --> RR1
    SIO --> RR2
    SIO --> RR3
    
    SIO <--> RL
    
    D1 -.->|location update<br/>every 5 sec| RL
    D2 -.->|location update<br/>every 5 sec| RL
    D3 -.->|location update<br/>every 5 sec| RL
```

---

## 9. Deployment Architecture (MVP)

```mermaid
graph TB
    subgraph "Internet"
        RIDER[📱 Rider App]
        DRIVER[📱 Driver App]
        ADMIN[🖥️ Admin Web<br/>Vercel]
    end
    
    subgraph "Cloud Server (VPS)"
        NG[🔒 Nginx<br/>Reverse Proxy + SSL]
        
        subgraph "Application"
            NODE[Node.js<br/>Express + Socket.IO<br/>Port 3000]
        end
        
        subgraph "Data"
            MYSQL[(MySQL<br/>Port 3306)]
            REDIS[(Redis<br/>Port 6379)]
        end
    end
    
    subgraph "External APIs"
        GMAP[Google Maps]
        MSG[MSG91 SMS]
        FIRE[Firebase FCM]
        CLOUD[Cloudinary]
    end
    
    RIDER -->|HTTPS:443| NG
    DRIVER -->|HTTPS:443| NG
    ADMIN -->|API calls| NG
    
    NG --> NODE
    NODE --> MYSQL
    NODE --> REDIS
    NODE --> GMAP
    NODE --> MSG
    NODE --> FIRE
    NODE --> CLOUD
```

---

## 10. Ride Status State Machine

```mermaid
stateDiagram-v2
    [*] --> REQUESTED: Rider books ride
    
    REQUESTED --> SEARCHING: Finding drivers
    SEARCHING --> ACCEPTED: Driver accepts
    SEARCHING --> NO_DRIVER: No driver found
    REQUESTED --> CANCELLED_BY_RIDER: Rider cancels
    
    ACCEPTED --> DRIVER_ARRIVED: Driver reaches pickup
    ACCEPTED --> CANCELLED_BY_RIDER: Rider cancels
    ACCEPTED --> CANCELLED_BY_DRIVER: Driver cancels
    
    DRIVER_ARRIVED --> IN_PROGRESS: Trip starts
    DRIVER_ARRIVED --> CANCELLED_BY_RIDER: Rider cancels
    
    IN_PROGRESS --> COMPLETED: Trip ends ✅
    
    COMPLETED --> [*]
    NO_DRIVER --> [*]
    CANCELLED_BY_RIDER --> [*]
    CANCELLED_BY_DRIVER --> [*]
```

---

## How to View These Diagrams

1. **VS Code** — Install "Markdown Preview Mermaid Support" extension
2. **GitHub** — Push this file to GitHub, it renders Mermaid natively
3. **Online** — Copy mermaid code to [mermaid.live](https://mermaid.live)
4. **Export as Image** — Use mermaid.live to export PNG/SVG

---
