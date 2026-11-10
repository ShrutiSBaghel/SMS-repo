# SMS — Employee and Salary Management

A full-stack application for managing departments, employees, and salary structures, with JWT login and salary reports grouped by department or country.

## Application demo

[Watch the application demo](docs/app-recording.mp4) (MP4, approximately 102 MB).

## Architecture

The Angular frontend calls the Spring Boot REST API, which stores application data and login accounts in PostgreSQL:

```text
Browser (localhost:4200) → Spring Boot API (localhost:8080) → PostgreSQL (localhost:5432)
```

## Repository structure

`backend/` and `frontend/` are Git submodules. Initialize both when cloning this repository.

```text
SMS-repo/
├── README.md
├── .gitmodules                       Submodule repository locations
├── backend/                          Java 21 / Spring Boot 4.1.2 API
│   ├── pom.xml                       Maven dependencies and build configuration
│   ├── mvnw, mvnw.cmd                Maven wrappers for Unix and Windows
│   ├── src/main/java/org/project/sms/
│   │   ├── SmsApplication.java        Application entry point
│   │   ├── config/                   Security, password encoder, and JWT configuration
│   │   ├── controller/               Login, health, department, employee, and report endpoints
│   │   ├── dto/                      Request/response objects and input validation
│   │   ├── entity/                   User, Department, Employee, and SalaryStructure entities
│   │   ├── repository/               Spring Data JPA database access and report queries
│   │   ├── security/                 JWT authentication filter
│   │   └── service/                  Business logic, authentication, and transactions
│   ├── src/main/resources/           Development and production application properties
│   ├── src/test/                     Unit and PostgreSQL integration tests
│   ├── scripts/seed.sql              Optional synthetic departments, employees, and salaries
│   └── Dockerfile.vercel, vercel.json Deployment configuration
└── frontend/                         Angular 21 / TypeScript application
    ├── package.json, package-lock.json Dependencies and npm scripts
    ├── angular.json                  Angular build, development server, and test configuration
    ├── public/                       Static assets
    ├── src/main.ts                   Browser entry point
    ├── src/styles.css                Global styles
    ├── src/environments/             API origin settings
    ├── src/app/
    │   ├── app.routes.ts             Application routes
    │   ├── api.config.ts             Shared API base URL provider
    │   ├── login/                    Login form, session service, guards, and token interceptor
    │   ├── dashboard-shell/, header/ Authenticated layout and navigation
    │   ├── dashboard/                Departments and employee counts
    │   ├── controller/               Shared department API service
    │   ├── employee/                 Employee list and pagination
    │   ├── create-employee-dialog/    Employee creation
    │   ├── edit-employee-dialog/      Employee editing
    │   ├── employee-form-dialog/     Shared form, validation, and save service
    │   ├── delete-employee-dialog/    Deletion confirmation
    │   ├── salary-structure-dialog/  Salary breakdown and CSV export
    │   ├── report/                   Report data and grouping controls
    │   └── salary-distribution-chart/ Chart.js chart and report table
    ├── scripts/                      Deployment build script and configuration tests
    └── vercel.json                   Static deployment and SPA routing configuration
```

Backend requests flow through controllers, services, and repositories; controllers return DTOs. The frontend uses standalone components, services, signals, reactive forms, and RxJS. Component tests live alongside the frontend source in `*.spec.ts` files.

## Start locally from scratch

The commands below use PowerShell and start from the repository root unless a step says otherwise. On macOS/Linux, use `./mvnw` instead of `.\mvnw.cmd` and `export NAME='value'` instead of `$env:NAME = 'value'`.

### 1. Install prerequisites

- Git.
- JDK 21, with `JAVA_HOME` pointing to that JDK and Java on `PATH`.
- Node.js 24.x and npm (the frontend declares npm 11.13.0).
- A running local PostgreSQL server and the `psql` command-line client on `PATH`. PostgreSQL 17 is used by the backend integration tests.
- Internet access for the first Maven and npm dependency downloads.

Maven and Angular CLI do not need separate global installations. Docker is needed only if you run the backend integration tests.

Check your tools:

```powershell
git --version
java -version
node --version
npm --version
psql --version
```

### 2. Clone the repository and initialize submodules

```powershell
git clone --recurse-submodules https://github.com/ShrutiSBaghel/SMS-repo.git
cd SMS-repo
```

If you already cloned the repository, run this from its root:

```powershell
git submodule update --init --recursive
```

### 3. Create the local database

With PostgreSQL running, create the database using your local administrator account:

```powershell
psql -h localhost -U postgres -d postgres -c 'CREATE DATABASE "SMS";'
```

Enter the PostgreSQL password when prompted. Skip database creation if `SMS` already exists. Hibernate creates or updates the development tables when the API starts; it does not create the database itself.

### 4. Configure and start the backend

In the first terminal:

```powershell
cd backend
.\mvnw.cmd -v

$env:DB_URL = 'jdbc:postgresql://localhost:5432/SMS'
$env:DB_USERNAME = 'postgres'
$env:DB_PASSWORD = 'replace-with-your-local-postgres-password'
$env:PORT = '8080'
$env:CORS_ALLOWED_ORIGINS = 'http://localhost:4200'

# Generate a Base64 signing key containing 32 random bytes.
$jwtBytes = New-Object byte[] 32
$jwtRng = [System.Security.Cryptography.RandomNumberGenerator]::Create()
$jwtRng.GetBytes($jwtBytes)
$jwtRng.Dispose()
$env:JWT_SECRET = [Convert]::ToBase64String($jwtBytes)

.\mvnw.cmd spring-boot:run
```

Confirm Maven reports Java 21. Keep this terminal open. These environment variables apply to this terminal's process and its children; set them again in a new terminal. Keep the same private JWT key across restarts if existing tokens should remain valid.

Use the default development profile for this local setup. `SPRING_PROFILES_ACTIVE=prod` switches to the separate Supabase configuration. The checked-in `.env.example` files describe deployment settings; copying them to `.env` does not automatically load them into Spring Boot or the frontend build script.

In a second terminal, check that the API and database are reachable:

```powershell
Invoke-RestMethod http://localhost:8080/health
```

Expect a response with `status` equal to `UP` before continuing.

### 5. Create your first application login

There is no signup endpoint or automatically created account. After the API has created its tables, connect to the local database from the second terminal:

```powershell
psql -h localhost -U postgres -d SMS
```

Run the following SQL in `psql`, replacing the example password before executing it. This uses PostgreSQL's `pgcrypto` extension to store a BCrypt hash compatible with the backend password encoder:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

INSERT INTO public.users (name, password)
VALUES ('admin', crypt('replace-with-your-local-login-password', gen_salt('bf', 12)));
```

Create this user once and use those credentials in the UI. The database account (`postgres`) and this application account (`admin`) serve different purposes. Supabase Auth accounts also do not create application users. Creating the extension requires a database account with the necessary privileges; the standard local `postgres` administrator can perform this setup.

### 6. Add departments or load sample data

Employees need an existing department, and the UI has no department creation page. Choose one of these options before creating employees.

For a small, empty department, run this in the same `psql` session:

```sql
INSERT INTO public.department (name) VALUES ('Engineering');
```

Then exit `psql` with `\q`.

Alternatively, skip that department insert and load the supplied sample data from a PowerShell terminal at the repository root:

```powershell
psql -h localhost -U postgres -d SMS -v ON_ERROR_STOP=1 -f backend/scripts/seed.sql
```

The script creates 7 departments, 10,000 employees, and their salary structures. It requires empty `department`, `employee`, and `salary_structure` tables and refuses to run if they contain data. It does not create login users, so step 5 is still required.

### 7. Install and start the frontend

In a separate terminal at the repository root:

```powershell
cd frontend
npm ci
npm start
```

Open [http://localhost:4200/](http://localhost:4200/) and sign in with the account from step 5. Keep both application terminals and PostgreSQL running. Stop each development server with `Ctrl+C`.

The local API origin is configured in `frontend/src/environments/environment.ts` as `http://localhost:8080`. If you change the API port or host, update that value and restart the frontend. Services use absolute API URLs, so backend `CORS_ALLOWED_ORIGINS` must match the frontend origin exactly.

## Using the application

| Route | Purpose |
| --- | --- |
| `/` | Sign in |
| `/dashboard` | View departments and employee counts |
| `/employee/:departmentId` | List, create, edit, and delete employees; view salary breakdowns and export CSV |
| `/report` | View average salaries grouped by department or country |

The frontend sends the JWT as a bearer token for protected API requests. Sessions are held in memory, so refreshing the browser requires signing in again. Tokens expire after 30 minutes by default.

## Tests and builds

Run backend commands from `backend/` with JDK 21. The full test suite uses Testcontainers and requires a running Docker daemon:

```powershell
.\mvnw.cmd clean test
.\mvnw.cmd package
```

`package` also runs tests and creates `backend/target/SMS-0.0.1-SNAPSHOT.jar`. To run it from `backend/`, with the same backend environment variables set:

```powershell
java -jar target/SMS-0.0.1-SNAPSHOT.jar
```

Run frontend commands from `frontend/`:

```powershell
npm test -- --watch=false
npm run build
```

Frontend tests use Vitest/jsdom and mock HTTP responses; they do not need the API or database. The ordinary frontend build writes to `frontend/dist/sms-ui/` and uses the local API origin. Deployment builds use `npm run build:vercel` with an HTTPS API origin supplied through `SMS_API_URL`.

## Troubleshooting

| Problem | Check |
| --- | --- |
| Empty `backend/` or `frontend/` directory | Run `git submodule update --init --recursive`. |
| Java compilation or Maven startup error | Check `JAVA_HOME` and `.\mvnw.cmd -v`; use JDK 21. |
| Database connection refused or authentication failed | Start PostgreSQL, confirm `SMS` exists, and verify `DB_URL`, `DB_USERNAME`, and `DB_PASSWORD`. |
| Missing or invalid `JWT_SECRET` | Set the environment variable in the terminal that starts the API; it must decode from Base64 to at least 32 bytes. |
| Login returns HTTP 401 | Use the `users` account created in step 5 and confirm its password is BCrypt-hashed. |
| Dashboard has no departments | Insert a department or load the sample data from step 6, then reload the dashboard. |
| Browser cannot reach the API or reports CORS errors | Check `/health`, the frontend API origin, and the backend's exact allowed frontend origin. |
| Backend integration tests cannot start | Start Docker and allow Testcontainers to download its PostgreSQL image. |
