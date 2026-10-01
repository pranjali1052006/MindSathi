# MindSaathi

MindSaathi is a hackathon prototype for confidential student wellbeing support. It demonstrates a student check-in and support journey, human counselor review, and privacy-preserving campus analytics. It is a prototype, not a clinical tool or emergency service.

## Run the interactive demo

Requirements: Node.js 20.19+ or 22.12+ and npm.

```sh
npm install
npm run dev
```

Open the local URL Vite prints (normally `http://localhost:5173`). The UI requires email and password sign-in before entering a portal. When the Spring API is running, sign-in calls `/api/auth/login` and stores its signed JWT for the browser tab. Vite proxies `/api` to `localhost:8080`.

Without the API, Vite development mode accepts these explicitly seeded fictional demo credentials and locally registered demo accounts. This offline shortcut is for demonstrations only; it is not a security boundary and is disabled in production builds. Do not enter real student data or reuse real passwords:

- `student@demo.mindsathi.local` / `mindsathi-demo`
- `counselor@demo.mindsathi.local` / `mindsathi-demo` (Dr. Aanya Sharma)
- `counselor.rohan@demo.mindsathi.local` / `mindsathi-demo` (Dr. Rohan Kapoor)
- `admin@demo.mindsathi.local` / `mindsathi-demo`

Both counselor accounts use the same counselor workspace and review queue; the signed-in counselor's name is shown in the dashboard. The API seeder creates both counselor users on a fresh database. If a database was already initialized with the earlier prototype seeder, reset the local demo database volume or add the Rohan account through your normal account provisioning process before API login.

Choose **Fill** beside an account, then press **Sign in**. There is no role switch inside the portals. Sign out clears the tab-scoped session.

- Student: private check-in, mood tracking, resources, and counselor appointment request.
- Counselor: anonymous, risk-prioritized case queue, human disposition, appointments, and audit log.
- Admin: campus-wide charts using fictional aggregates only; the interface states and the API enforces `k >= 10` group suppression.

Each student sees a stable, copyable counselor ID in the student portal. The counselor case queue uses that same ID and does not reveal the student's name. In the offline demo, the seeded student uses `MS-3187`; newly created students receive their own saved ID.

The student registration form calls the Spring API when available. In offline development mode, it creates a browser-only demo account so it can be used again after signing out. Each email gets a separate local mood, assessment, booking, and resource-reflection space. The offline demo stores its fictional login credentials in this browser and is not suitable for real accounts or sensitive data.

## Complete service stack

Requirements: Docker Engine and Docker Compose v2.

```sh
copy .env.example .env
```

Replace `DATABASE_PASSWORD`, `JWT_SECRET` (at least 32 random bytes), and `DOMAIN` in `.env` before deployment. Then run:

```sh
docker compose up --build
```

Open `http://localhost:4173`. Compose starts PostgreSQL 16, the Spring Boot API, the FastAPI scoring service, and the production Vite build behind Nginx. The API schema is applied at startup and fictional demo accounts and a sufficiently large aggregate cohort are seeded once.

To enable HTTPS through Caddy, configure a public DNS name in `DOMAIN`, point it at the host, then run:

```sh
docker compose --profile https up --build
```

Caddy obtains and renews certificates for a public domain. For local `DOMAIN=localhost`, Caddy uses a locally trusted CA certificate; browsers will require trusting that CA. Keep the API, database, and risk service on the private Compose network in deployment.

For a native backend-only run, start PostgreSQL and the AI service first, then from `backend/` run `mvn spring-boot:run`. The risk service can be run with Python 3.11+ after installing `ai/requirements.txt`, using `uvicorn main:app --app-dir ai --port 8001` from the project root.

## Demo identities

The native API seeds these accounts with the password `mindsathi-demo`:

- `student@demo.mindsathi.local`
- `counselor@demo.mindsathi.local`
- `admin@demo.mindsathi.local`

The same identities are seeded by the API with BCrypt-hashed passwords. API login returns a signed, expiring JWT containing the role claim; protected endpoints use Spring Security role checks. The dev-only offline shortcut exists solely so the UI walkthrough is possible without PostgreSQL/Docker.

## Risk scoring and human review

`ai/main.py` calculates a transparent prototype signal from weighted response totals (PHQ-9 40%, GAD-7 35%, PSS-4 25%), optionally adjusts for the direction of recent mood check-ins, and returns its components and contributing factors. A non-zero response to the sensitive final PHQ-9 item requests timely human review and raises the signal floor. Bands are prototype prioritization rules, not validated clinical thresholds; the result is not a diagnosis and does not decide a counselor's action.

Counselor decisions are stored separately in `counselor_reviews` and auditable in `audit_logs`. Admin analytics return aggregates only; queries use `HAVING COUNT(DISTINCT ...) >= 10`, and do not return student identifiers or individual assessment results.

## Security and privacy notes

- Passwords for native API accounts are BCrypt-hashed; JWTs are signed, expire, and carry a role claim enforced by Spring Security.
- Configure strong secrets, TLS, backups, retention, and institution-approved crisis contacts before any real deployment.
- The prototype has fictional seed data. Do not enter real student wellbeing or identity information.
- The browser-only demo persists student check-ins in local storage and is not a substitute for an encrypted production client.
