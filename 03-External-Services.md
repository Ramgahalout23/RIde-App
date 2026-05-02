# 🔌 Ride Book — External Services & Integrations

> **Version:** 1.0 (MVP)  
> **Last Updated:** April 24, 2026

---

## 1. Services Overview

```
┌─────────────────────────────────────────────────────────┐
│                    RIDE BOOK APP                        │
├──────────┬──────────┬──────────┬───────────┬───────────┤
│  Maps &  │   SMS    │  Push    │  Cloud    │  Server   │
│ Location │ Gateway  │ Notif.   │ Storage   │ Hosting   │
├──────────┼──────────┼──────────┼───────────┼───────────┤
│ Google   │ Twilio / │ Firebase │ AWS S3 /  │ AWS EC2 / │
│ Maps API │ MSG91    │   FCM    │ Cloudinary│ DigitalO. │
└──────────┴──────────┴──────────┴───────────┴───────────┘
```

---

## 2. Google Maps Platform (Maps, Routes, Places)

### 2.1 Required APIs

| API | Purpose | Used By |
|-----|---------|---------|
| **Maps SDK for Android** | Show maps in rider & driver app | Android App |
| **Places API** | Autocomplete search for pickup/drop locations | Android App |
| **Directions API** | Route between pickup → drop, ETA, polyline | Backend |
| **Distance Matrix API** | Calculate distance & time between two points | Backend |
| **Geocoding API** | Convert lat/lng → address & vice versa | Backend |

### 2.2 Usage in App

| Feature | API Used | Request Flow |
|---------|----------|-------------|
| Rider types pickup location | Places Autocomplete | App → Google directly |
| Show route on map | Directions API | Backend → Google → App |
| Fare estimation | Distance Matrix API | Backend → Google |
| Find nearby drivers | Haversine formula (DB query) | Backend (internal) |
| Real-time tracking on map | Maps SDK + Socket.IO | App renders driver location |

### 2.3 Estimated Cost

| API | Free Tier | After Free Tier |
|-----|-----------|-----------------|
| Maps SDK | Unlimited | Unlimited |
| Places Autocomplete | $200/month credit | ~$2.83 per 1000 requests |
| Directions | $200/month credit | ~$5 per 1000 requests |
| Distance Matrix | $200/month credit | ~$5 per 1000 elements |
| Geocoding | $200/month credit | ~$5 per 1000 requests |

> **Google gives $200 free credit/month** — sufficient for MVP stage (~40,000 requests)

### 2.4 Alternative (Cost Saving)
- **Mapbox** — Cheaper for high volume, 100K free requests/month
- **OpenStreetMap + OSRM** — Completely free, self-hosted routing

---

## 3. SMS Gateway (OTP & Notifications)

### 3.1 Options

| Service | Best For | SMS Cost (India) | OTP API |
|---------|----------|-------------------|---------|
| **MSG91** | India-focused apps | ₹0.15-0.20/SMS | Yes (built-in OTP flow) |
| **Twilio** | International apps | ₹0.25-0.50/SMS | Yes |
| **2Factor** | Budget India apps | ₹0.10-0.15/SMS | Yes |

### 3.2 Recommended: MSG91

**Why MSG91?**
- Indian company, best delivery rates in India
- Built-in OTP send + verify API (no need to build OTP logic)
- DLT registration support
- Very affordable

### 3.3 SMS Use Cases

| Event | Message | Trigger |
|-------|---------|---------|
| Login OTP | "Your OTP is XXXX. Valid for 5 minutes." | User login |
| Ride Booked | "Ride booked! Driver {name} is on the way." | Driver accepts |
| Ride Started | "Trip started. Track your ride in the app." | Trip begins |
| Ride Completed | "Trip completed. Fare: ₹{amount}. Rate your ride!" | Trip ends |

### 3.4 Estimated Cost
- MVP Stage: ~500-1000 OTPs/day
- Cost: ₹75-200/day → ₹2,250-6,000/month

---

## 4. Firebase Cloud Messaging (FCM) — Push Notifications

### 4.1 Why FCM?
- **Free** — No cost regardless of volume
- Native Android support
- Reliable delivery

### 4.2 Push Notification Events

| Event | Recipient | Notification |
|-------|-----------|-------------|
| New Ride Request | Driver | "New ride request! Pickup: {address}" |
| Ride Accepted | Rider | "Driver {name} accepted your ride!" |
| Driver Arrived | Rider | "Your driver has arrived at pickup!" |
| Trip Started | Rider | "Your trip has started." |
| Trip Completed | Rider | "Trip completed. Fare: ₹{amount}" |
| Ride Cancelled | Both | "Ride has been cancelled." |
| Driver Verified | Driver | "Congratulations! Your account is verified." |
| Promotional | All/Segment | Promo messages from admin |

### 4.3 Implementation
```
Backend → Firebase Admin SDK → FCM → Device
```
- Each device registers FCM token on login
- Token stored in `users` table or separate `device_tokens` table
- Backend sends push via Firebase Admin SDK

### 4.4 Cost: **FREE**

---

## 5. Cloud Storage — File Uploads

### 5.1 What Needs Storage?
- Driver documents (Aadhar, PAN, License, RC)
- Profile photos
- Vehicle photos

### 5.2 Options

| Service | Free Tier | Cost After | Best For |
|---------|-----------|------------|----------|
| **AWS S3** | 5 GB/12 months | ~$0.023/GB | Production apps |
| **Cloudinary** | 25 GB + 25K transforms | Pay per usage | Image optimization |
| **Firebase Storage** | 5 GB | $0.026/GB | Quick setup |

### 5.3 Recommended: Cloudinary (MVP) → AWS S3 (Scale)

**MVP:** Cloudinary is easiest — auto image optimization, resize, CDN included.  
**Scale:** Move to AWS S3 + CloudFront CDN for better pricing.

### 5.4 Estimated Storage
- MVP: ~2-5 GB (first 6 months)
- Cost: **FREE** at MVP stage

---

## 6. Server Hosting

### 6.1 Options

| Service | Spec | Cost/Month | Best For |
|---------|------|------------|----------|
| **AWS EC2** (t3.small) | 2 vCPU, 2GB RAM | ~$15-20 | Production |
| **DigitalOcean** Droplet | 2 vCPU, 2GB RAM | $18 | Simple setup |
| **Railway / Render** | Auto-scale | $7-20 | Quick deploy |
| **VPS (Hostinger)** | 2 vCPU, 2GB RAM | ₹500-800 | Budget |

### 6.2 Recommended Architecture (MVP)

```
┌─────────────────────────────────────────┐
│            SINGLE SERVER (MVP)          │
│                                         │
│  ┌──────────┐  ┌────────┐  ┌────────┐ │
│  │ Node.js  │  │ MySQL  │  │ Redis  │ │
│  │ Backend  │  │   DB   │  │ Cache  │ │
│  │ +Socket  │  │        │  │        │ │
│  └──────────┘  └────────┘  └────────┘ │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │      Nginx (Reverse Proxy)      │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘
         ↑
    SSL (Let's Encrypt)
```

### 6.3 Estimated Monthly Cost: ₹1,500-2,500

---

## 7. Database Hosting

| Option | Cost | Best For |
|--------|------|----------|
| **Same server** (MySQL on VPS) | ₹0 extra | MVP |
| **AWS RDS** | $15-30/month | Production |
| **PlanetScale** (MySQL) | Free tier available | Serverless |

### Recommended: Same server for MVP → Separate DB server at scale

---

## 8. Redis (Caching & Real-Time)

### 8.1 Why Redis?
- Store driver locations (fast read/write)
- Cache frequently accessed data
- Socket.IO adapter for scaling

### 8.2 Options
| Option | Cost |
|--------|------|
| Same server (redis-server) | ₹0 |
| **Redis Cloud** (free tier) | Free (30MB) |
| **AWS ElastiCache** | $13+/month |

### Recommended: Install on same VPS for MVP

---

## 9. Admin Web Dashboard Hosting

| Option | Cost |
|--------|------|
| **Same server** (serve React build via Nginx) | ₹0 |
| **Vercel** | Free |
| **Netlify** | Free |

### Recommended: Vercel (free, auto-deploy from GitHub)

---

## 10. Total MVP Monthly Cost Estimate

| Service | Cost (₹) |
|---------|----------|
| Server (VPS) | ₹1,500-2,500 |
| Google Maps APIs | ₹0 (within free credit) |
| SMS (MSG91) | ₹3,000-6,000 |
| Firebase FCM | ₹0 (free) |
| Cloud Storage | ₹0 (free tier) |
| Domain + SSL | ₹100-200 |
| Admin Hosting | ₹0 (Vercel free) |
| Redis | ₹0 (same server) |
| **TOTAL** | **₹4,600-8,700/month** |

---

## 11. Service Integration Diagram

```
                    ┌─────────────┐
                    │  RIDER APP  │
                    │  (Android)  │
                    └──────┬──────┘
                           │
              HTTPS + Socket.IO
                           │
┌──────────┐      ┌───────┴────────┐      ┌──────────┐
│  DRIVER  │◄────►│   BACKEND API  │◄────►│  ADMIN   │
│   APP    │      │   (Node.js)    │      │   WEB    │
│(Android) │      └───┬──┬──┬──┬───┘      │ (React)  │
└──────────┘          │  │  │  │          └──────────┘
                      │  │  │  │
         ┌────────────┘  │  │  └────────────┐
         │               │  │               │
    ┌────┴─────┐  ┌─────┴──┴───┐    ┌─────┴──────┐
    │  MySQL   │  │ Google Maps │    │   MSG91    │
    │ Database │  │   APIs      │    │  SMS OTP   │
    └──────────┘  └────────────┘    └────────────┘
         │
    ┌────┴─────┐  ┌────────────┐    ┌────────────┐
    │  Redis   │  │  Firebase  │    │ Cloudinary │
    │  Cache   │  │    FCM     │    │  Storage   │
    └──────────┘  └────────────┘    └────────────┘
```

---
