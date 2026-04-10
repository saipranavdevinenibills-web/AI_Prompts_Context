# 🤖 AI SYSTEM PROMPT: REPO GENERATION INSTRUCTIONS

## 1. MACRO CONTEXT: THE BIG PICTURE
**Project:** TrustVault (Makeathon: Portable Proof with VC and VP)
**Goal:** We are building the "Digital Notary" ecosystem. The world currently runs on "Trust by PDF" which is easily faked. We are moving to "Trust by Cryptography" using W3C Verifiable Credentials.
**The Ecosystem:** We are building the **Holder (Mobile Wallet)**. It must act as a Secure Enclave that receives an SD-JWT credential via OID4VCI (scanning a QR code), stores it permanently, and allows the user to selectively present specific fields to a Verifier via OID4VP.

## 2. REPOSITORY ROLE: TRUSTVAULT MOBILE WALLET
**Function:** A React Native (Expo) application. 
**CRITICAL ARCHITECTURE NOTE:** This app is designed to be compiled natively (iOS/Android) AND as a Progressive Web App (PWA). Therefore, any hardware-specific APIs (Camera, Keychain) MUST have a `Platform.OS === 'web'` fallback implementation.

## 3. TECH STACK & EXACT DEPENDENCIES
*   **Framework:** React Native + Expo (SDK 50+)
*   **Styling:** `nativewind: ^2.0.11`, `tailwindcss: 3.3.2`. 
*   **Navigation:** `@react-navigation/native-stack` v7.
*   **Camera/Scanning:** 
    *   `expo-camera` (~17.0.10) for Native iOS/Android.
    *   `html5-qrcode` (npm library) strictly for `Platform.OS === 'web'` to handle PWA camera feeds.
*   **Security:** `expo-secure-store` (~15.0.8) for hardware-bound storage (Keychain/Keystore).

## 4. CORE MODELS (`src/types/index.ts`)
*   `CredentialOffer`: `{issuer_url, pre_authorized_code, user_pin_required, expires_in}`
*   `TrustVaultCredential`: `{id, format, rawCredential, metadata: {issuer, type, issueDate, parsedClaims: Record<string, any>}}`

## 5. SERVICES & PROTOCOL LOGIC
### A. StorageService (`src/services/StorageService.ts`)
*   **The Bug We Fixed:** `expo-secure-store` crashes if run in a web browser.
*   **The Implementation:** You MUST use a Dual-Engine check.
    *   If `Platform.OS === 'web'`, use `localStorage.getItem` / `setItem`.
    *   If Native, use `SecureStore.getItemAsync` / `setItemAsync`.
    *   `saveCredential(cred)`: Fetches array, filters out duplicates by `id`, appends new cred, stringifies to JSON, saves.

### B. OID4VCIClient (`src/utils/oid4vciClient.ts`)
*   `processQR(qrUrl)`: Parses `openid-credential-offer://` URI. Extracts `credential_issuer` and `pre-authorized_code`.
*   `claimCredential(offer)`: 
    1.  `POST` to `${issuer_url}/vci/token` with `{pre_authorized_code}`.
    2.  `POST` to `${issuer_url}/vci/credential` with `Authorization: Bearer <access_token>`.
    3.  `POST` to `${issuer_url}/vci/notification` with `{event: 'credential_accepted', session_id: pre_authorized_code}` to trigger Issuer WebSockets.
    4.  Return formatted `TrustVaultCredential` object.

### C. OID4VPClient (`src/utils/oid4vpClient.ts`)
*   `processQR(qrUrl)`: Parses `openid-vp://` URI. Extracts `response_uri` and `nonce`.
*   `presentCredential(credential, sharedClaims, responseUri)`: Simulates SD-JWT Selective Disclosure. `POST` to `response_uri` sending `{vp_token: credential.rawCredential, shared_claims: ["DegreeName", "Name"]}`. (The keys are filtered from the `sharedClaims` record where value === true).

## 6. UI/UX SPECIFICATIONS (NATIVEWIND)
All components MUST be styled with Tailwind CSS via NativeWind (`className="..."`).

### A. AppNavigator (`src/navigation/AppNavigator.tsx`)
*   Native Stack. `VaultScreen` (Home), `ScannerScreen` (presentation: fullScreenModal), `ConsentScreen` (presentation: modal). Dark theme (`backgroundColor: '#000'`).

### B. VaultScreen (`src/screens/VaultScreen.tsx`)
*   **State:** `credentials` array, `loading` boolean. Uses `navigation.addListener('focus', ...)` to reload credentials from `StorageService`.
*   **Empty State:** Displays a "Your Vault is Empty" message and a "Scan to Claim ID" button navigating to `ScannerScreen`.
*   **List State:** Renders a `FlatList` of `TrustVaultCredential` objects. Cards are styled dark (`bg-gray-800 rounded-2xl`). Clicking a card navigates to `ConsentScreen`.

### C. ScannerScreen (`src/screens/ScannerScreen.tsx`)
*   **CRITICAL DUAL-ENGINE ARCHITECTURE:** 
    *   If `Platform.OS === 'web'`: Safely require `html5-qrcode`. Render an empty `<View nativeID="reader" style={{ width: '100%', height: '100%', position: 'absolute' }} />` and manually initialize `new Html5Qrcode("reader")` inside a `useEffect()`. **Do NOT use `qrbox` configuration**; it breaks responsiveness. Just use `{ fps: 10 }`.
    *   If Native: Render `<CameraView>` from `expo-camera`. Ensure `useCameraPermissions()` is requested.
*   **Layout:** A `flex-1` view with a centered, absolute square box (`aspect-square border-4 border-blue-500 rounded-3xl`). The camera feed is mounted *inside* this box so it doesn't break PWA layouts.
*   **Logic:** On scan, sets `loading=true`, calls `OID4VCIClient` -> `StorageService.saveCredential` -> `navigation.navigate('Vault')`. Shows `<ActivityIndicator>` overlay during HTTP requests.

### D. ConsentScreen (`src/screens/ConsentScreen.tsx`)
*   Receives `TrustVaultCredential` via route params.
*   **State (`sharedClaims`):** Initializes a `Record<string, boolean>` where every key in `credential.metadata.parsedClaims` is set to `true`.
*   **The Magic SD-JWT UI:** Maps over `parsedClaims`. Renders a Native `<Switch>` component next to every claim. 
    *   If toggled off: Text gets `line-through text-gray-600` styling and displays `••••••••••••`.
*   "Share Credential" button triggers `OID4VPClient.presentCredential()`. Navigates `goBack()` on success.
