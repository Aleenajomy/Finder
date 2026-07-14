# Finder 👷

Finder is a hyper-local, direct-connect worker directory designed to connect community members with local service providers (plumbers, electricians, painters, cleaners, etc.) in a friction-free ecosystem. By removing booking processes, middleman commissions, and mandatory client logins, Finder offers instant connectivity via direct Call and WhatsApp shortcuts.

Developed as a Progressive Web Application (PWA), Finder features offline asset caching, a responsive mobile-first UI, bilingual English/Malayalam translations, and an automated Indian PIN code verification lookup.

---

## 🚀 Key Features

* **Zero-Friction Client Search**: Users can search services (by name or category) and physical areas (by name or pincode) instantly. No login or sign-up required.
* **Passwordless Worker Management**: Workers control their listings using their registered phone number as proof of ownership to update or delete their profiles.
* **Bilingual Translation & Input Transliteration**: Full English/Malayalam support. Includes real-time transliteration (typing an English name auto-fills Malayalam script via Google Input Tools API).
* **Automatic Address Resolution**: Entering a 6-digit Indian PIN code queries the Postal PIN Code API to fetch districts and validate post office names.
* **Admin-Moderated Pipeline**: Submissions and update requests sit in a pending queue for admin review before public verification.
* **PWA Offline Capabilities**: Service worker implementation via `vite-plugin-pwa` allows workers and clients to access cached layout assets offline.
* **Rating System**: 1-5 star feedback restricted to one rating per IP per worker to prevent review bombing.

---

## 🛠️ Technology Stack

* **Backend**: Django 6.0.3, Django REST Framework (DRF) 3.16+, PostgreSQL (NeonDB or local).
* **Frontend**: React 18, React Router v6, Lucide React Icons, Tailwind CSS 3, Vite 5.
* **Server Middleware**: WhiteNoise (static file server), django-cors-headers.
* **APIs Integrated**: Google Input Tools API, Indian Postal PIN Code API.

For a deep dive into the technical details and architecture, check out the [PROJECT_DESCRIPTION.md](file:///d:/Project/Finder/Finder/PROJECT_DESCRIPTION.md) document.

---

## 📂 Project Architecture

```
Finder/
├── directory/                 # Django App for directory logic
│   ├── management/commands/   # Custom commands (seed_directory.py)
│   ├── models.py              # Models (Worker, Location, Submissions, Ratings)
│   ├── serializers.py         # DRF serializers
│   ├── views.py               # API controllers & throttle settings
│   └── urls.py                # App routing
├── server/                    # Django project configuration (settings, wsgi/asgi)
├── frontend/                  # React source files (Vite + PWA)
│   ├── src/
│   │   ├── App.jsx            # Consolidated views & React routing
│   │   └── styles.css         # Styling styles
│   └── package.json           # Frontend dependencies
├── manage.py                  # Django CLI runner
└── requirements.txt           # Backend dependencies
```

---

## ⚙️ Local Development Setup

### 1. Backend Setup

First, activate a virtual environment and install backend dependencies:

```powershell
# Create and activate virtual environment
python -m venv venv
.\venv\Scripts\activate

# Install requirements
pip install -r requirements.txt
```

Create a `.env` file inside the `Finder` directory:

```env
SECRET_KEY=your-secret-key
DEBUG=True
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/finder
ALLOWED_HOSTS=127.0.0.1,localhost
```

Run migrations and seed local dummy data (including mock categories and locations):

```powershell
# Apply schema migrations
python manage.py migrate

# Seed dummy categories, locations, and workers
python manage.py seed_directory
```

Start the backend API server:

```powershell
python manage.py runserver
```
The API server will run at `http://127.0.0.1:8000/api/`.

---

### 2. Frontend Setup

Move into the frontend folder, install npm packages, and spin up the Vite development server:

```powershell
cd .\frontend
npm install
npm run dev
```

Create a `.env` file in the `frontend` folder if the API runs on a different port:

```env
VITE_API_BASE_URL=http://127.0.0.1:8000/api
```

The frontend client will run at `http://127.0.0.1:5173`.

---

## 🧪 Running Verification

To execute Django unit tests (testing serializers, validation rules, rate-limits, and models):

```powershell
python manage.py test
```

---

## 📡 API Endpoint Overview

| Method | Endpoint | Query Params / Body | Description |
|--------|----------|---------------------|-------------|
| **GET** | `/api/home/` | None | Stats, categories list, featured workers. |
| **GET** | `/api/categories/` | None | Service categories list with worker counts. |
| **GET** | `/api/locations/` | `?q=<term>` | Area/pincode search suggestions. |
| **GET** | `/api/workers/` | `?category=<c>&pincode=<p>&search=<s>` | Filter and paginate active listings. |
| **GET** | `/api/workers/:id/` | `?phone=<phone>` | Fetch detailed profile (requires phone if unverified). |
| **PATCH** | `/api/workers/:id/self/` | `{ phone_number, ...updates }` | Submit update request (awaits admin review). |
| **DELETE** | `/api/workers/:id/self/` | `{ phone_number }` | Instantly delete owned listing. |
| **POST** | `/api/workers/:id/ratings/` | `{ rating }` | Rate worker (1-5 stars; limit 1 per IP). |
| **POST** | `/api/workers/:id/track-call/` | None | Track phone call clicks. |
| **POST** | `/api/workers/:id/track-whatsapp/` | None | Track WhatsApp clicks. |
| **POST** | `/api/worker-submissions/` | `{ name, phone_number, ... }` | Register worker profile (starts as unverified). |
| **GET** | `/api/submission-status/` | `?phone=<phone>` | Query status of current worker submission. |

---

## 👮 Content Moderation & Administrative Control

All workers join with an unverified status. To review, verify, approve, or reject submissions and update requests, log in to the Django Admin panel at `http://127.0.0.1:8000/admin/`:
1. **Worker Submissions**: Review details and use the custom action **Approve selected submissions** to make them live/verified.
2. **Worker Update Requests**: Select update requests and execute **Approve selected update requests** or **Reject selected update requests** to process self-service edits.
