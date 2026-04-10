# 🤖 AI SYSTEM PROMPT: REPO GENERATION INSTRUCTIONS

## 1. MACRO CONTEXT: THE BIG PICTURE
**Project:** TrustVault (Makeathon: Portable Proof with VC and VP)
**Goal:** This repository implements the **Verifier Backend**. In the "Triangle of Trust", this is the Employer (or Bank) that needs to cryptographically verify a student's degree.
**The Flow:** It generates an OID4VP presentation request (a QR code payload). The student's mobile wallet scans it and POSTs an SD-JWT (Selective Disclosure JWT) back to this server. This server must then call the **Trust Registry** to ensure the University is legitimate, verify the cryptographic signature, and apply the Selective Disclosure logic to reveal only what the student consented to share.

## 2. REPOSITORY ROLE: VERIFIER BACKEND
**Function:** A Java Spring Boot microservice. It serves a REST API for the Employer Frontend portal, acts as an OAuth2 Client to receive Webhook callbacks from the mobile wallet, and acts as an HTTP Client to query the Trust Registry backend. It uses WebSockets to push real-time verification results to the Employer's browser.

## 3. TECH STACK & EXACT DEPENDENCIES
*   **Language:** Java 21
*   **Framework:** Spring Boot 3.5.0 (Maven)
*   **Dependencies:** `spring-boot-starter-web`, `spring-boot-starter-websocket`, `lombok`.
*   **Cryptography:** `com.nimbusds:nimbus-jose-jwt:9.37.2` (To parse and verify signatures), `commons-codec:commons-codec:1.16.1` (For Base64 decoding).

## 4. ENVIRONMENT VARIABLES (`application.yml`)
```yaml
spring:
  application:
    name: verifier-backend
server:
  port: 8080
trustvault:
  registry:
    url: "https://trust-registry-backend-26472113294.us-central1.run.app/api/v1/registry"
  verifier:
    public-url: "https://verifier-backend-26472113294.us-central1.run.app"
```

## 5. EXTERNAL CLIENTS
*   **`TrustRegistryClient.java`**: Uses `RestTemplate` to execute a `GET` request to `${trustvault.registry.url}/resolve/{did}`. It returns a `Map<String, Object>` containing the Issuer's `name`, `status`, and `publicKey`. Throws a `RuntimeException` if the DID is not found.

## 6. CORE LOGIC (`VerifierController.java`)
**Base Mapping:** `@RequestMapping("/api/v1/verifier")`

### A. Presentation Request
*   **`POST /request`:** 
    *   Generates a random UUID `sessionId`. Stores it in a `ConcurrentHashMap<String, Boolean> activeSessions`.
    *   Constructs the `responseUri` pointing to `/callback/{sessionId}`.
    *   Constructs the `presentationUrl` (The OID4VP deep link): `"openid-vp://?client_id=TechCorp%20Employer&response_type=vp_token&response_mode=post&response_uri=" + responseUri + "&nonce=" + sessionId`.
    *   Returns `{presentation_url, session_id}`.

### B. The Callback (The Verification Engine)
*   **`POST /callback/{sessionId}`:** This is the endpoint the mobile wallet calls. It accepts a JSON body containing `vp_token` (the giant SD-JWT string) and `shared_claims` (a `List<String>` of fields the user consented to share).
*   **The Cryptographic Pipeline:**
    1.  **Parse:** Splits the `vp_token` by `~`. `parts[0]` is the main JWT.
    2.  **Trust Registry Check:** Parses the JWT to get the `issuer` DID. Calls `TrustRegistryClient.resolveDid(issuerDid)`. If the returned `status` is NOT `"ACTIVE"`, throw a security exception.
    3.  **Signature Check:** Extracts `publicKey` from the registry response. Uses `MACVerifier(publicKey)` from Nimbus-JOSE. Runs `signedJWT.verify(verifier)`. Throw exception if signature is invalid.
    4.  **Selective Disclosure (SD-JWT) Parsing:**
        *   Removes the raw `_sd` hash array from the claims.
        *   Loops through `parts[1]` to `parts[N]` (the clear-text disclosures sent by the wallet).
        *   Base64URL decodes each disclosure (e.g., `["salt", "gpa", 3.8]`).
        *   Extracts the field name and value, putting them into a temporary map.
    5.  **Consent Enforcement:** Creates a `finalDisclosedClaims` map. It strictly loops through the temporary map and only copies over fields that exist in the standard JWT spec (`sub`, `iss`, etc.) OR exist in the user's `shared_claims` list. 
    6.  **Cleanup & Notify:** Removes `sessionId` from `activeSessions`. Calls `NotificationService` to push a WebSocket message containing `{status: "VERIFIED", issuer: registryInfo.get("name"), claims: finalDisclosedClaims}`. Return `200 OK`.

## 7. WEBSOCKETS (`NotificationService.java`)
*   Extends `TextWebSocketHandler`. Intercepts query param `?sessionId=` in `afterConnectionEstablished`. Maps `sessionId` to `WebSocketSession`.
*   **`notifyStatus(sessionId, Map<String, Object> payload)`**: Uses `ObjectMapper` to convert the complex payload (containing the disclosed claims) into a JSON string and sends it down the pipe to the Angular frontend.

## 8. DEPLOYMENT (`Dockerfile`)
*   Standard multi-stage build (`eclipse-temurin:21-jdk-alpine` -> `eclipse-temurin:21-jre-alpine`). Expose `8080`.
