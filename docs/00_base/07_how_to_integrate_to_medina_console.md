# Medina Partners Ecosystem Architecture Guide

**Last Updated: 2026-03-05**

**Update History:**
- 2026-03-05: Initial comprehensive documentation of Console SSO, existing services (Dashboard, Brain Studio, HRP Studio), design language, and integration patterns.

---

## Purpose of This Document

This document is written for AI assistants who will help William Chou develop new services within the Medina Partners ecosystem. The goal is to bring any AI assistant to the same level of understanding about the architecture, design patterns, and integration requirements that a thoroughly onboarded developer would have.

When William asks you to build a new frontend service, dashboard, or internal tool, you must first read this document completely. It contains everything you need to know about how the Medina ecosystem works, how services authenticate through Console SSO, what the design language looks like, and how to properly integrate a new service.

This document will be continuously updated as new services are added or architectural decisions change. Always check the "Last Updated" date and the "Update History" section to understand what has changed since your training data.

---

## Part 1: Understanding the Medina Ecosystem

### What is Medina Partners

Medina Partners is William Chou's investment management operation. The Medina ecosystem is a collection of internal tools for portfolio management, quantitative analysis, market intelligence, and trading operations. All tools are internal-use only and require authentication.

### The Services Landscape

As of March 2026, the ecosystem consists of:

**Console** — The central authentication portal and service directory. Located at `/Users/chouwilliam/Medina/console`. Production URL: `console.medina-partners.com`. This is the entry point for all users. It handles login (email verification, password, TOTP 2FA) and provides access to other services.

**Trading Dashboard** — The original monitoring dashboard built with Streamlit. Located at `/Users/chouwilliam/Medina/dashboardv1`. Production URL: `dashboard.medina-partners.com`. This has 20+ pages covering portfolio monitoring, market analysis, options tracking, risk management, and more. While fully functional and actively used, new services should use Next.js rather than Streamlit.

**Brain Studio** — Quantitative factor analysis tool. Located at `/Users/chouwilliam/Medina/brain-studio`. Production URL: `brain.medina-partners.com`. Allows users to discover similar stocks through multi-factor screening and analyze factor performance through quintile-based portfolio construction.

**HRP Studio** — Portfolio optimization tool. Located at `/Users/chouwilliam/Medina/hrp-studio`. Production URL: `hrp.medina-partners.com`. Implements Hierarchical Risk Parity algorithm for optimal portfolio weight calculation with backtesting.

### The Repository Structure

William's development environment is organized under `/Users/chouwilliam/Medina/`:

```
/Users/chouwilliam/Medina/
├── hub/                    # Documentation and prompts hub (this repo)
│   └── prompts/
│       └── 00_base/        # Core documentation including this file
├── console/                # SSO authentication portal
├── dashboardv1/            # Streamlit trading dashboard
├── brain-studio/           # Quantitative factor analysis
├── hrp-studio/             # Portfolio optimization
├── pm/                     # Portfolio management scripts and connectors
│   └── pkg/
│       ├── mongo_connector.py    # Shared MongoDB connection module
│       └── config/
│           └── database.json     # MongoDB credentials
└── [future services]/
```

---

## Part 2: Console SSO Architecture

### The Authentication Flow

Console is the gatekeeper for all Medina services. No service handles its own authentication. Here is exactly how the flow works:

1. **User visits Console**: The user goes to `console.medina-partners.com` and sees the login page.

2. **Email verification**: User enters their email. Console checks if the email is in the whitelist (`CONSOLE_PROD.users` collection). If found, Console sends a 6-digit verification code via Resend email service.

3. **Code + Password + TOTP**: User enters the verification code, then their password, then their TOTP code from their authenticator app. Console verifies all three.

4. **Session created**: Console creates a session and sets a cookie scoped to `.medina-partners.com`. User sees the service directory dashboard.

5. **Clicking a service**: When user clicks on a service (e.g., "Brain Studio"), Console generates a short-lived access token (5 minutes, single-use) and redirects to `https://brain.medina-partners.com?access_token=xyz123`.

6. **Service verifies token**: Brain Studio's frontend extracts the access token from the URL and calls `POST https://console.medina-partners.com/api/auth/verify-access-token` with `{"token": "xyz123"}`.

7. **Console responds**: If valid, Console returns user info and a JWT service token valid for 24 hours:
```json
{
  "valid": true,
  "email": "user@medina-partners.com",
  "name": "User Name",
  "role": "user",
  "user_id": "64abc123...",
  "service": "brain-studio",
  "is_intern": false,
  "token": "eyJhbGciOiJIUzI1NiIs..."
}
```

8. **Service stores JWT**: The frontend stores this JWT in sessionStorage and clears the access token from the URL (security measure).

9. **API calls use JWT**: All subsequent API calls from the frontend to the backend include the JWT in the Authorization header as `Bearer {token}`.

10. **Backend validates JWT**: The backend validates the JWT using the shared secret `JWT_SECRET` (default: `medina-jwt-secret-production-2026`).

### Console Backend Endpoints

Console backend runs FastAPI on port 8002. Key endpoints:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/auth/check-email` | POST | Check if email is whitelisted |
| `/api/auth/send-verification` | POST | Send email verification code |
| `/api/auth/verify-email` | POST | Verify email code |
| `/api/auth/login` | POST | Verify password |
| `/api/auth/verify-totp` | POST | Verify TOTP and create session |
| `/api/auth/me` | GET | Get current user from session |
| `/api/auth/logout` | POST | Invalidate session |
| `/api/auth/generate-access-token` | POST | Generate token for service redirect (Console internal) |
| `/api/auth/verify-access-token` | POST | Verify token from service (Services call this) |

### Console File Locations

```
/Users/chouwilliam/Medina/console/
├── backend/
│   ├── main.py              # FastAPI app entry point
│   ├── api/
│   │   └── routes.py        # All auth endpoints
│   ├── auth/
│   │   ├── email.py         # Resend email integration
│   │   ├── password.py      # Password hashing (bcrypt)
│   │   ├── session.py       # Session management
│   │   └── totp.py          # TOTP generation/verification
│   └── db/
│       └── mongo.py         # MongoDB connection
├── src/
│   └── app/
│       ├── page.tsx         # Login page
│       └── dashboard/
│           └── page.tsx     # Service directory (post-login)
└── docs/
    └── 00_base/
        └── 06_deployment_config.md  # Environment variables
```

### User Permissions

Users are stored in `CONSOLE_PROD.users` with this structure:

```json
{
  "_id": "ObjectId",
  "email": "user@medina-partners.com",
  "name": "User Name",
  "role": "user",           // "user" or "admin"
  "is_active": true,
  "is_intern": false,       // Interns have restricted access
  "is_setup_complete": true,
  "password_hash": "...",
  "totp_secret": "...",
  "allowed_services": ["dashboard", "brain-studio", "hrp-studio"],
  "created_at": "ISODate",
  "updated_at": "ISODate"
}
```

Admins bypass `allowed_services` and have access to everything. The `is_intern` flag can be used within services to hide sensitive features.

---

## Part 3: Technology Stack

### Frontend Stack

All new services must use:

- **Next.js 15** with App Router
- **React 19**
- **TypeScript** (strict mode)
- **Tailwind CSS 4** for styling
- **Recharts** for data visualization
- **Lucide-react** for icons

Next.js configuration must include `output: "standalone"` for Docker deployment.

### Backend Stack

All new services must use:

- **FastAPI** (Python 3.11+)
- **Pydantic** for request/response validation
- **PyMongo** for MongoDB access (no ORM)
- **JWT** validation using PyJWT

### Database

All services connect to a shared MongoDB Atlas cluster on DigitalOcean. Connection is managed through `/Users/chouwilliam/Medina/pm/pkg/mongo_connector.py`.

Key databases:

| Database | Purpose |
|----------|---------|
| CONSOLE_PROD | User auth, sessions, access tokens |
| IBFLEX_PROD | US stock positions and trades from Interactive Brokers |
| PM_PROD | Processed portfolio data, S&P 500 constituents |
| TW_NAS_PROD | Taiwan stock data |
| THETAV2_PROD | US options data |
| FMPV2_PROD | Fundamental data from Financial Modeling Prep |
| BEACON_MONITOR_PROD | Market monitoring data |
| BRAIN_V2_PROD | Quantitative factor data |
| HRP_STUDIO_PROD | Portfolio optimization strategies |

New services should create their own database: `{SERVICE_NAME}_PROD` and `{SERVICE_NAME}_DEV`.

### Deployment

- **Docker** containers for frontend and backend
- **Nginx** reverse proxy with SSL termination
- **Let's Encrypt** for SSL certificates
- **DigitalOcean** server hosting

---

## Part 4: Design Language

### The Visual Identity

Medina services share a unified dark theme with the "Medina blue" accent. This design must be replicated exactly in new services.

### Color Palette

**Backgrounds** (darkest to lightest):
```css
--bg-primary: #000000;      /* Pure black, main background */
--bg-secondary: #0a0a0a;    /* Inputs, secondary surfaces */
--bg-tertiary: #111111;     /* Hover states */
--bg-card: #1a1a1a;         /* Card backgrounds */
--bg-sidebar: #0d0d0d;      /* Sidebar background */
```

**Accent Colors**:
```css
--accent-primary: #0077BB;   /* Medina blue, primary actions */
--accent-secondary: #50AEE4; /* Lighter accent */
--accent-hover: #0099DD;     /* Hover state */
```

**Text Colors**:
```css
--text-primary: #FFFFFF;     /* Primary text, headings */
--text-secondary: #888888;   /* Secondary text */
--text-muted: #666666;       /* Muted text, placeholders */
```

**Border Colors**:
```css
--border-color: #222222;     /* Default borders */
--border-hover: #333333;     /* Hover state borders */
```

**Status Colors**:
```css
--success: #22c55e;          /* Green */
--warning: #eab308;          /* Yellow */
--danger: #ef4444;           /* Red */
```

### Typography

**Body Text**: Outfit (Google Fonts)
- Weights: 300, 400, 500, 600
- Geometric sans-serif for clean readability

**Headings**: Source Serif 4 (Google Fonts)
- Weights: 400, 500
- Sophisticated serif for editorial feel

Load fonts in `layout.tsx`:
```tsx
<link rel="preconnect" href="https://fonts.googleapis.com" />
<link rel="preconnect" href="https://fonts.gstatic.com" crossOrigin="anonymous" />
<link href="https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;500;600&family=Source+Serif+4:wght@400;500&display=swap" rel="stylesheet" />
```

### Component Patterns

**Buttons** (`.btn-primary`):
```css
.btn-primary {
  background-color: var(--accent-primary);
  color: white;
  padding: 10px 24px;
  border-radius: 6px;
  font-weight: 500;
  transition: background-color 0.2s ease;
}
.btn-primary:hover:not(:disabled) {
  background-color: var(--accent-hover);
}
.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
```

**Cards** (`.card`):
```css
.card {
  background-color: var(--bg-card);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  padding: 24px;
  transition: border-color 0.2s ease;
}
.card:hover {
  border-color: var(--border-hover);
}
```

**Inputs** (`.input`):
```css
.input {
  width: 100%;
  padding: 0.75rem 1rem;
  background-color: var(--bg-secondary);
  border: 1px solid var(--border-color);
  border-radius: 6px;
  color: var(--text-primary);
  transition: border-color 0.2s ease;
}
.input:focus {
  outline: none;
  border-color: var(--accent-primary);
}
```

**Badges** (`.badge`, `.badge-success`, `.badge-warning`, `.badge-danger`):
```css
.badge {
  display: inline-flex;
  align-items: center;
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.75rem;
  font-weight: 500;
}
.badge-success {
  background-color: rgba(34, 197, 94, 0.15);
  color: var(--success);
}
.badge-warning {
  background-color: rgba(234, 179, 8, 0.15);
  color: var(--warning);
}
.badge-danger {
  background-color: rgba(239, 68, 68, 0.15);
  color: var(--danger);
}
```

**Tables** (`.table-header`, `.table-row`):
```css
.table-header {
  text-transform: uppercase;
  letter-spacing: 0.05em;
  font-size: 0.75rem;
  color: var(--text-secondary);
  padding: 0.75rem 1rem;
  border-bottom: 1px solid var(--border-color);
}
.table-row {
  border-top: 1px solid var(--border-color);
  transition: background-color 0.15s ease;
}
.table-row:hover {
  background-color: var(--bg-tertiary);
}
```

### Animation

Only use subtle fade-in:
```css
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}
.animate-fadeIn {
  animation: fadeIn 0.3s ease-out;
}
```

---

## Part 5: Project Structure Template

### Frontend Structure

```
your-service/
├── src/
│   ├── app/
│   │   ├── layout.tsx           # Root layout (fonts, metadata, AppShell)
│   │   ├── globals.css          # Design tokens (copy from brain-studio)
│   │   ├── page.tsx             # Home page
│   │   └── [feature]/
│   │       └── page.tsx         # Feature pages
│   ├── components/
│   │   ├── AppShell.tsx         # Auth wrapper (SSO integration)
│   │   ├── Sidebar.tsx          # Navigation sidebar
│   │   └── [Feature]*.tsx       # Feature components
│   └── lib/
│       ├── sso.ts               # Console SSO utilities
│       └── api.ts               # Backend API client
├── public/
│   └── favicon.jpg              # Copy from logo/ folder
├── logo/
│   └── medinapartners_logo_vertical.jpg
├── docs/
│   ├── 00_base/                 # Foundational guides
│   ├── 01_note/                 # Development notes (yyyymmdd_xxx.md)
│   └── 02_plan/                 # Future plans
├── Dockerfile.frontend
├── package.json
├── tsconfig.json
├── next.config.ts               # Must have output: "standalone"
├── postcss.config.mjs
└── tailwind.config.ts
```

### Backend Structure

```
your-service/
├── api/
│   ├── __init__.py
│   ├── main.py                  # FastAPI app
│   ├── auth.py                  # JWT validation
│   ├── schemas.py               # Pydantic models
│   └── routes/
│       └── [feature].py         # Feature routes
├── db/
│   ├── __init__.py
│   └── connector.py             # MongoDB connection
├── services/
│   └── [feature].py             # Business logic
├── engine/                      # Core algorithms (optional)
├── tests/
├── Dockerfile.backend
├── requirements.txt
└── venv/                        # Not committed
```

---

## Part 6: SSO Integration Code

### sso.ts (Frontend)

```typescript
const IS_DEV = process.env.NEXT_PUBLIC_ENV === "DEV";

const CONSOLE_URL = IS_DEV
  ? "http://localhost:3000"
  : "https://console.medina-partners.com";

const CONSOLE_API = IS_DEV
  ? "http://localhost:8001/api"
  : "https://console.medina-partners.com/api";

export interface User {
  email: string;
  name: string;
  role: string;
  user_id: string;
  is_intern: boolean;
  token: string;
}

export async function verifyAccessToken(accessToken: string): Promise<User | null> {
  try {
    const res = await fetch(`${CONSOLE_API}/auth/verify-access-token`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ token: accessToken }),
    });
    if (!res.ok) return null;
    const data = await res.json();
    return {
      email: data.email,
      name: data.name,
      role: data.role,
      user_id: data.user_id,
      is_intern: data.is_intern,
      token: data.token,
    };
  } catch {
    return null;
  }
}

// Replace YOUR_SERVICE with actual service name (e.g., brain_studio)
const STORAGE_PREFIX = "YOUR_SERVICE";

export function getStoredUser(): User | null {
  if (typeof window === "undefined") return null;
  const email = sessionStorage.getItem(`${STORAGE_PREFIX}_email`);
  const token = sessionStorage.getItem(`${STORAGE_PREFIX}_token`);
  if (!email || !token) return null;
  return {
    email,
    name: sessionStorage.getItem(`${STORAGE_PREFIX}_name`) || email.split("@")[0],
    role: sessionStorage.getItem(`${STORAGE_PREFIX}_role`) || "user",
    user_id: sessionStorage.getItem(`${STORAGE_PREFIX}_user_id`) || "",
    is_intern: sessionStorage.getItem(`${STORAGE_PREFIX}_is_intern`) === "true",
    token,
  };
}

export function storeUser(user: User): void {
  sessionStorage.setItem(`${STORAGE_PREFIX}_email`, user.email);
  sessionStorage.setItem(`${STORAGE_PREFIX}_name`, user.name);
  sessionStorage.setItem(`${STORAGE_PREFIX}_token`, user.token);
  sessionStorage.setItem(`${STORAGE_PREFIX}_user_id`, user.user_id);
  sessionStorage.setItem(`${STORAGE_PREFIX}_role`, user.role);
  sessionStorage.setItem(`${STORAGE_PREFIX}_is_intern`, String(user.is_intern));
}

export function clearUser(): void {
  sessionStorage.removeItem(`${STORAGE_PREFIX}_email`);
  sessionStorage.removeItem(`${STORAGE_PREFIX}_name`);
  sessionStorage.removeItem(`${STORAGE_PREFIX}_token`);
  sessionStorage.removeItem(`${STORAGE_PREFIX}_user_id`);
  sessionStorage.removeItem(`${STORAGE_PREFIX}_role`);
  sessionStorage.removeItem(`${STORAGE_PREFIX}_is_intern`);
}

export function redirectToConsole(): void {
  window.location.href = CONSOLE_URL;
}
```

### AppShell.tsx (Frontend)

```typescript
"use client";

import { useState, useEffect, ReactNode } from "react";
import { useSearchParams, useRouter } from "next/navigation";
import { verifyAccessToken, getStoredUser, storeUser, clearUser, redirectToConsole, User } from "@/lib/sso";
import Sidebar from "./Sidebar";

export default function AppShell({ children }: { children: ReactNode }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const searchParams = useSearchParams();
  const router = useRouter();

  useEffect(() => {
    const authenticate = async () => {
      // DEV mode bypass
      if (process.env.NEXT_PUBLIC_ENV === "DEV") {
        setUser({
          email: "dev@medina-partners.com",
          name: "Dev User",
          role: "admin",
          user_id: "dev",
          is_intern: false,
          token: "dev-token",
        });
        setLoading(false);
        return;
      }

      // Check existing session
      const stored = getStoredUser();
      if (stored) {
        setUser(stored);
        setLoading(false);
        return;
      }

      // Check URL for access token
      const accessToken = searchParams.get("access_token");
      if (accessToken) {
        const verified = await verifyAccessToken(accessToken);
        if (verified) {
          storeUser(verified);
          setUser(verified);
          // Clear token from URL
          const url = new URL(window.location.href);
          url.searchParams.delete("access_token");
          router.replace(url.pathname + url.search);
        } else {
          setError("Invalid or expired access token");
        }
        setLoading(false);
        return;
      }

      setError("Authentication required");
      setLoading(false);
    };

    authenticate();
  }, [searchParams, router]);

  const handleLogout = () => {
    clearUser();
    redirectToConsole();
  };

  if (loading) {
    return (
      <div className="min-h-screen flex items-center justify-center bg-primary">
        <div className="text-muted">Loading...</div>
      </div>
    );
  }

  if (error || !user) {
    return (
      <div className="min-h-screen flex flex-col items-center justify-center bg-primary">
        <div className="card text-center max-w-md">
          <h2 className="text-xl font-semibold mb-4">Authentication Required</h2>
          <p className="text-muted mb-6">{error || "Please log in through Console"}</p>
          <button onClick={redirectToConsole} className="btn-primary">
            Go to Console
          </button>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen flex bg-primary">
      <Sidebar user={user} onLogout={handleLogout} />
      <main className="flex-1 overflow-auto">{children}</main>
    </div>
  );
}
```

### auth.py (Backend)

```python
import os
import jwt
from fastapi import HTTPException, Depends, Request
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

JWT_SECRET = os.getenv("JWT_SECRET", "medina-jwt-secret-production-2026")
IS_DEV = os.getenv("ENV", "PROD") == "DEV"

security = HTTPBearer(auto_error=False)


async def get_current_user(
    request: Request,
    credentials: HTTPAuthorizationCredentials = Depends(security)
) -> dict:
    # DEV mode bypass
    if IS_DEV:
        user_id = request.headers.get("X-User-Id")
        if user_id:
            return {
                "user_id": user_id,
                "email": "dev@medina-partners.com",
                "role": "admin"
            }

    if not credentials:
        raise HTTPException(status_code=401, detail="Not authenticated")

    try:
        payload = jwt.decode(
            credentials.credentials,
            JWT_SECRET,
            algorithms=["HS256"]
        )
        return {
            "user_id": payload.get("sub") or payload.get("user_id"),
            "email": payload.get("email"),
            "name": payload.get("name"),
            "role": payload.get("role", "user"),
        }
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")
```

---

## Part 7: Port Assignments

Each service has assigned ports:

| Service | Frontend | Backend |
|---------|----------|---------|
| Console | 3002 | 8002 |
| Dashboard | 8501 | — |
| Brain Studio | 3001 | 8000 |
| HRP Studio | 3003 | 8003 |
| (Next available) | 3004 | 8004 |

---

## Part 8: Deployment

### Docker Commands

```bash
# Build
docker build -f Dockerfile.frontend -t service-frontend .
docker build -f Dockerfile.backend -t service-backend .

# Run locally
docker run -d -p 3004:3000 -e NEXT_PUBLIC_ENV=DEV service-frontend
docker run -d -p 8004:8000 -e ENV=DEV service-backend

# Production (bind to localhost, Nginx handles external)
docker run -d -p 127.0.0.1:3004:3000 -e NEXT_PUBLIC_ENV=PROD service-frontend
docker run -d -p 127.0.0.1:8004:8000 -e ENV=PROD -e JWT_SECRET="..." service-backend
```

### Nginx Configuration

Create `/etc/nginx/sites-available/service.medina-partners.com`:

```nginx
server {
    listen 443 ssl http2;
    server_name service.medina-partners.com;

    ssl_certificate /etc/letsencrypt/live/service.medina-partners.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/service.medina-partners.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:3004;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
    }

    location /api {
        proxy_pass http://127.0.0.1:8004;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### Adding to Console

1. Edit `/Users/chouwilliam/Medina/console/src/app/dashboard/page.tsx`
2. Add service to `SERVICES` array
3. Add service ID to `tokenServices` array
4. Update CORS in Console backend

---

## Part 9: Reference Implementations

When implementing features, look at these files:

| Feature | Reference File |
|---------|---------------|
| Collapsible Sidebar | `/Users/chouwilliam/Medina/brain-studio/src/components/Sidebar.tsx` |
| SSO Integration | `/Users/chouwilliam/Medina/hrp-studio/src/lib/sso.ts` |
| API Client | `/Users/chouwilliam/Medina/hrp-studio/src/lib/api.ts` |
| Design Tokens | `/Users/chouwilliam/Medina/brain-studio/src/app/globals.css` |
| Form Validation | `/Users/chouwilliam/Medina/hrp-studio/src/app/page.tsx` |
| Data Tables | `/Users/chouwilliam/Medina/hrp-studio/src/app/strategies/page.tsx` |
| Charts (Recharts) | `/Users/chouwilliam/Medina/brain-studio/src/components/FactorPerformanceChart.tsx` |
| MongoDB Service | `/Users/chouwilliam/Medina/hrp-studio/services/strategy.py` |
| JWT Auth Middleware | `/Users/chouwilliam/Medina/hrp-studio/api/auth.py` |

---

## Part 10: Development Workflow

### Local Development

1. Start Console (if testing SSO):
```bash
cd /Users/chouwilliam/Medina/console/backend
source venv/bin/activate
uvicorn main:app --reload --port 8001

# Another terminal
cd /Users/chouwilliam/Medina/console
npm run dev
```

2. Start your service:
```bash
# Backend
cd /your-service
source venv/bin/activate
ENV=DEV uvicorn api.main:app --reload --port 8004

# Frontend
cd /your-service
NEXT_PUBLIC_ENV=DEV npm run dev
```

In DEV mode, authentication is bypassed. You're automatically logged in as `dev@medina-partners.com`.

### Writing Development Notes

After completing significant work, write notes following the format in `/Users/chouwilliam/Medina/hub/prompts/00_base/02_how_to_write_note.md`. Save to `docs/01_note/yyyymmdd_description.md`.

---

## Part 11: Collaboration Principles

When helping William develop, follow the principles in `/Users/chouwilliam/Medina/hub/prompts/00_base/00_how_to_guide_develop.md`:

1. **Discuss before building** — Ask questions to understand the full picture before writing code.

2. **Present options** — When there are design decisions, explain trade-offs and let William choose.

3. **Build incrementally** — Write small pieces, test, verify, discuss, then continue.

4. **Self-review** — Check your own code for bugs, consistency, and over-engineering before presenting.

5. **Speak up** — If you see potential issues, say so early rather than after writing 200 lines.

6. **No over-engineering** — Do what's requested, do it well, then stop. No extra features, no premature abstractions.

---

## Appendix: Quick Checklist for New Service

- [ ] Use Next.js 15 + React 19 + TypeScript + Tailwind CSS 4
- [ ] Use FastAPI backend with PyMongo
- [ ] Copy `globals.css` design tokens from brain-studio
- [ ] Implement SSO with `sso.ts` and `AppShell.tsx`
- [ ] Add JWT validation in backend `auth.py`
- [ ] Configure CORS to allow Console and your service
- [ ] Use assigned ports (next available: 3004/8004)
- [ ] Create `Dockerfile.frontend` and `Dockerfile.backend`
- [ ] Create `deploy_local.sh` and `deploy_server.sh`
- [ ] Add to Console's SERVICES array and tokenServices
- [ ] Update Console backend CORS
- [ ] Add Nginx configuration
- [ ] Obtain SSL certificate with Certbot
- [ ] Test full SSO flow end-to-end
