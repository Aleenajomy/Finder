# Finder: Detailed Project Description Document

**Finder** is a hyper-local worker directory application built as a modern Progressive Web Application (PWA) with a Django REST Framework (DRF) backend and a React (Vite-powered) frontend. The platform connects users directly with local service providers (plumbers, electricians, painters, cleaners, etc.) without requiring registration, bookings, or transaction fees. It operates with a fully decentralized, passwordless worker model where listing updates/removals are verified using phone numbers as proof of ownership, backed by administrative moderation.

---

## Table of Contents
1. [System Architecture](#1-system-architecture)
2. [Database Schema & Data Models](#2-database-schema--data-models)
3. [Core Product Modules & Business Logic](#3-core-product-modules--business-logic)
4. [Third-Party APIs & Integration Endpoints](#4-third-party-apis--integration-endpoints)
5. [API Routes & Controller Map](#5-api-routes--controller-map)
6. [Frontend Architecture & Page Routes](#6-frontend-architecture--page-routes)
7. [Local Setup & Environment Configuration](#7-local-setup--environment-configuration)

---

## 1. System Architecture

Finder utilizes a decoupled client-server architecture with:
- **Frontend Client**: React 18, React Router v6, and Lucide React icons, bundled with Vite 5. Includes PWA capabilities via `vite-plugin-pwa` for offline assets caching, a mobile bottom nav bar, and instant call/WhatsApp actions.
- **Backend API Server**: Django 6.0.3 and Django REST Framework (DRF) 3.16. Exposes a stateless REST API, utilizes Django's Admin Panel for content moderation, and manages database interactions.
- **Database Layer**: PostgreSQL database managed via Django's Object-Relational Mapper (ORM).

### Architecture Diagram
```mermaid
graph TD
    subgraph Client [React PWA Client]
        UI[React Components / UI Pages]
        State[React States & Hooks]
        Translation[Bilingual System <br/> English + Malayalam]
        Locate[Postal Pincode API / Address Resolution]
        SW[Service Worker / Cache]
    end

    subgraph API [Django REST Framework API Server]
        CORS[django-cors-headers]
        Throttling[DRF Rate Throttling]
        Views[API Views]
        Serializers[DRF Serializers]
        AdminPanel[Django Admin Console]
    end

    subgraph DB [Database Layer]
        ORM[Django ORM]
        Postgres[(PostgreSQL Database)]
    end

    UI -->|JSON API Requests| CORS
    CORS --> Throttling
    Throttling --> Views
    Views --> Serializers
    Serializers --> ORM
    ORM --> Postgres
    AdminPanel -->|Review, Approve, & Moderation| Postgres
```

### Infrastructure & Deployment Stack
- **Frontend Hosting**: Served as static hosted files.
- **Backend API Hosting**: Served using Gunicorn + WhiteNoise on Render.
- **Database**: Serverless PostgreSQL (NeonDB or local PostgreSQL).
- **Static Assets Manager**: Django WhiteNoise (serving Django static files securely in production environments).

---

## 2. Database Schema & Data Models

Finder's relational database models are defined in Django's directory app ([models.py](file:///d:/Project/Finder/Finder/directory/models.py)):

```mermaid
erDiagram
    Location ||--o{ Worker : "contains"
    Category ||--o{ Worker : "classifies"
    Category ||--o{ WorkerSubmission : "submits to"
    Worker ||--o{ Rating : "receives"
    Worker ||--o{ WorkerUpdateRequest : "requests"
    Worker ||--o| WorkerSubmission : "references"
```

### 2.1 Location
Maintains physical coordinates/pincode mapping for hyper-local filtering.
- `id` (Auto-incrementing Integer, Primary Key).
- `city` (CharField, max length 100): City name in English.
- `city_ml` (CharField, max length 100, default empty): City name in Malayalam.
- `area_name` (CharField, max length 150): Area/Neighborhood name in English.
- `area_name_ml` (CharField, max length 150, default empty): Area/Neighborhood name in Malayalam.
- `pincode` (CharField, max length 10, db_index=True): Indian postal pincode.
- *Constraints*: `unique_together = ("area_name", "pincode")`.
- *Default Ordering*: `("city", "area_name")`.

### 2.2 Category
Represents service classifications.
- `id` (Auto-incrementing Integer, Primary Key).
- `name` (CharField, max length 100, unique=True): English category name.
- `name_ml` (CharField, max length 100, default empty): Malayalam category name.
- `icon_url` (URLField, max length 500, nullable): URL for category illustration.
- `search_keywords` (CharField, max length 300, default empty): Synonyms and search tags to improve matches.
- *Default Ordering*: `("name",)`.

### 2.3 Worker
Represents the active and listed local worker profile.
- `id` (Auto-incrementing Integer, Primary Key).
- `name` (CharField, max length 200): English/transliterated name.
- `name_ml` (CharField, max length 200, default empty): Malayalam script name.
- `phone_number` (CharField, max length 15, unique=True): Normalized unique contact number.
- `category` (ForeignKey -> Category, on_delete=CASCADE).
- `location` (ForeignKey -> Location, on_delete=CASCADE).
- `is_verified` (BooleanField, default=False): Admin-verified flag.
- `availability_status` (BooleanField, default=True): Active status (Available vs. Busy).
- `call_clicks` (PositiveIntegerField, default=0): Total direct phone call trigger count.
- `whatsapp_clicks` (PositiveIntegerField, default=0): Total WhatsApp message trigger count.
- `created_at` (DateTimeField, auto_now_add=True).
- *Properties*:
  - `normalized_phone_number`: Strips all non-digit characters.
  - `whatsapp_url`: Formats link to `https://wa.me/<number>`.
  - `call_url`: Formats string to `tel:<number>`.
- *Default Ordering*: `("-is_verified", "-availability_status", "name")`.

### 2.4 WorkerSubmission
Records new registrations from the `/join` page. Submissions automatically create an unverified `Worker` entry immediately.
- `id` (Auto-incrementing Integer, Primary Key).
- `name` (CharField, max length 200): Submitter's name.
- `phone_number` (CharField, max length 15, db_index=True): Submitter's contact.
- `category` (ForeignKey -> Category, on_delete=CASCADE).
- `city` (CharField, max length 100).
- `area_name` (CharField, max length 150).
- `pincode` (CharField, max length 10, db_index=True).
- `service_description` (TextField): Detailed description of services offered.
- `availability_status` (BooleanField, default=True).
- `consent_to_contact` (BooleanField, default=True).
- `status` (CharField, default='pending', choices: `pending`, `approved`, `rejected`).
- `approved_worker` (OneToOneField -> Worker, on_delete=SET_NULL, null=True, blank=True): The corresponding `Worker` profile generated.
- `reviewed_at` (DateTimeField, null=True).
- `created_at` (DateTimeField, auto_now_add=True).
- *Default Ordering*: `("-created_at",)`.

### 2.5 WorkerUpdateRequest
Represents requested updates to active worker profiles. Modifying critical details (name, category, location, description) submits a request to the admin pipeline.
- `id` (Auto-incrementing Integer, Primary Key).
- `worker` (ForeignKey -> Worker, on_delete=CASCADE).
- `name` (CharField, max length 200, blank=True).
- `phone_number` (CharField, max length 15, blank=True): New phone number requested.
- `category` (ForeignKey -> Category, null=True, blank=True).
- `city` (CharField, max length 100, blank=True).
- `area_name` (CharField, max length 150, blank=True).
- `pincode` (CharField, max length 10, blank=True).
- `service_description` (TextField, blank=True).
- `availability_status` (BooleanField, null=True, blank=True).
- `status` (CharField, default='pending', choices: `pending`, `approved`, `rejected`).
- `reviewed_at` (DateTimeField, null=True).
- `created_at` (DateTimeField, auto_now_add=True).
- *Default Ordering*: `("-created_at",)`.

### 2.6 Rating
Allows clients to rate workers anonymously (from 1 to 5 stars).
- `id` (Auto-incrementing Integer, Primary Key).
- `worker` (ForeignKey -> Worker, on_delete=CASCADE).
- `rating` (IntegerField, choices: 1–5).
- `client_ip` (CharField, max length 45, blank=True, db_index=True): Used for rate-limiting (maximum one rating per IP per worker).
- `created_at` (DateTimeField, auto_now_add=True).
- *Default Ordering*: `("-created_at",)`.

---

## 3. Core Product Modules & Business Logic

### 3.1 Passwordless Worker Management (Self-Service)
To maximize accessibility and reduce friction, Finder does not use usernames or passwords:
- Workers submit their details from `/join`.
- An unverified `Worker` profile is automatically created and shown immediately in search results.
- When workers want to update or delete their listing from their profile page (`/workers/:id`), they input their registered phone number.
- The system checks if the input matches the stored phone number:
  - **Deletions** take place instantly.
  - **Updates** generate a `WorkerUpdateRequest` which sits in a pending queue for admin review (ensuring names/services remain appropriate and locations are valid).
- A background client poller checks the status of the update request every 5 seconds. Once approved by an admin, the profile refreshes instantly.

### 3.2 Bilingual Transliteration & Localization
The directory is optimized for regions like Kerala, India, supporting English and Malayalam:
- **Toggle**: EN / ML toggle is accessible globally.
- **Real-time Transliteration**: In the `/join` registration form, typing an English name and tabbing out fetches a transliterated Malayalam name automatically using Google Input Tools API (`https://inputtools.google.com/request?...`).
- **Reverse Detection**: Typing Malayalam first automatically duplicates the text in the English field. The backend detects Malayalam Unicode block values (`0D00` to `0D7F`) to appropriately structure the database values.

### 3.3 Dynamic Pincode Mapping & Address Resolution
To prevent location typing mistakes, Finder integrates India's Postal PIN Code API:
- When a worker enters a 6-digit pincode in the registration or update forms, the frontend queries `https://api.postalpincode.in/pincode/<pincode>`.
- The system automatically parses the district and post offices list, auto-populating the "City" and providing a dropdown selector of post office areas for "Area Name".

### 3.4 Call & WhatsApp Click Tracking
Finder acts as a lead generation directory. When a user clicks "Call" or "WhatsApp" on a worker's card:
- The system fires a POST request to `/api/workers/<id>/track-call/` or `/api/workers/<id>/track-whatsapp/`.
- Django increments the respective click counter field atomically in the database (`F("call_clicks") + 1` / `F("whatsapp_clicks") + 1`).
- The browser then redirects to the dialer (`tel:<phone>`) or WhatsApp API link.

### 3.5 Rating Protection & Throttling
To protect workers from review bombing:
- An IP address filter checks if the client has already rated the worker (`Rating.objects.filter(worker=worker, client_ip=client_ip).exists()`). If so, it blocks the review submission and returns a 409 Conflict.
- An API rate throttle limits ratings submissions to 30 per hour per IP.

---

## 4. Third-Party APIs & Integration Endpoints

Finder leverages public open APIs:
1. **Google Input Tools API**:
   - URL: `https://inputtools.google.com/request`
   - Purpose: Performs real-time English-to-Malayalam transliteration for worker names on focus loss.
2. **Indian Postal PIN Code API**:
   - URL: `https://api.postalpincode.in/pincode/<pincode>`
   - Purpose: Dynamic location lookup based on Indian ZIP codes to fetch cities and post offices.

---

## 5. API Routes & Controller Map

All REST endpoints are exposed via Django REST Framework. By default, API results are paginated (20 per page).

| Method | Endpoint | Query / Body Params | Controller View | Description |
|--------|----------|---------------------|-----------------|-------------|
| **GET** | `/api/home/` | None | `HomeDataAPIView` | Retrieves directory statistics (workers, categories, verified count) along with categories, top locations, and featured verified workers. |
| **GET** | `/api/categories/` | None | `CategoryListAPIView` | Lists all categories and appends worker count annotations. |
| **GET** | `/api/locations/` | `?q=<term>` or `?pincode=<pin>` | `LocationListAPIView` | Searches and filters locations containing workers. |
| **GET** | `/api/workers/` | `?category=<id/name>&pincode=<pin>&search=<term>&verified=<bool>&available=<bool>&page=<int>` | `WorkerListAPIView` | Fetches active (verified) worker profiles matching query filters. |
| **GET** | `/api/workers/<id>/` | `?phone=<phone>` | `WorkerDetailAPIView` | Retrieves detailed worker profile. Unverified profiles require the registered phone number. |
| **PATCH** | `/api/workers/<id>/self/` | `{ phone_number, name, category, city, area_name, pincode, service_description, availability_status }` | `WorkerSelfServiceAPIView` | Submits a pending update request for the worker. Requires phone number verification. |
| **DELETE** | `/api/workers/<id>/self/` | `{ phone_number }` | `WorkerSelfServiceAPIView` | Instantly deletes the worker's profile. Requires phone number verification. |
| **GET** | `/api/workers/<id>/pending-update/` | `?phone=<phone>` | `WorkerPendingUpdateAPIView` | Checks if there is an active pending update request for the worker. |
| **POST** | `/api/workers/<id>/track-call/` | None | `WorkerTrackCallAPIView` | Atomically increments call click counter. |
| **POST** | `/api/workers/<id>/track-whatsapp/` | None | `WorkerTrackWhatsAppAPIView` | Atomically increments WhatsApp click counter. |
| **POST** | `/api/workers/<id>/ratings/` | `{ rating }` | `RatingCreateAPIView` | Submits a 1-5 star review. Throttled and restricted by IP. |
| **POST** | `/api/worker-submissions/` | `{ name, phone_number, category, city, area_name, pincode, service_description, availability_status }` | `WorkerSubmissionCreateAPIView` | Submits a new worker listing. Instantly creates an unverified worker entry. |
| **GET** | `/api/submission-status/` | `?phone=<phone>` | `SubmissionStatusAPIView` | Tracks the status (`pending`, `approved`, `rejected`) of the latest submission matching the phone number. |

---

## 6. Frontend Architecture & Page Routes

The client-side is structured as a light-weight React Single Page Application (SPA) in [App.jsx](file:///d:/Project/Finder/Finder/frontend/src/App.jsx) and styling is compiled in [styles.css](file:///d:/Project/Finder/Finder/frontend/src/styles.css).

### 6.1 State Management & Translation Context
- **Translations Engine**: Translation JSON structures (`translations.en` and `translations.ml`) are mapped dynamically. React states `lang` and `text` render text values automatically when the language is toggled.
- **Search & Filters Deferral**: The homepage filters use React 18's `useDeferredValue` on the search query input (`deferredSearch`) and pincode (`deferredPincode`). This delays API request triggers until the user finishes typing, reducing network overhead.

### 6.2 Frontend Page Routing (React Router v6)
Using hash-based routing, the app handles three views:
1. **Homepage (`/`)**:
   - Hero banner with stats cards.
   - Search bar (by service/name) and Pincode input (with suggestions list).
   - Category grid with worker counters.
   - List of matching workers with interactive call, WhatsApp, and details links.
2. **Registration Page (`/join`)**:
   - Double bilingual inputs (English & Malayalam names).
   - Automated postal validation.
   - Worker listing status checker (remembers and checks the last submitted number in `localStorage`).
3. **Worker Details Page (`/workers/:id`)**:
   - Comprehensive profile showing category, description, and ratings.
   - Self-service update form (location changes warn about admin review).
   - Self-service deletion button.
   - 5-star ratings submission card.

---

## 7. Local Setup & Environment Configuration

### Prerequisites
- Python 3.10+
- Node.js 18+
- PostgreSQL database (running locally or cloud instance)

### Environment Variables

#### Backend `.env` ([settings.py](file:///d:/Project/Finder/Finder/server/settings.py)):
```env
SECRET_KEY=django-secure-random-key
DEBUG=True
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/finder
ALLOWED_HOSTS=127.0.0.1,localhost
```

#### Frontend `.env` ([App.jsx](file:///d:/Project/Finder/Finder/frontend/src/App.jsx)):
```env
VITE_API_BASE_URL=http://127.0.0.1:8000/api
```

### Installation Steps

1. **Activate Virtual Environment & Install Django Dependencies**:
   ```powershell
   cd d:\Project\Finder\Finder
   python -m venv venv
   .\venv\Scripts\activate
   pip install -r requirements.txt
   ```

2. **Run Migrations & Seed Local Database**:
   ```powershell
   python manage.py migrate
   python manage.py seed_directory
   ```

3. **Start Django API Server**:
   ```powershell
   python manage.py runserver
   ```

4. **Install & Run Frontend Client**:
   ```powershell
   cd .\frontend
   npm install
   npm run dev
   ```

5. **Run Django Test Suite**:
   ```powershell
   python manage.py test
   ```
