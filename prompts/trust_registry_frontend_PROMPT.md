# 🤖 AI SYSTEM PROMPT: REPO GENERATION INSTRUCTIONS

## 1. MACRO CONTEXT: THE BIG PICTURE
**Project:** TrustVault (Makeathon: Portable Proof with VC and VP)
**Goal:** We are building the "Digital Notary" ecosystem. 
**The Ecosystem:** We are building the frontend for the **Trust Registry**. This is a global "Governance Dashboard" for consortium regulators. It allows an administrator to visually monitor all active "Trust Anchors" (Universities/Banks) in the decentralized network. It provides instant toggles to Suspend or Revoke an Issuer, which immediately invalidates their cryptographically signed credentials across the entire global ecosystem.

## 2. REPOSITORY ROLE: TRUST REGISTRY FRONTEND
**Function:** An Angular 18+ web application. A single-page dashboard using Tailwind CSS for an enterprise-grade UI. It fetches and mutates data directly from the Trust Registry Java Backend API.

## 3. TECH STACK & EXACT DEPENDENCIES
*   **Framework:** Angular 18+ (Standalone Components).
*   **Styling:** Tailwind CSS 3.4.19, PostCSS 8.5.9.
*   **Architecture:** `provideHttpClient` in `app.config.ts`, `ReactiveFormsModule` for onboarding new Issuers.

## 4. ENVIRONMENT VARIABLES (`src/environments/environment.ts`)
*   `registryApi`: `https://trust-registry-backend-26472113294.us-central1.run.app/api/v1/registry`

## 5. CORE COMPONENTS & UI SPECIFICATIONS (`src/app/app.ts`)
The entire application is a single, dense `App` component that acts as the global command center.

### A. The Dashboard Layout (Tailwind)
*   **Navbar:** `bg-indigo-900 text-white`. Displays "Trust Registry Governance" and a "Global Admin" badge.
*   **Header:** Title "Ecosystem Authorities" and a primary blue button: `[Onboard New Issuer]`.
*   **Network Stats Cards:** Three metric cards displaying real-time data using Angular getters:
    1.  `Active Issuers`: Renders `activeCount` (`this.issuers.filter(i => i.status === 'ACTIVE').length`).
    2.  `Suspended / Revoked`: Renders `suspendedCount` (`this.issuers.filter(i => i.status !== 'ACTIVE').length`).
    3.  `Network Protocol`: Hardcoded to "OID4VC".

### B. The Ledger Table
*   Displays a list of all registered Issuers fetched from `GET /issuers`.
*   Columns:
    *   **Institution Name:** Displays the name and an avatar circle with the first letter of the name.
    *   **DID (Decentralized ID):** Renders the `did` string in a monospaced font box (`bg-gray-100 font-mono`).
    *   **Public Key Fragment:** Only displays the last 8 characters of the `publicKey` (e.g., `...678912`).
    *   **Trust Status:** Dynamic Tailwind badge based on `status`:
        *   `ACTIVE`: Green (`bg-green-50 text-green-800`).
        *   `REVOKED`: Red (`bg-red-50 text-red-800`).
        *   `SUSPENDED`: Yellow (`bg-amber-50 text-amber-800`).
    *   **Governance (Actions):** 
        *   If `ACTIVE`, show [Suspend] and [Revoke] buttons.
        *   If `SUSPENDED`, show [Activate] and [Revoke].
        *   If `REVOKED`, show [Activate].
        *   Buttons map to `updateStatus(id, status)` -> `PUT /issuers/{id}/status`.

### C. The Onboarding Modal (Reactive Form)
*   Triggered by "Onboard New Issuer". Displays a modal overlay (`bg-gray-900 bg-opacity-75 backdrop-blur-sm`).
*   **Form Controls:**
    1.  `name`: Required.
    2.  `did`: Required (Placeholder: `did:web:bank.com`).
    3.  `publicKey`: Required. Rendered as a `<textarea>`.
*   **Validation UI:** 
    *   If `isInvalid('fieldName')` is true (invalid AND dirty/touched), apply red border ring (`border-red-300 ring-1 ring-red-300`).
    *   Display explicit red `<p>` error messages below the field.
*   **Submit Logic:** 
    *   If invalid, `markAllAsTouched()` and return.
    *   Else, set `isSaving = true`, `POST /issuers`, `fetchIssuers()`, close modal, reset form.

## 6. DEPLOYMENT (`Dockerfile` & `nginx.conf`)
*   **Stage 1:** `node:20-alpine`, `npm run build -- --configuration production`.
*   **Stage 2:** `nginx:alpine`, copies `/dist/trustvault-registry-frontend/browser` to `/usr/share/nginx/html`.
*   **nginx.conf:** Must use `try_files $uri $uri/ /index.html;` for Angular routing compatibility.
