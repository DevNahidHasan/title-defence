# Nirapod Parking: Complete System Blueprint & Defense Specification

> **Execution Strategy:**
> * **Stage 1 (Core Microservices + Minimal MVP):** Lay out the enterprise Spring Cloud Microservices architecture (`api-gateway`, `discovery-service`, `user-service`, `parking-service`, `booking-service`) with **Database-per-Service isolation**, secured by **Keycloak (OAuth2 / OIDC)**, including Card-First search, online KYC, basic booking, and **Host/Admin Dashboard Summary Cards (`SUM`/`COUNT`)**.
> * **Stage 2 (Geospatial "Wow Factor"):** Plug in **Haversine Proximity Search** (*"Find Near Me"*) and the **Airbnb-style Split-View Leaflet Clustered Map** (`Leaflet.markercluster`) using the GPS coordinates already captured in Stage 1.
> * **Stage 3 (Lightweight Defense Polish):** Add a simple SQL overlap check (preventing double-booking) and a printable **Digital Parking Pass** (with Vehicle Plate matching).

---

## 1. System Architecture (Microservices + Database-per-Service + Keycloak OAuth2)

```mermaid
flowchart TB
    subgraph Client ["Angular 17+ Frontend"]
        DriverView["🚗 Driver View<br>(Card Feed + Split Clustered Map + Booking Pass)"]
        HostView["🏠 Host View<br>('Become a Host' KYC + Listing Mgmt + Earnings Stats)"]
        AdminView["⚙️ Admin Portal<br>(Host KYC Approval + System Summary Cards)"]
    end

    subgraph Auth_Infra ["Identity & Infrastructure"]
        Keycloak["🔐 Keycloak Server (OAuth2 / OIDC)<br>(Login, Registration, JWT Issuing, Roles)"]
        KC_DB[("keycloak_db")]
        Eureka["🧭 discovery-service<br>(Netflix Eureka Server)"]
        Gateway["🌐 api-gateway<br>(Spring Cloud Gateway + OAuth2 Token Relay)"]
        Keycloak --> KC_DB
    end

    subgraph Microservices ["Core Business Microservices (Spring Boot 3 + OpenFeign)"]
        UserSvc["👤 1. user-service<br>(Unified Profile, Vehicle Plate, Host KYC & Admin Approval)"]
        ParkingSvc["🅿️ 2. parking-service<br>(Garage Listings, Area Search & Haversine Geo-Search)"]
        BookingSvc["📅 3. booking-service<br>(Slot Reservation, Fare Calculation & Overlap Check)"]
    end

    subgraph Databases ["Isolated Database-per-Service Layer (PostgreSQL / MySQL)"]
        UserDB[("🗄️ user_db<br>(users, host_kyc)")]
        ParkingDB[("🗄️ parking_db<br>(parking_listings)")]
        BookingDB[("🗄️ booking_db<br>(bookings)")]
    end

    subgraph External_Free ["Zero-Cost Map & Browser Tools"]
        GPS["📍 HTML5 Browser Geolocation<br>(1-Click Lat/Lng Capture)"]
        GMaps["🗺️ Google Maps Iframe & Deep-Link<br>(Stage 1 Preview & Navigation)"]
        Leaflet["🗺️ Leaflet.js + OpenStreetMap + MarkerCluster<br>(Stage 2 Split-View Clustered Map)"]
    end

    Client <-->|OAuth2 Login| Keycloak
    Client <-->|REST + Bearer JWT| Gateway

    Gateway --> UserSvc
    Gateway --> ParkingSvc
    Gateway --> BookingSvc

    UserSvc -.->|Registers| Eureka
    ParkingSvc -.->|Registers| Eureka
    BookingSvc -.->|Registers| Eureka
    Gateway -.->|Discovers Routes| Eureka

    ParkingSvc <-->|OpenFeign: Verify Host KYC| UserSvc
    BookingSvc <-->|OpenFeign: Fetch Slot Rate & Status| ParkingSvc

    UserSvc --> UserDB
    ParkingSvc --> ParkingDB
    BookingSvc --> BookingDB

    HostView -.-> GPS
    DriverView -.-> GMaps
    DriverView -.-> Leaflet
```

---

## 2. Database-per-Service Schema (3 Isolated Databases, 4 Core Tables)

Each microservice owns its own isolated database. There are **no cross-database foreign keys**; cross-service references (`host_user_id`, `listing_id`, `driver_user_id`) are stored as logical IDs and validated via **OpenFeign** REST calls.

### A. `user_db` (Owned exclusively by `user-service`)
```mermaid
erDiagram
    USERS ||--o| HOST_KYC : submits

    USERS {
        bigint id PK
        string keycloak_id UK "Sub claim from Keycloak JWT"
        string full_name
        string email UK
        string phone_number
        string vehicle_plate "e.g., Dhaka Metro-Ga 12-3456"
        boolean is_verified_host "Default false; true after Admin KYC approval"
        timestamp created_at
    }

    HOST_KYC {
        bigint id PK
        bigint user_id FK "References users.id in user_db"
        string nid_number
        string nid_photo_url
        string ownership_doc_url "Holding Tax Receipt / Parking Allotment Slip"
        string slot_photo_url "Photo of Garage Pillar showing Flat/Bay No"
        enum status "PENDING, APPROVED, REJECTED"
        string admin_remarks
    }
```

### B. `parking_db` (Owned exclusively by `parking-service`)
```mermaid
erDiagram
    PARKING_LISTINGS {
        bigint id PK
        bigint host_user_id "Logical ID -> user-service (users.id)"
        string title "e.g., Covered Garage at Green Heritage"
        string area_name "e.g., Dhanmondi, Banani, Mirpur, Uttara"
        string full_address "House 24, Road 4, Flat 4A, Bay A3"
        double latitude "Captured via 1-click Browser GPS"
        double longitude "Captured via 1-click Browser GPS"
        enum vehicle_type "CAR, BIKE"
        decimal hourly_rate "e.g., 50.00"
        time available_from "e.g., 09:00"
        time available_until "e.g., 18:00"
        boolean has_cctv
        boolean has_guard
        enum status "ACTIVE, OCCUPIED, INACTIVE"
    }
```

### C. `booking_db` (Owned exclusively by `booking-service`)
```mermaid
erDiagram
    BOOKINGS {
        bigint id PK
        bigint listing_id "Logical ID -> parking-service (parking_listings.id)"
        bigint host_user_id "Logical ID -> user-service (for fast Host earnings SUM)"
        bigint driver_user_id "Logical ID -> user-service (users.id)"
        string driver_vehicle_plate "Displayed on Booking Pass for Building Guard"
        datetime start_time
        datetime end_time
        decimal total_amount "Hours x Hourly Rate"
        enum status "CONFIRMED, COMPLETED, CANCELLED"
        timestamp created_at
    }
```

---

## 3. Stage 1 End-to-End Sequence Workflow

### Unified Dual-Role Concept (The "Mirpur-to-Banani Commuter" Model)
Every user signs up once via **Keycloak** and starts as a **Driver** by default. If they also own a flat with an empty daytime garage, they click **`[ 🏠 Become a Host ]`**, upload their KYC proof, and once approved by the Admin, they can switch freely between **`Driver View`** and **`Host Dashboard`** using the same account.

```mermaid
sequenceDiagram
    actor User as User (Driver / Host)
    actor Admin as Admin
    participant UI as Angular App
    participant KC as Keycloak (OAuth2)
    participant GW as API Gateway
    participant US as user-service (user_db)
    participant PS as parking-service (parking_db)
    participant BS as booking-service (booking_db)

    Note over User, US: FLOW 1: Sign Up & Default Driver Profile
    User->>KC: 1. Registers / Logs in via Keycloak Login Page
    KC-->>UI: 2. Returns OAuth2 Access Token (Default Role: DRIVER)
    UI->>GW: 3. Completes Profile (Phone + Vehicle Plate No)
    GW->>US: 4. Saves User Profile in user_db linked to keycloak_id

    Note over User, US: FLOW 2: "Become a Host" & Admin KYC Approval
    User->>UI: 5. Clicks "Become a Host" & uploads NID + Tax/Allotment Slip + Slot Photo
    UI->>GW: 6. POST /users/kyc
    GW->>US: 7. Saves KYC request in user_db (status = PENDING)
    Admin->>GW: 8. Views Admin Summary Cards & Pending KYC -> clicks "Approve"
    GW->>US: 9. Updates status = APPROVED & sets is_verified_host = true

    Note over User, PS: FLOW 3: Host Creates a Parking Listing
    User->>UI: 10. Switches to Host View -> Fills Listing Form + Clicks "Capture GPS"
    UI->>GW: 11. POST /listings (Area, Address, Rate, Lat, Lng, Vehicle Type)
    GW->>PS: 12. PS checks US (via OpenFeign) that is_verified_host == true -> Saves in parking_db

    Note over User, BS: FLOW 4: Driver Searches, Books & Navigates
    User->>UI: 13. Searches by Area ("Dhanmondi") & Vehicle ("CAR")
    UI->>GW: 14. GET /listings/search?area=Dhanmondi&type=CAR
    GW->>PS: 15. Returns matching active listings for Card Feed
    User->>UI: 16. Opens Card -> Views iframe Map Preview -> Picks Time & clicks "Book Now"
    UI->>GW: 17. POST /bookings (listingId, startTime, endTime)
    GW->>BS: 18. BS fetches Listing rate from PS (OpenFeign), calculates Total Fare & saves in booking_db
    BS-->>UI: 19. Booking Confirmed! Updates Host Summary Stats + Unlocks Google Maps Navigation
```

---

## 4. Three-Stage Build Roadmap

```mermaid
flowchart LR
    S1["🟢 STAGE 1: Core Microservices MVP<br>(Keycloak + 3 DBs + Card Feed + Stats Cards)"] --> S2["🔵 STAGE 2: Geospatial Wow-Factor<br>(Haversine Near-Me + Clustered Map)"]
    S2 --> S3["🟣 STAGE 3: Defense Polish<br>(Overlap Check + Printable Booking Pass)"]
```

### 🟢 STAGE 1: Core Microservices Architecture + Minimal Functional MVP
*Goal: Get the entire distributed architecture, isolated databases, and core end-to-end flow running first.*
* **Infrastructure Setup:** Run Keycloak (`keycloak_db`), `discovery-service` (Eureka), and `api-gateway` (Spring Cloud Gateway).
* **`user-service` (`user_db`):** Sync logged-in Keycloak user, save Driver phone & vehicle plate, submit Host KYC form (NID + Holding Tax/Allotment Slip + Slot Photo), and Admin 1-click `[Approve]` button.
* **`parking-service` (`parking_db`):** Verified Host creates a parking listing with 1-click browser GPS (`navigator.geolocation`), toggles spot status (`ACTIVE / INACTIVE`), and Driver searches by **Area dropdown** + **Vehicle Type** in a clean **Card Feed**.
* **`booking-service` (`booking_db`):** Driver books a spot for a selected time window; service calls `parking-service` via **OpenFeign** to calculate price (`Hours × Rate`), saves booking, and unlocks the **Google Maps iframe preview** & **URL navigation redirect**.
* **Basic Dashboard Summary Cards (`SUM` / `COUNT`):**
  * **Host Dashboard:** 3 counter cards showing *Active Spots* (`COUNT` from `parking_db`), *Total Bookings* (`COUNT` from `booking_db`), and *Total Earnings ৳* (`SUM(total_amount)` from `booking_db`).
  * **Admin Portal:** 3 counter cards showing *Pending KYC Requests*, *Verified Hosts* (`COUNT` from `user_db`), and *Total Active Listings* (`COUNT` from `parking_db`).

---

### 🔵 STAGE 2: Geospatial "Wow Factor" Upgrades
*Goal: Add high-impact visual and algorithmic features for the Defense Committee without changing your database schema.*
* **"Find Near Me" Proximity Search (Haversine SQL):** Add a single `@Query` in `parking-service` using the Haversine distance formula so drivers can find spots within `2 km` of their current GPS coordinate and see **`"0.4 km away"`** on every card.
* **Airbnb-Style Split Screen (Card Feed + Leaflet Clustered Map):**
  * Render **Leaflet.js + OpenStreetMap** (100% free, no API key) alongside the Card Feed.
  * Add **`Leaflet.markercluster`** so if 3 flat owners in the same building list spots at the same coordinate, the map automatically groups them into a **`(3)`** cluster pin that fans out ("spiderfies") when clicked.
  * Clicking a card smoothly pans/zooms the map (`map.flyTo`) to that building's pin.

---

### 🟣 STAGE 3: Minimal, High-Visual Polish for Final Defense
*Goal: Quick 1-hour additions that make the live demo look complete without complex business logic.*
* **Simple Double-Booking Prevention:** A single SQL `COUNT(*)` check in `booking-service` ensuring no existing `CONFIRMED` booking overlaps the requested `start_time` and `end_time`.
* **Printable Digital Parking Pass:** A clean ticket screen showing the Booking ID, **Driver's Vehicle Plate Number** (so the building guard can visually verify the car at the gate), Host address/phone, time window, and a simulated *"Paid via Mock Wallet"* badge.

---

## 5. How This Maps to Your University Defense Phases

| Defense Phase | What You Present / Demo | Why It Wins Marks |
| :--- | :--- | :--- |
| **Phase 1: Title Defense** *(Current)* | Present the **Microservices + Database-per-Service Architecture**, **Isolated ERDs**, **Dual-Role KYC concept**, and the **3-Stage Roadmap**. | Shows true microservices isolation (`user_db`, `parking_db`, `booking_db`) + Keycloak OAuth2 + OpenFeign + Geospatial clustering. |
| **Phase 2: Pre-Defense** *(Mid-term)* | Demo **Stage 1 (100% Working Microservices MVP + Dashboard Summary Cards)** + start of **Stage 2**. | Complete working flow from Keycloak login $\rightarrow$ Host KYC approval $\rightarrow$ Listing creation $\rightarrow$ Booking & live dashboard counters. |
| **Phase 3: Final Defense** *(End-term)* | Demo **Stage 1 + Stage 2 (Haversine + Clustered Map) + Stage 3 (Overlap Prevention & Digital Pass)**. | Visual split-screen map clustering + live proximity search + complete microservices demonstration. |
