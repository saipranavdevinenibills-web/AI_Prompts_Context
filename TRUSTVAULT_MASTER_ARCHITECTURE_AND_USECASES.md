# 🏆 TrustVault: Master Architecture, Use Cases, & Hackathon Strategy

This document is the definitive guide to the TrustVault ecosystem. It explains **what** we built, **how** the 7 microservices interact, **why** we made specific architectural choices, and the **detailed use cases** designed to win the Makeathon. Use this document to pitch to judges, align your team, and train AI agents on the system's business logic.

---

## 1. The Core Problem & Our Vision
**The Problem:** The current verification landscape relies on "Trust by PDF" or manual phone calls. Documents are easily forged (Photoshop), manual verification takes days (friction), and standard digital documents force users to share everything or nothing (privacy leakage).
**The Solution:** TrustVault. A W3C-compliant Decentralized Identity (SSI) ecosystem using Verifiable Credentials (VCs). We replace manual phone calls with instant cryptographic math.

**Hackathon KPIs Hit:**
*   Issuance in `< 5 seconds`.
*   Verification in `< 3 seconds`.
*   Revocation detection in `< 10 seconds`.

---

## 2. The Ecosystem: How the 7 Services Combine
We built a true "Triangle of Trust" using 7 distinct microservices. Here is the lifecycle of a credential through our system:

1.  **The Trust Registry (Backend & Frontend):** The Global Referee. Before anything happens, a Governance Authority (e.g., the Government) logs into the `Trust Registry Frontend`, registers "University of Texas" with a `did:web:university.edu`, and saves their Public Key to the `Trust Registry Backend`.
2.  **The Issuer (Backend & Frontend):** The Source of Truth. An admin logs into the `Issuer Frontend`, enters a student's data, and clicks "Issue". The `Issuer Backend` generates a unique, one-time QR code.
3.  **The Wallet (React Native App):** The Holder. The student scans the QR code. The app connects to the `Issuer Backend` via the **OID4VCI** protocol, cryptographically locks the credential to the phone's hardware, and saves it in the Vault.
4.  **The Verifier (Backend & Frontend):** The Gatekeeper. The student goes to TechCorp for a job. TechCorp's `Verifier Frontend` displays an **OID4VP** QR code. The student scans it, uses the Wallet UI to *hide their GPA*, and beams the proof to the `Verifier Backend`.
5.  **The Verification Loop:** The `Verifier Backend` intercepts the proof, dynamically asks the `Trust Registry Backend` if the University is still "ACTIVE", verifies the mathematical signature using the Registry's Public Key, and uses **WebSockets** to instantly push the verified (and filtered) data back to the `Verifier Frontend` screen.

---

## 3. Deep Dive Use Cases (For the Judges)

### Use Case A: The Privacy-Preserving Job Application (The Primary MVP)
*   **Scenario:** Sai Pranav applies for a Software Engineering role at TechCorp. TechCorp needs to verify his Degree and Major, but by law, they cannot ask for his Age or GPA to prevent hiring bias.
*   **The Flawed Old Way:** Sai emails a PDF transcript. The HR rep sees his GPA anyway, introducing bias. HR spends 3 days calling the University to verify the PDF isn't photoshopped.
*   **The TrustVault Way:** 
    *   TechCorp's portal shows a QR code requesting the Degree.
    *   Sai scans it. His Wallet shows a Consent Screen. 
    *   He toggles **OFF** the "GPA" field. (This utilizes **SD-JWT Selective Disclosure**).
    *   He clicks "Share".
    *   In `1.2 seconds`, TechCorp's screen turns Green. The cryptography proves the University issued the degree, but the GPA field mathematically reads as a permanently sealed "scratch-off" sticker. TechCorp hires him instantly.

### Use Case B: The 3-Second Revocation (Security & Governance)
*   **Scenario:** A hacker manages to infiltrate the University's database and issues themselves a fake Medical Degree. The University discovers this 2 hours later.
*   **The Flawed Old Way:** The University sends out an email to hospitals. The hacker has already used the PDF at a hospital that didn't check their email.
*   **The TrustVault Way:**
    *   The University Admin logs into the `Trust Registry Frontend` and clicks **[Revoke]** next to the credential.
    *   The `Trust Registry Backend` updates the ledger.
    *   *5 seconds later*, the hacker tries to present the ID at a hospital. 
    *   The Hospital's `Verifier Backend` checks the Trust Registry as part of the cryptographic handshake. It sees `STATUS: REVOKED`.
    *   The verification fails instantly. The fraud is stopped at the door.

### Use Case C: Multi-Domain Portability (AgriTech vs. Finance)
*   **Scenario:** We need to prove the system isn't hardcoded just for "Students".
*   **The TrustVault Way:** Because we rely on dynamic JSON claims and schema-agnostic SD-JWTs, the exact same Mobile Wallet app can hold a **"Soil Health Certificate"** (issued by the Dept of Agriculture) and a **"Credit Score"** (issued by a Bank). The Verifier Backend's Policy Engine simply asks for different JSON keys. This proves to the judges that TrustVault is a global Digital Public Infrastructure (DPI) platform, not just a niche app.

---

## 4. Architectural Reasoning: The "Why"

To win a Hackathon, you must defend your tech choices. Use these explanations:

### Why SD-JWT instead of standard W3C JWTs or JSON-LD?
*   *Reasoning:* Standard JWTs are "All-or-Nothing". If you change one byte to hide your address, the Issuer's signature breaks. JSON-LD requires complex Zero-Knowledge Proofs (ZKPs) which are computationally heavy for mobile phones. 
*   *The Winning Move:* We chose **SD-JWT (Selective Disclosure)**. By hashing each field individually with a cryptographic Salt (`Hash(Salt + "GPA" + 3.8)`), the Wallet can simply withhold the Salt for the GPA. The Verifier can still validate the Master Signature, but cannot reverse-engineer the GPA hash. It is lightweight, modern, and highly secure.

### Why OID4VCI "Pre-Authorized Code" Flow instead of standard OAuth2?
*   *Reasoning:* Standard OAuth2 requires the user to scan a QR code, open a browser on their phone, log in to the University, and then get redirected back to the app. It's clunky.
*   *The Winning Move:* We used the **Pre-Authorized Code Flow**. Because the student is *already logged in* on their laptop to see the QR code, the Java Backend generates a one-time secret tied to that exact laptop session. The phone just acts as a pure receiver. It turns a 5-step process into a 1-tap "Scan and Claim" experience.

### Why Dual-Engine React Native (Native + Web)?
*   *Reasoning:* Most hackathon teams fail because judges can't easily install `.apk` files or deal with Apple TestFlight restrictions on their personal phones to test the app.
*   *The Winning Move:* We built the Wallet in Expo React Native with a **Dual-Engine Web Fallback**. If compiled natively, it uses physical hardware Keystores and Native Cameras. If accessed via the Cloud Run Web URL, it dynamically swaps to `html5-qrcode` and `localStorage`. The judges can test the full "Mobile App" experience directly in Safari/Chrome without installing anything.

### Why WebSockets over REST Polling?
*   *Reasoning:* If the laptop is waiting for the phone to scan a QR code, hitting the server with a REST `GET` request every 2 seconds wastes massive database resources and causes a noticeable delay.
*   *The Winning Move:* We implemented raw WebSockets. The laptop opens a silent tunnel to the Java server. When the phone completes the handshake, the Java server pushes an event down the pipe, causing the laptop UI to update in `~50 milliseconds`. This guarantees we smash the Makeathon's `< 3 seconds` validation KPI.

---

## 5. Future Roadmap (What's Next for Production)

If asked by the judges "How would you scale this?", present this roadmap:
1.  **AI-Assisted Legacy Intake:** Integrate an LLM (like GPT-4o or Gemini) into the Issuer portal. An admin uploads a blurry 1990s scanned PDF; the AI extracts the data, maps it to the SD-JWT schema, and presents it to the admin for 1-click signing.
2.  **Bitstring Status Lists:** For the MVP, the Verifier queries the Trust Registry API directly. For production, we will implement W3C Bitstring Status Lists—a highly compressed binary file cached on the Verifier's edge network—allowing revocation checks in `< 50ms` without making network calls to the Registry.
3.  **Social Recovery:** If a user loses their hardware-bound phone, we will implement Shamir's Secret Sharing to split their backup key among 3 trusted friends, ensuring true Self-Sovereign Identity without the risk of permanent lock-out.
