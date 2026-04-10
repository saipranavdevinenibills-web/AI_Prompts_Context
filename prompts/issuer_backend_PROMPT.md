# 🤖 AI SYSTEM PROMPT: REPO GENERATION INSTRUCTIONS

## 1. MACRO CONTEXT: THE BIG PICTURE
**Project:** TrustVault (Makeathon: Portable Proof with VC and VP)
**Goal:** Solve identity fraud and friction using Decentralized Identity. We are moving from a world of "Trusting a piece of paper" to "Trusting Cryptography" via the W3C Verifiable Credentials 2.0 standard.
**The Ecosystem (Triangle of Trust):**
1.  **Issuer:** The source of truth (University/Bank) that digitally signs the data.
2.  **Holder:** The mobile wallet that securely stores the data and selectively shares it.
3.  **Verifier:** The destination (Employer/Bank) that verifies the cryptography without calling the Issuer.
**Protocols Used:** OID4VCI (for issuing), OID4VP (for presenting), SD-JWT (for Selective Disclosure/Privacy).

## 2. REPOSITORY ROLE: ISSUER BACKEND
**Function:** This is a Java Spring Boot microservice acting as the "University". It holds student records in a database, provides a REST API for a Frontend portal, and implements the **OID4VCI Pre-Authorized Code Flow** to securely hand off a digitally signed SD-JWT credential to a mobile wallet.

## 3. TECH STACK & EXACT DEPENDENCIES
*   **Language:** Java 21
*   **Framework:** Spring Boot 3.5.0 (Maven)
*   **Core Dependencies:** `spring-boot-starter-web`, `spring-boot-starter-data-jpa`, `h2` (In-memory DB), `spring-boot-starter-websocket`, `spring-boot-starter-security`, `spring-boot-starter-validation`, `lombok`.
*   **Custom Cryptography Dependencies:** 
    *   `com.nimbusds:nimbus-jose-jwt:9.37.2` (For standard JWT and JWS signing)
    *   `commons-codec:commons-codec:1.16.1` (For SHA-256 hashing during SD-JWT creation)

## 4. DATABASE & INITIALIZATION
*   **Database:** H2 In-Memory (`jdbc:h2:mem:trustvaultdb`). Property `spring.jpa.defer-datasource-initialization=true` is required.
*   **Startup Seeding:** Use a `CommandLineRunner` bean in `IssuerApplication.java` to inject two users into the `StudentRepository` if they don't exist:
    1.  **Admin:** `studentId`: "admin", `password`: "admin" (BCrypt hashed), `fullName`: "System Administrator", `role`: "ROLE_ADMIN".
    2.  **Student:** `studentId`: "SP-1010101", `password`: "password" (BCrypt hashed), `fullName`: "Sai Pranav Devineni", `degreeName`: "B.E. Computer Science", `major`: "Software Engineering", `gpa`: 3.8, `graduationDate`: "2026-05-15", `role`: "ROLE_STUDENT".

## 5. DOMAIN MODELS & DTOs
*   **Entity:** `Student.java` -> `@Id @GeneratedValue Long id`, `String studentId`, `String password`, `String fullName`, `String degreeName`, `Double gpa`, `String graduationDate`, `String major`, `String didBinding`, `String role`.
*   **DTOs:** 
    *   `LoginRequest` (`studentId`, `password`)
    *   `LoginResponse` (`token`, `role`)
    *   `StudentProfile` (Same as Student entity but excludes `id`, `password`, `didBinding`, `role`)
    *   `Oid4vciOffer` (`issuer_url`, `pre_authorized_code`, `user_pin_required`, `expires_in`)

## 6. SECURITY ARCHITECTURE (RBAC)
*   **Configuration:** `SecurityConfig.java`. Disable CSRF. Set Session Creation to `STATELESS`.
*   **CORS:** Global `CorsConfiguration` allowing `*` for Origins, Methods, and Headers.
*   **Route Protections:**
    *   `.requestMatchers("/api/v1/auth/**", "/api/v1/vci/**", "/ws/**", "/h2-console/**").permitAll()`
    *   `.requestMatchers("/api/v1/admin/**").hasAuthority("ROLE_ADMIN")`
    *   `.anyRequest().authenticated()`
*   **JWT Filter:** `JwtAuthTokenFilter.java` intercepts `Authorization: Bearer <token>`. Parses it using `Nimbus SignedJWT`. Extracts `sub` (studentId) and `role` claim. Sets `UsernamePasswordAuthenticationToken` in `SecurityContextHolder`.

## 7. CORE SERVICES & LOGIC (CRITICAL)
*   **OfferService.java:** Implements the OID4VCI caching. 
    *   *Hackathon Bypass:* Instead of Redis, use a `ConcurrentHashMap<String, String> redisTemplateMock` to map `"offer:" + pre_authorized_code` to `studentId`.
    *   `createOffer(studentId)`: Generates a UUID `preAuthCode`. Returns `Oid4vciOffer` object with `expires_in` = 300.
    *   `validateAndConsume(preAuthCode)`: Fetches `studentId` from map, **deletes the key immediately** (single-use guarantee), and returns `studentId`.
*   **JwtSignerService.java:** Implements the SD-JWT Selective Disclosure cryptography.
    *   Uses a dummy `SECRET` = `"01234567890123456789012345678912"`.
    *   *The Magic Logic:* Generate a random UUID `gpaSalt`. Hash it: `DigestUtils.sha256Hex(gpaSalt + "gpa" + student.getGpa())`.
    *   Create `JWTClaimsSet` with standard claims (`fullName`, `degreeName`, etc.), but add the hash to a String array claim named `_sd`.
    *   Sign the JWT using `MACSigner`.
    *   **Return Format:** `signedJWT.serialize() + "~" + Base64URLSafe("[\"" + gpaSalt + "\", \"gpa\", " + student.getGpa() + "]") + "~"`. (This glues the disclosure to the token).
*   **NotificationService.java:** Implements Raw WebSockets (No STOMP).
    *   Extends `TextWebSocketHandler`. Intercepts query param `?sessionId=` in `afterConnectionEstablished`. Maps `sessionId` to `WebSocketSession`.
    *   `notifyStatus(sessionId, status)`: Sends JSON `{"status": "ISSUED"}` down the pipe.

## 8. CONTROLLERS (API CONTRACT)
*   **AuthController:** `POST /api/v1/auth/login`. Returns JWT Session Token (expires in 1 hour) and `role`.
*   **AdminController:** Base path `/api/v1/admin/students`. `GET` (returns all except ROLE_ADMIN), `POST` (hashes password, sets ROLE_STUDENT), `PUT /{id}` (updates details).
*   **StudentController:** `GET /api/v1/students/me`. Returns profile for the currently logged-in principal.
*   **VciController (The Protocol):** Base path `/api/v1`
    *   `POST /issue/offer`: Calls `OfferService.createOffer`.
    *   `POST /vci/token`: Accepts `pre_authorized_code`. Returns standard OAuth2 `{access_token: "vci-access-SP-1010101-[UUID]", token_type: "Bearer", expires_in: 300}`.
    *   `POST /vci/credential`: Requires `Bearer` token. *Hackathon Bypass:* Split the access token by `-` to extract the `studentId`. Calls `JwtSignerService`. Returns `{format: "vc+sd-jwt", credential: "<SD_JWT_STRING>"}`.
    *   `POST /vci/notification`: Receives `{event: "credential_accepted", session_id: "<pre_auth_code>"}`. Calls `NotificationService.notifyStatus("ISSUED")`.

## 9. DEPLOYMENT
*   A multi-stage `Dockerfile` using `eclipse-temurin:21-jdk-alpine` to build (`./mvnw clean package -DskipTests`) and `eclipse-temurin:21-jre-alpine` to expose port 8080 and run the jar.
