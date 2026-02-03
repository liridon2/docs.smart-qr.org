# 🍽️ SmartQR – Digital Ordering System for Restaurants

Welcome to **SmartQR** – a B2B SaaS solution that enables restaurants, cafés, and bars to digitalize their ordering process. Guests simply scan a QR code at the table, browse the menu, and place orders directly from their smartphone.  

> 🚀 Built by **B21 Solutions**  
> 🌍 Deployed on **STRATO Shared Hosting**  
> 💻 Frontend: **Vite + React + TypeScript**  
> 🔧 Backend: **PHP + MySQL** (Shared-Hosting-friendly)  

---

## ✨ Features

- 📱 **Customer App**
  - Scan QR → Browse menu
  - Add items to cart
  - Place orders without needing an account
  - Sessionless flow with local storage

- 🛠️ **Admin Dashboard**
  - Live order overview
  - Manage order status (new → in progress → served → paid)
  - Real-time table status
  - Sales overview & daily revenue

- 🔄 **Realtime Updates**
  - Powered by **Ably** (or SSE fallback for shared hosting)
  - Instant order notifications for staff

- 💳 **Stripe Integration**
  - Subscription-based SaaS model
  - Webhooks for subscription management
  - Feature flags per plan

- 📊 **Analytics (planned)**
  - Peak hours
  - Best-selling items
  - Weekly/Monthly revenue charts

---

## 🏗️ Tech Stack

| Layer       | Technology            |
|-------------|-----------------------|
| Frontend    | Vite + React + TS     |
| Styling     | Tailwind CSS          |
| State Mgmt  | React Query / Zustand |
| Backend     | PHP 8.x + MySQL (PDO) |
| Hosting     | STRATO Shared Hosting |
| Realtime    | Ably / SSE fallback   |
| Payments    | Stripe                |

---

## 📂 Repository Structure

```bash
.
├── /frontend           # React app (Vite + TS)
│   ├── src
│   └── public
├── /backend            # PHP backend (API)
│   ├── public          # index.php, .htaccess
│   ├── src             # Router, Services, Models
│   └── migrations      # SQL migrations
├── /docs               # Documentation (setup, onboarding, API)
└── README.md
```

---

## 🚀 Getting Started

### 1. Clone the Repo
```bash
git clone https://github.com/<org>/<repo>.git
cd smartqr
```

### 2. Frontend Setup
```bash
cd frontend
npm install
npm run dev
```

### 3. Backend Setup
- Upload `/backend/public` to your STRATO webspace.  
- Configure database connection in `config.php`.  
- Run migrations from `/backend/migrations`.  

---

## 📖 Documentation

- [📚 Project Onboarding Guide](./docs/onboarding.md)  
- [🔌 API Reference](./docs/api.md)  
- [🛠️ Deployment on STRATO](./docs/deploy_strato.md)  
- [📊 Architecture Overview](./docs/architecture.md)  

---

## 🧑‍💻 Contributing

We ❤️ contributions!  

1. Fork the repo & create a feature branch (`git checkout -b feature/my-feature`)  
2. Commit your changes (`git commit -m "Add new feature"`)  
3. Push to the branch (`git push origin feature/my-feature`)  
4. Create a Pull Request 🎉  

👉 For details, check out our [Onboarding Guide](./docs/onboarding.md).

---

## 📅 Project Management

We use **GitHub Issues** for:
- 🐞 Bug tracking  
- 💡 Feature requests  
- 📋 Task management  

Each issue should be linked to a milestone & labeled accordingly.

---

## 🔒 Security

- PDO prepared statements for all DB queries  
- Strict JSON response handling (never HTML)  
- CSRF protection for Admin UI  
- Rate limiting for API endpoints  
- HTTPS enforced  

---

## 👥 Team

- 🧑‍💻 **Liridon Bytyqi** – Founder & Lead Developer  
- 🌐 **B21 Solutions** – Software Company  

---

## 📜 License

© 2025 B21 Solutions. All rights reserved.  
---

## Environment Configuration

### Backend `.env`

| Variable | Description | Example |
|----------|-------------|---------|
| `APP_ENV` | Application environment name | `production` |
| `APP_DEBUG` | Enable verbose PHP errors | `false` |
| `APP_URL` | Public URL for the app shell | `https://app.smartqr.example` |
| `APP_SECRET` | Symmetric application secret (64+ chars) | `change-me` |
| `DB_HOST` | Database host | `localhost` |
| `DB_DATABASE` | Database schema | `smartqr` |
| `DB_USERNAME` | Database username | `smartqr` |
| `DB_PASSWORD` | Database password | `super-secure-password` |
| `JWT_SECRET` | JWT signing secret (64+ chars) | `change-me-jwt` |
| `JWT_ACCESS_TOKEN_TTL` | Access token lifetime in seconds | `3600` |
| `JWT_REFRESH_TOKEN_TTL` | Refresh token lifetime in seconds | `2592000` |
| `STRIPE_SECRET_KEY` | Stripe secret API key | `sk_live_xxx` |
| `STRIPE_PUBLIC_KEY` | Stripe publishable key | `pk_live_xxx` |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret | `whsec_xxx` |
| `RATE_LIMIT_REQUESTS` | Requests allowed per window | `60` |
| `RATE_LIMIT_WINDOW` | Window length in seconds | `60` |
| `REALTIME_PROVIDER` | Realtime transport provider | `ably` |
| `REALTIME_ABLY_API_KEY` | Ably API key | `ably-xxx` |
| `REALTIME_ABLY_APP_ID` | Ably app id | `ably-app` |
| `SMTP_FROM_EMAIL` | Default sender email | `noreply@smartqr.example` |
| `SMTP_FROM_NAME` | Default sender name | `SmartQR` |
| `CORS_ALLOWED_ORIGINS` | Comma separated allowed origins | `http://localhost:5173,https://app.smartqr.example` |
| `CORS_ALLOW_CREDENTIALS` | Allow cookies/headers across origins | `true` |
| `CORS_ALLOWED_METHODS` | Comma separated HTTP methods | `GET,POST,PATCH,DELETE,OPTIONS` |
| `CORS_ALLOWED_HEADERS` | Comma separated headers | `Content-Type,Authorization,X-Admin-Token` |
| `CORS_MAX_AGE` | Seconds to cache preflight | `86400` |

### Frontend `.env`

| Variable | Description | Example |
|----------|-------------|---------|
| `VITE_APP_NAME` | Display name for the UI | `SmartQR` |
| `VITE_APP_ENV` | Frontend environment label | `production` |
| `VITE_API_BASE_URL` | Base URL for API requests | `https://api.smartqr.example` |
| `VITE_API_TIMEOUT_MS` | Request timeout in milliseconds | `15000` |
| `VITE_API_MAX_RETRIES` | Automatic retry attempts for transient errors | `1` |
| `VITE_API_RETRY_DELAY_MS` | Delay between retries in milliseconds | `500` |
