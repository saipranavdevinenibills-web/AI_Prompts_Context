# 🤖 AI SYSTEM PROMPT: REPO GENERATION INSTRUCTIONS

## 1. MACRO CONTEXT: THE BIG PICTURE
**Project:** TrustVault (Makeathon: Portable Proof with VC and VP)
**Goal:** We are building the "Digital Notary" ecosystem. The world currently runs on "Trust by PDF" which is easily faked. We are moving to "Trust by Cryptography" using W3C Verifiable Credentials.
**The Ecosystem:** We are building the frontend for the **Issuer (University)**. It must have two parts:
1.  **An Admin Dashboard:** For University Staff to CRUD student records and assign GPAs.
2.  **A Student Portal:** A self-service dashboard where a student logs in, views their official degree, and scans a QR code to securely transfer it to their mobile wallet.

## 2. REPOSITORY ROLE: ISSUER FRONTEND
**Function:** An Angular 18+ web application that implements the visual layer of the **OID4VCI Pre-Authorized Code Flow**. It manages JWT sessions, renders the `openid-credential-offer://` QR code, and listens to a real-time WebSocket to display a green "Success" animation the exact second the mobile wallet finishes downloading the credential.

## 3. TECH STACK & EXACT DEPENDENCIES
*   **Framework:** Angular 18+ (Standalone Components, Signals, RxJS)
*   **Styling:** Tailwind CSS 3.4.19, PostCSS 8.5.9.
*   **Core Libraries:** `angularx-qrcode:18.0.0` (Critical for rendering the dense JSON URL as a 2D matrix).
*   **Architecture:** `provideHttpClient` with functional interceptors (`HttpInterceptorFn`). `ReactiveFormsModule`.

## 4. ENVIRONMENT VARIABLES (`src/environments/environment.ts`)
*   `apiUrl`: `https://trustvault-issuer-backend-26472113294.us-central1.run.app/api/v1`
*   `wsUrl`: `wss://trustvault-issuer-backend-26472113294.us-central1.run.app/ws/status`
*   `issuerDid`: `did:web:university.edu`

## 5. MODELS & STATE (`src/app/models/credential.model.ts`)
*   `StudentProfile`: `{studentId: string, fullName: string, degreeName: string, gpa: number, graduationDate: string, major: string}`
*   `OID4VCI_Offer`: `{issuer_url: string, pre_authorized_code: string, user_pin_required: boolean, expires_in: number}`
*   `HandshakeStatus`: `{status: 'PENDING' | 'SCANNED' | 'ISSUED' | 'FAILED'}`

## 6. SERVICES & GUARDS
*   **`AuthService`:** Manages `isAuthenticated` (`signal<boolean>`) and `userRole` (`signal<string|null>`). `login(id, password)` maps to `/api/v1/auth/login`. Sets `token` and `role` in `localStorage`.
*   **`AuthInterceptor`:** Clones request, sets `Authorization: Bearer ${token}`.
*   **`AuthGuard`:** `CanActivateFn`. Checks `isAuthenticated()`. Redirects to `/login`.
*   **`AdminGuard`:** `CanActivateFn`. Checks `isAuthenticated() && getRole() === 'ROLE_ADMIN'`. Redirects to `/login`.
*   **`Oid4vciService`:** 
    *   `getCredentialOffer(id)` maps to `POST /issue/offer`.
    *   `generateQrUri(offer)` returns `openid-credential-offer://?credential_issuer=${encodeURIComponent(offer.issuer_url)}&pre-authorized_code=${offer.pre_authorized_code}`.
    *   `pollStatus(sessionId)` uses `rxjs/webSocket`. Subject connects to `${wsUrl}?sessionId=${sessionId}`. Sets `handshakeStatus` signal.

## 7. CORE COMPONENTS & UI SPECIFICATIONS
All components MUST be styled extensively with Tailwind CSS to look like a premium, modern "University Portal".

### A. AuthComponent (`/login`)
*   A clean, centered login card with two toggle buttons: **[Student]** and **[Administrator]**.
*   **State:** Uses `activeTab: 'STUDENT' | 'ADMIN'`. Resets form when switched.
*   **Inputs:** `studentId` (or 'Admin Username'), `password`.
*   **Logic:** Calls `AuthService.login`. If `res.role === 'ROLE_ADMIN'`, navigate to `/admin-dashboard`. Else, navigate to `/dashboard`.
*   **Error Handling:** Displays a red box (`bg-red-50`) if it catches a 401/403.

### B. AdminDashboardComponent (`/admin-dashboard`)
*   **Logic:** Fetches `GET /admin/students`. Displays in a `<table>`.
*   **"+ Add Student" Button:** Opens a fixed, full-screen Modal overlay (`bg-gray-900/50`).
*   **The Reactive Form:** Handles Create and Edit (`PUT /admin/students/{id}`).
*   **CRITICAL VALIDATION UI:** 
    *   Fields: `studentId`, `fullName`, `password`, `degreeName`, `major`, `gpa` (min 0.0, max 4.0), `graduationDate`.
    *   All required fields have a red asterisk `*`.
    *   If `isInvalid('fieldName')` is true (meaning it's invalid AND dirty/touched), the input box gets a red border (`border-red-300 ring-1 ring-red-300`).
    *   Explicit red `<p>` text appears below the field (e.g., *"Must be 0.0 to 4.0"*).
    *   The "Save" button is `[disabled]="studentForm.invalid"`.

### C. DashboardComponent (`/dashboard`)
*   **Logic:** Fetches `GET /students/me`. 
*   **The Card:** Uses `CredentialCardComponent` to render a glassmorphism gradient card (`bg-gradient-to-br from-gray-900 to-gray-800`). Displays GPA, Name, Degree.
*   **The Effect:** Uses an Angular `effect(() => { if (this.status() === 'ISSUED') ... })` to automatically close the QR modal and show the `SuccessToastComponent` the exact millisecond the WebSocket fires.

### D. IssuanceModalComponent (The OID4VCI Handshake UI)
*   **Logic:** Triggered when the student clicks "Claim to Wallet".
*   Shows an `ActivityIndicator` (Tailwind spinner) while `getCredentialOffer` is resolving.
*   Once resolved, renders `<qrcode [qrdata]="qrData" [width]="200" [errorCorrectionLevel]="'M'"></qrcode>`. 
*   *Hackathon Pro-Tip:* The `errorCorrectionLevel` MUST be `'M'`. If it's `'H'`, the QR code is too dense for web browsers to scan reliably.

### E. SuccessToastComponent
*   A fixed top-right card (`fixed top-10 right-10 bg-white border-l-4 border-green-500`).
*   Uses a custom keyframe animation in the `@Component` styles: `@keyframes bounce-in { 0% { transform: translateY(-20px); opacity: 0; } 100% { transform: translateY(0); opacity: 1; } }`.
*   Says: "Credential Issued! Your B.E. Computer Science has been successfully secured to your TrustVault mobile wallet."

## 8. DEPLOYMENT
*   A multi-stage `Dockerfile`. 
*   **Stage 1:** `node:20-alpine`, `npm run build -- --configuration production`.
*   **Stage 2:** `nginx:alpine`, copies `/dist/.../browser` to `/usr/share/nginx/html`.
*   Uses a custom `nginx.conf` (`try_files $uri $uri/ /index.html;`) to prevent 404s on Angular routes.
