# Karpagam Earthmovers

React, TypeScript, Vite, Tailwind CSS, and an Express/Mongoose REST API for the Karpagam Earthmovers website.

## Requirements

- Windows 10/11
- Node.js 22 or newer
- npm (included with Node.js)
- MongoDB Community Server locally, or a MongoDB Atlas cluster for persistent data

## Windows Setup

Open PowerShell in the project folder:

```powershell
npm install
if (-not (Test-Path .env)) { Copy-Item .env.example .env }
```

Edit `.env` and set the security values. Generate a signing secret with:

```powershell
node -e "console.log(require('node:crypto').randomBytes(48).toString('hex'))"
```

Use the output as `JWT_SECRET`. Set a unique `ADMIN_USERNAME`, `ADMIN_EMAIL`, and strong `ADMIN_PASSWORD` (at least 12 characters). The first server start seeds that administrator only when all four values are configured. Passwords are hashed with bcrypt. Never commit `.env` or place server secrets in `VITE_*` variables.

### MongoDB

For a local database, install MongoDB Community Server, start the MongoDB Windows service, and set:

```dotenv
MONGODB_URI=mongodb://127.0.0.1:27017/karpagam_earthmovers
```

For Atlas, create a cluster and database user, allow the application host in Atlas Network Access, and set the provided connection string in `MONGODB_URI`. Keep the username/password out of source control. MongoDB collections and indexes are created by Mongoose; existing service and machinery content is seeded only into empty collections.

If `MONGODB_URI` is empty or unreachable in development, the backend uses `data/dev-store.json` (change with `DEV_STORE_PATH`). This is a local development convenience, not durable production persistence, and it is not suitable for multiple server instances. Production refuses to start without MongoDB and strong explicit admin/JWT configuration.

### Optional Email

Set `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `SMTP_FROM`, and `NOTIFICATION_EMAIL` to enable server-side email notifications. Leave SMTP settings blank to store enquiries without sending email. SMTP failure does not undo a successfully stored enquiry. Credentials are only read by the backend.

Set `CORS_ORIGIN` to a comma-separated list of exact frontend origins, for example `https://www.example.com`. Local development origins are allowed in development. Set `WHATSAPP_NUMBER` as digits including country code, without `+`.

## Run

Run frontend and backend together:

```powershell
npm run dev:all
```

The Vite site uses port 8443 by default and proxies `/api` to the backend on port 5000. The backend API is at `http://localhost:5000/api`; health is `http://localhost:5000/api/health`. In this workspace, the Figma preview server may already occupy the frontend port; use its preview panel or change the `PORT` environment variable before starting Vite.

Run separately when preferred:

```powershell
npm run dev:server
npm run dev
```

## API

Public endpoints:

- `GET /api/health` reports service health and whether the database is persistent MongoDB or development fallback.
- `GET /api/config` returns public company contact settings and service configuration status.
- `GET /api/services` lists active services; `GET /api/services/:id` retrieves a service.
- `GET /api/machinery` lists available machinery; `GET /api/machinery/:id` retrieves machinery.
- `POST /api/quotations` validates and stores an enquiry, returns a unique quotation reference, and rejects matching phone/service submissions within five minutes with `409`.
- `GET /api/quotations/:referenceNumber` looks up the public status for a quotation reference.
- `POST /api/contact` validates and stores a general contact message.

Admin endpoints require `Authorization: Bearer <token>` after `POST /api/admin/login`:

- `GET /api/admin/me` and `GET /api/admin/stats`
- `GET /api/admin/quotations?status=Pending&search=...&page=1&limit=10`
- `GET /api/admin/quotations/:id`, `PATCH /api/admin/quotations/:id`, `DELETE /api/admin/quotations/:id`
- `GET /api/admin/messages?status=Unread&search=...&page=1&limit=10`
- `PATCH /api/admin/messages/:id`, `DELETE /api/admin/messages/:id`
- `POST /api/admin/machinery`, `PUT /api/admin/machinery/:id`, `DELETE /api/admin/machinery/:id`
- `POST /api/admin/services`, `PUT /api/admin/services/:id`, `DELETE /api/admin/services/:id`

Quotation statuses are `Pending`, `Contacted`, `Quoted`, `Approved`, and `Rejected`. Catalogue mutations and enquiry management are protected. This repository provides the protected administration API, not a separate admin dashboard UI.

## Test and Build

```powershell
npm run typecheck
npm run test:api
npm run build
npm run build:server
```

The API tests use the local development store; they remove the enquiries created during the run. To confirm persistent database operation, configure MongoDB and check `GET /api/health` for `database.mode: "mongodb"` and `database.connected: true`.

## Deployment

1. Build the frontend with `npm run build` and the API with `npm run build:server`.
2. Deploy the Vite `dist/` output to a static host or serve it from the chosen web server.
3. Deploy the Node API from `server/dist/` on a persistent Node.js host. Configure `NODE_ENV=production`, `MONGODB_URI`, a random `JWT_SECRET` of at least 32 characters, all `ADMIN_*` variables, exact `CORS_ORIGIN`, and optional SMTP settings through the hosting provider's secret manager.
4. Set the frontend's `VITE_API_BASE_URL` to the public API prefix (for example `https://api.example.com/api`) at build time, or configure the production web server to proxy `/api` to the Node service. Do not expose database, SMTP, admin-password, or JWT-secret values to Vite.
5. Use HTTPS, restrict MongoDB network access, and use a durable MongoDB backup policy. The JSON fallback is disabled in production.

The API server listens on `PORT` (default `5000`). There is no admin dashboard UI in the current frontend; admin workflows use the authenticated endpoints above.
