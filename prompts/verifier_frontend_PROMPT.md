# 🤖 AI SYSTEM PROMPT: REPO GENERATION INSTRUCTIONS

## 1. MACRO CONTEXT: THE BIG PICTURE
**Project:** TrustVault (Makeathon: Portable Proof with VC and VP)
**Goal:** This repository implements the **Verifier Frontend**. In the "Triangle of Trust", this represents an Employer (e.g., TechCorp) verifying a job applicant's degree.
**The Flow:** The HR rep clicks "Request Proof". A QR code is generated. The applicant scans it with their mobile wallet. The backend verifies the cryptographic signature and Trust Registry status, then instantly pushes the verified data (Name, Degree, etc.) to this frontend via WebSockets, while mathematically proving that any hidden fields (like GPA) remain hidden.

## 2. REPOSITORY ROLE: VERIFIER FRONTEND
**Function:** An Angular 18+ web application. A single-page HR Dashboard that orchestrates the **OID4VP (Presentation)** UI flow. It renders the `openid-vp://` QR code and uses an RxJS WebSocket to listen for the backend's real-time verification result.

## 3. TECH STACK & EXACT DEPENDENCIES
*   **Framework:** Angular 18+ (Standalone Components).
*   **Styling:** Tailwind CSS 3.4.19, PostCSS 8.5.9.
*   **Core Libraries:** `angularx-qrcode:18.0.0` (Must be imported as `QRCodeModule`, not a standalone component due to ng-compiler restrictions in v18).
*   **Architecture:** `provideHttpClient` in `app.config.ts`.

## 4. ENVIRONMENT VARIABLES (`src/environments/environment.ts`)
*   `verifierApi`: `https://verifier-backend-26472113294.us-central1.run.app/api/v1/verifier`
*   `wsUrl`: `wss://verifier-backend-26472113294.us-central1.run.app/ws/verification`

## 5. CORE COMPONENTS & UI SPECIFICATIONS (`src/app/app.ts`)
The entire application is a single `App` component that acts as the "TechCorp HR Dashboard".

### A. The State Variables
*   `requestActive` (boolean)
*   `presentationUrl` (string): The OID4VP deep link payload.
*   `sessionId` (string): UUID for WebSocket mapping.
*   `verifiedData` (any | null): The JSON payload received from the WebSocket.
*   `error` (string | null)
*   `wsSubscription` (rxjs Subscription)

### B. The Logic Flow
1.  **`generateRequest()`:** 
    *   Fires `POST /request`.
    *   Receives `{presentation_url, session_id}`.
    *   Sets `requestActive = true`.
    *   Calls `listenForVerification(session_id)`.
2.  **`listenForVerification(sessionId)`:**
    *   Uses `rxjs/webSocket` to connect to `${wsUrl}?sessionId=${sessionId}`.
    *   If message `.status === 'VERIFIED'`, set `verifiedData`, close QR code UI, disconnect WS.
    *   If message `.status === 'FAILED'`, set `error` string, display error UI, disconnect WS.
3.  **`getClaimKeys()`:**
    *   Helper function for the template.
    *   Returns `Object.keys(this.verifiedData.claims)` BUT filters out standard JWT claims (`'sub', 'iss', 'iat', 'exp', 'jti', 'vc+sd-jwt'`) so the HR rep only sees the actual student data (Name, Degree, etc.).

### C. The View States (Tailwind CSS)
The component has three distinct, mutually exclusive visual states:

**State 1: The Request Screen (`*ngIf="requestActive && !verifiedData"`)**
*   Displays a centered card with a blue bounding box.
*   Renders the `<qrcode>` component.
*   **CRITICAL BUG FIX:** Must use `[errorCorrectionLevel]="'M'"`. If `'H'` is used, mobile browsers fail to decode the dense `openid-vp://` string.
*   Displays an `animate-pulse` amber badge saying "Listening for cryptographic proof".

**State 2: Verification Failed (`*ngIf="error"`)**
*   Displays a red danger box (`bg-red-50 border-t border-red-100`).
*   Shows the exact backend error message (e.g., "Invalid Cryptographic Signature" or "Missing vp_token").
*   Includes a "Try Again" button mapped to `reset()`.

**State 3: Cryptographically Verified (`*ngIf="verifiedData"`)**
*   Displays a green success header (`bg-green-50 text-green-900`) confirming signature and registry validity.
*   **Data Grid:** Maps over `getClaimKeys()`. Displays each disclosed claim in a structured grid (`bg-slate-50 rounded-xl p-5 border border-slate-100`).
*   **Issuer Info:** Displays a footer proving *who* issued the credential, reading `verifiedData.issuer` (e.g., "University of Texas"), accompanied by a "Verified via Trust Registry" green shield icon.

## 6. DEPLOYMENT (`Dockerfile` & `nginx.conf`)
*   **Stage 1:** `node:20-alpine`, `npm run build -- --configuration production`.
*   **Stage 2:** `nginx:alpine`, copies `/dist/verifier-frontend/browser` to `/usr/share/nginx/html`.
*   **nginx.conf:** Must use `try_files $uri $uri/ /index.html;` for Angular routing compatibility.
