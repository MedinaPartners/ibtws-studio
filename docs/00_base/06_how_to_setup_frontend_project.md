# Frontend Project Setup Guide

## Purpose and Background

This guide documents the standard conventions and requirements for setting up a new frontend project in the Medina Partners ecosystem. When William asks you to create a new web application with a frontend interface, you should follow these conventions to ensure consistency across all Medina projects. The design language, deployment patterns, and tooling choices have been standardized based on existing projects like Brain Studio and HRP Studio.

All Medina frontend projects share a common visual identity: a dark theme with the Medina blue accent color (#0077BB), consistent typography using Outfit for body text and Source Serif 4 for headings, and a unified component styling system. This consistency makes it easier for users to navigate between different tools in the ecosystem.

## Technology Stack

The standard frontend stack for Medina projects consists of Next.js 15 with React 19, TypeScript, and Tailwind CSS 4. For charting and data visualization, Recharts is the preferred library. Icons come from lucide-react. This combination provides a modern, performant foundation that works well with the FastAPI backends used throughout the Medina ecosystem.

When a project requires a Python backend, the backend uses FastAPI with uvicorn, and the Python dependencies are managed in a virtual environment. The standard approach is to use venv rather than conda or other environment managers, as this keeps the setup simple and consistent across development and production environments.

## Favicon Setup

Every Medina frontend project must include the Medina Partners favicon. The logo file is located at the project root in a logo folder, specifically at `logo/medinapartners_logo_vertical.jpg`. This file should be copied to the public folder as `public/favicon.jpg`, and the Next.js layout.tsx metadata should reference it.

In the layout.tsx file, the metadata export should include an icons property that points to the favicon:

```tsx
export const metadata: Metadata = {
  title: "Project Name",
  description: "Project description",
  icons: {
    icon: "/favicon.jpg",
  },
};
```

The Dockerfile for the frontend must also be configured to copy the public folder contents to the production image. In Next.js standalone builds, the public folder is not automatically included, so you need an explicit COPY instruction:

```dockerfile
COPY --from=builder /app/public ./public
```

## Design Language

The visual design must match Brain Studio and other Medina projects. The globals.css file should import Tailwind CSS and define CSS custom properties (design tokens) that establish the color palette and component styles. The core design tokens are:

Background colors use pure black (#000000) as the primary background, with progressively lighter shades for secondary surfaces (#0a0a0a), tertiary elements (#111111), cards (#1a1a1a), and the sidebar (#0d0d0d). The accent color is Medina blue (#0077BB) with a lighter variant (#50AEE4) and a hover state (#0099DD).

Text colors follow a hierarchy: primary text is white (#FFFFFF), secondary text is medium gray (#888888), and muted text is darker gray (#666666). Border colors are subtle dark grays (#222222 for default, #333333 for hover states).

Status colors are consistent across all projects: green (#22c55e) for success, yellow (#eab308) for warnings, and red (#ef4444) for errors or danger states.

Typography uses two font families loaded from Google Fonts. Body text uses Outfit with weights 300, 400, 500, and 600. Headings use Source Serif 4 with weights 400 and 500. The fonts should be loaded in the layout.tsx head section via preconnect links to fonts.googleapis.com and fonts.gstatic.com.

The globals.css file should define utility classes for buttons (.btn-primary), cards (.card), inputs (.input), badges (.badge, .badge-success, .badge-warning, .badge-danger), and table rows (.table-header, .table-row). These classes ensure consistent styling throughout the application without requiring inline styles or repeated Tailwind class combinations.

## Project Structure

A typical Medina frontend project with a Python backend follows this structure:

```
project-name/
├── api/                    # FastAPI backend
│   ├── __init__.py
│   ├── main.py            # FastAPI app entry point
│   ├── auth.py            # Authentication (Console SSO)
│   ├── schemas.py         # Pydantic models
│   └── routes/            # API route modules
├── db/                     # Database connectors
│   ├── __init__.py
│   ├── connector.py       # MongoDB connection
│   └── setup.py           # Collection/index setup
├── services/              # Business logic
├── engine/                # Core algorithms (if applicable)
├── test/                  # Python tests
├── src/                   # Next.js frontend
│   ├── app/              # App router pages
│   │   ├── layout.tsx    # Root layout with fonts and metadata
│   │   ├── page.tsx      # Home page
│   │   └── globals.css   # Design tokens and utility classes
│   ├── components/       # React components
│   └── lib/              # Utilities (API client, SSO helpers)
├── public/               # Static assets
│   └── favicon.jpg       # Medina Partners logo
├── logo/                 # Source logo files
├── docs/                 # Documentation
│   ├── 00_base/         # Base prompts and guides
│   ├── 01_major/        # Schema and architecture docs
│   └── 02_note/         # Development session notes
├── Dockerfile.frontend   # Next.js Docker build
├── Dockerfile.backend    # FastAPI Docker build
├── deploy_local.sh       # Local development script
├── deploy_server.sh      # Production deployment script
├── package.json          # Node.js dependencies
├── requirements.txt      # Python dependencies
├── tsconfig.json         # TypeScript config
└── .gitignore           # Git ignore rules
```

## Gitignore Configuration

When creating a new repository, GitHub's default Node.js gitignore template is typically selected. However, since Medina projects often include Python backends, the gitignore must be extended to include Python-specific patterns. The following Python patterns should be added to the gitignore:

Python bytecode and cache directories must be ignored: `__pycache__/`, `*.py[cod]`, `*$py.class`. Distribution and packaging artifacts should also be excluded: `build/`, `dist/`, `*.egg-info/`, `.eggs/`.

Virtual environment directories are critical to exclude: `.venv`, `venv/`, `env/`, `ENV/`. These can be large and should never be committed to version control.

Test coverage directories and caches should be ignored: `.pytest_cache/`, `htmlcov/`, `.coverage`, `.tox/`, `.nox/`.

Environment files containing secrets must be ignored: `.env`, `.env.*` (except `.env.example` if provided as a template).

IDE and editor directories like `.idea/` (PyCharm), `.vscode/` settings (though .vscode can optionally be included for shared settings), and `.mypy_cache/` should be excluded.

## Virtual Environment Setup

Python backends should use venv for environment isolation. The standard location is a `venv` folder at the project root. To set up the environment:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

When running the backend locally for development, always activate the virtual environment first. The uvicorn command for local development is typically:

```bash
source venv/bin/activate
uvicorn api.main:app --reload --port 8001
```

## Deploy Scripts

Every Medina frontend project needs two deployment scripts: one for local development and one for production server deployment.

The deploy_local.sh script handles local Docker builds and runs for testing the containerized application before pushing to production. It should accept an environment parameter (dev or prod) to select the appropriate configuration.

The deploy_server.sh script handles production deployment. It follows a consistent pattern across all Medina projects: stop and remove old containers, remove old images, build new images (backend first, then frontend), run containers with appropriate port bindings and environment variables, clean up Docker resources, and display the final status.

For projects that use different databases in development versus production, the backend container must receive an environment variable indicating which database to use. For example, HRP Studio uses `HRP_STUDIO_ENV=prod` to select the production MongoDB database.

The deploy script should bind containers to localhost (127.0.0.1) rather than 0.0.0.0, as Nginx handles external traffic with SSL termination. Port assignments should avoid conflicts with other Medina services:

```
Console: 3002 (frontend), 8002 (backend)
HRP Studio: 3003 (frontend), 8003 (backend)
Brain Studio: 3001 (frontend), 8000 (backend)
Dashboard: 8080 (Streamlit)
```

## Hub Makefile Integration

After creating a new frontend project, the Hub Makefile at `/Users/chouwilliam/Medina/hub/Makefile` should be updated to include commands for the new project. The standard pattern includes a local development command and a deploy command.

The local command runs the Next.js development server directly (not containerized):

```makefile
PROJECT_DIR := $(MEDINA_ROOT)/project-name

project-local:
	@echo "啟動 Project Name (本地 port XXXX)..."
	@cd $(PROJECT_DIR) && npm run dev
```

The deploy command runs the deploy_server.sh script:

```makefile
project-deploy:
	@echo "部署 Project Name 到伺服器..."
	@cd $(PROJECT_DIR) && ./deploy_server.sh
```

If the project should be deployed alongside other services (like how HRP Studio deploys with Console), add it to the appropriate dependency chain in the Makefile.

## Console SSO Portal

Console is the centralized Single Sign-On (SSO) portal for the entire Medina Partners ecosystem. It is located at `/Users/chouwilliam/Medina/console` and is deployed to `console.medina-partners.com`. All other frontend applications authenticate through Console, making it the entry point for users accessing any Medina service.

### Console Architecture

Console uses the same technology stack as other Medina frontends: Next.js for the frontend and FastAPI for the backend. However, unlike other services that have separate frontend and backend containers, Console runs both in a single container for simplicity. The deployment ports are:

- Frontend: 127.0.0.1:3002 (internal) → console.medina-partners.com (Nginx)
- Backend: 127.0.0.1:8002 (internal) → console.medina-partners.com/api (Nginx)

### Authentication Flow

The Console login process has three stages:

1. **Email Stage**: User enters their email address. Console checks if the user exists and shows the password field.

2. **Password Stage**: User enters their password. Console verifies the password against the stored hash.

3. **TOTP Stage**: If 2FA is enabled, user enters their 6-digit TOTP code from their authenticator app.

After successful authentication, Console creates a session cookie with domain scope `.medina-partners.com`. This cookie is automatically sent with requests to all Medina subdomains (dashboard.medina-partners.com, brain.medina-partners.com, hrp.medina-partners.com, etc.).

### Access Token Flow for Services

When a user clicks on a service link from Console (e.g., clicking "Brain Studio" or "HRP Studio"), Console doesn't just redirect. Instead:

1. Console generates a short-lived access token (valid for ~30 seconds)
2. Console redirects to the service URL with the token as a query parameter: `https://hrp.medina-partners.com?access_token=xxx`
3. The service's frontend receives the token and calls Console's API to verify it
4. Console returns the user information if the token is valid
5. The service stores user info in sessionStorage and clears the token from the URL

This access token mechanism ensures that even if someone bookmarks a URL with a token, it expires quickly and cannot be reused.

### Integrating SSO in a New Service

To integrate SSO in a new Medina frontend:

1. **Create the SSO helper** at `src/lib/sso.ts`:

```typescript
const CONSOLE_API = process.env.NODE_ENV === 'production'
  ? 'https://console.medina-partners.com/api'
  : 'http://localhost:8001/api';

export interface User {
  id: string;
  email: string;
  name: string;
  role: string;
}

export async function verifyAccessToken(token: string): Promise<User | null> {
  try {
    const response = await fetch(`${CONSOLE_API}/auth/verify-access-token`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ access_token: token }),
    });
    if (!response.ok) return null;
    return await response.json();
  } catch {
    return null;
  }
}

export function getStoredUser(): User | null {
  const stored = sessionStorage.getItem('user');
  return stored ? JSON.parse(stored) : null;
}

export function storeUser(user: User): void {
  sessionStorage.setItem('user', JSON.stringify(user));
}

export function clearUser(): void {
  sessionStorage.removeItem('user');
}

export function redirectToConsole(): void {
  window.location.href = process.env.NODE_ENV === 'production'
    ? 'https://console.medina-partners.com'
    : 'http://localhost:3000';
}
```

2. **Handle authentication in AppShell or the root page**. Check for access_token in URL params, verify it, store user info, and clear the URL.

3. **Show appropriate UI** when authentication fails (typically a message with a link back to Console).

### Console Deployment Order

When deploying the full Medina ecosystem, Console must be deployed first because other services depend on its authentication API. The Hub Makefile's `console-deploy` target handles this:

```makefile
console-deploy: console-deploy-self dashboard-deploy brain-deploy hrp-deploy
```

This ensures Console is deployed before Dashboard, Brain Studio, and HRP Studio.

## Communication Language

When discussing this setup with William, always communicate in Traditional Chinese (繁體中文). However, code, configuration files, and technical documentation should be written in English, as this is the standard for the codebase.

## After Reading This Guide

When William asks you to set up a new frontend project, start by discussing the project's purpose and requirements. Clarify what features the project needs, whether it requires a Python backend, what port numbers to use, and how it fits into the Medina ecosystem. Then systematically work through the setup: create the project structure, set up the design system, configure the favicon, create deploy scripts, and update the Hub Makefile.
