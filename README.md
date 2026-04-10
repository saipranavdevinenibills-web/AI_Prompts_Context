# 🧠 TrustVault AI Context & Prompts

This repository contains the definitive, highly-detailed **AI Generation Prompts** for the entire Decentralized Identity ecosystem (Makeathon: Portable Proof with VC and VP).

Each file inside the `/prompts` directory corresponds to a specific microservice or application in the "Triangle of Trust". These files are specifically engineered to be copy-pasted into an LLM (like GitHub Copilot, ChatGPT, or Claude) to reliably regenerate the exact, working architecture, cryptography, and logic flows.

## 📂 Repositories Documented

1. **Issuer Backend** (`issuer_backend_PROMPT.md`) - Java/Spring Boot server generating OID4VCI offers and signing SD-JWTs.
2. **Issuer Frontend** (`issuer_frontend_PROMPT.md`) - Angular portal for University Admins to onboard students and for students to claim their degrees via QR code.
3. **Mobile Wallet** (`user_mobile_app_PROMPT.md`) - React Native (Expo) PWA app acting as the Holder's Secure Enclave, supporting Selective Disclosure (SD-JWT) UI and hardware-bound storage.
4. **Trust Registry Backend** (`trust_registry_backend_PROMPT.md`) - Java/Spring Boot API serving as the global decentralized phonebook/ledger.
5. **Trust Registry Frontend** (`trust_registry_frontend_PROMPT.md`) - Angular Governance Dashboard to Suspend or Revoke Issuers globally.
6. **Verifier Backend** (`verifier_backend_PROMPT.md`) - Java/Spring Boot engine that verifies OID4VP presentation logic, SD-JWT cryptography, and Trust Registry status.
7. **Verifier Frontend** (`verifier_frontend_PROMPT.md`) - Angular UI for Employers/HR to generate OID4VP QR requests and view verified candidate data in real-time via WebSockets.

---
*Generated for the TrustVault Makeathon.*
