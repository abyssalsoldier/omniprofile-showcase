# OmniProfile

A zero-knowledge digital business card and contact-sharing platform. Users build a card (name, title, bio, contact links, custom theming) that's shared via a public link — but unlike a typical SaaS product, the operator (server) is architecturally incapable of reading the card contents, the user's password, or their private key material.

**Goal:** the server should know a user's email address and essentially nothing else.

## How the zero-knowledge design works

- **Passwords never leave the browser in raw form.** The client derives an "auth token" from the password via PBKDF2 and sends *that* to the server — the server only ever sees and stores an Argon2id hash of the derived token, never the password itself.
- **Card content is encrypted client-side.** Every field on a card (contact details, bio, theme, images) is serialized to JSON and encrypted with AES-256-GCM using a per-card key, before it ever touches the network. The server stores and returns opaque ciphertext.
- **The key-of-keys is encrypted too.** A user's "keychain" (the map of per-card encryption keys) is itself encrypted with a master key derived from their password, so the server never holds a key that could decrypt any card content.
- **Password recovery without server-side escrow.** A one-time recovery code, generated and shown to the user exactly once, re-wraps the master key — there's no "reset my password and keep my data" backdoor that depends on the server being trustworthy.
- **Public share pages decrypt in the visitor's browser.** The decryption key lives in the URL fragment (`#k=...`), which by design is never transmitted to the server on page load.

## Architecture

- **Backend:** FastAPI (Python 3), PostgreSQL, session auth via signed cookies (`itsdangerous`), Argon2id for password-verifier hashing, PBKDF2 + AES-256-GCM for the client-side envelope, TOTP-based MFA (`pyotp`), IP-based rate limiting on auth-sensitive routes (`slowapi`).
- **Frontend:** vanilla JS (no framework) driving the card editor, dashboard, and public share view; Web Crypto API for all client-side encryption/decryption.
- **Payments:** Stripe, for optional Pro-tier card slots.
- **Infrastructure:** Docker Compose, Caddy as a reverse proxy/TLS terminator with a locked-down CSP, deployed behind a Cloudflare Tunnel.
- **Privacy-conscious analytics:** public-page view/click analytics are recorded without storing raw visitor IPs or user-agents — IPs are HMAC-hashed and user-agents are reduced to a coarse device/browser bucket before anything is persisted.

## Notable engineering details

- Careful handling of the reverse-proxy trust chain so that rate limiting and IP-hashing key off the *real* client IP rather than the proxy's own container IP — a subtle bug class that silently defeats both protections if the proxy-header trust boundary isn't configured correctly.
- Ongoing, structured security review: the project maintains its own internal audit process, tracking findings (severity, root cause, fix, verification) through to resolution rather than treating a security pass as a one-time checklist.
- Defense-in-depth on the public share surface: strict output-encoding on every user-controlled field rendered into the DOM, validated allowlists for style/color/font values, and a CSP as a second layer against any regression in that encoding.

---

*This is the public-facing overview for a private project. Source code is not published, in keeping with the project's own security posture.*
