# T-013: Research — check the IdentityApi design draft against industry standards

Output file: `research/identityapi-design-review.md`. This is research only — no code, no repo other than `garbati-platform`. It exists to help the owner (a C#/.NET learner) and the coordinator decide whether to ratify the IdentityApi design draft as v2 (D-9), by checking its specific technical choices against established standards and known failure modes — not to make any decision itself.

## Context

The owner has a design draft at `/Users/USER/Downloads/Garbati_IdentityApi_Design.md` (489 lines) for `IdentityApi` — the GARBATI service that owns customer identity, KYC, mandates, and issues a signed, short-lived "authorization grant" that `LedgerApi` verifies locally before posting a customer-originated transaction (this is `ARCHITECTURE.md`'s decision P-12). Read that file in full before anything else.

Also read, for cross-reference:
- `/Users/USER/Projects/garbati-platform/ARCHITECTURE.md` Sections 3, 9, and the P-12 row of Section 8 — the platform-level version of the same token design, already partly decided.
- `/Users/USER/Projects/garbati-platform/research/nigerian-regulatory-requirements.md` (T-008's output) — for KYC-tier and data-retention claims in the draft that should agree with, not contradict, that research.
- `/Users/USER/Projects/garbati-platform/research/architecture-patterns-p1-p6.md` (T-011's output) — it already validated the PDP/PEP-style decide/execute separation; this task is the next layer down, checking the *token* mechanism itself.

## Questions to answer

1. **Token design (JWT/JWS).** The draft proposes a JWS/JWT-shaped signed grant with `iss`, `aud`, `jti`, `sub`, operation, accounts, amount, currency, idempotency key, `iat`, `exp` (30s), asymmetric signing, local verification by the recipient (no synchronous call back to the issuer). Check this against published JWT/JOSE best-practice guidance (e.g. RFC 8725 "JWT Best Current Practices", RFC 9068 if relevant, OWASP's JWT guidance). Does the draft's design match current best practice? Specifically check: algorithm pinning (not trusting `alg` from the token itself), key-ID-based rotation, audience/issuer validation, and whether a 30-second expiry with "deliberately small" clock-skew tolerance is a reasonable, named pattern or unusually tight/loose compared to typical short-lived-token designs (e.g. OAuth2 token exchange, SAML assertions, or similar transaction-bound tokens used in payments).
2. **Local verification / no synchronous call-back.** The draft's "IdentityApi signs, LedgerApi verifies locally with a trusted public key, no runtime call back to IdentityApi" is a specific instance of a more general pattern. Is this a recognized approach (e.g., how OAuth2/OIDC's JWT access tokens let a resource server verify locally via JWKS instead of calling the authorization server's introspection endpoint every time)? What do standard sources say about the trade-off this draft explicitly accepts (a revoked mandate's already-issued grant stays valid until it expires)?
3. **Mandate / "power to act on an account" revocation.** Is "revoke-and-create" (never rewrite history, a revoked mandate is retained) standard practice for authorization/permission records in financial systems? Compare against how banking mandates, OAuth scopes, or RBAC/ABAC permission grants are typically revoked and audited elsewhere.
4. **Fail-closed and audit/logging rules.** The draft's Section 4.2 (fail closed on timeout/ambiguous failure) and Section 6.2 (never log BVN/NIN, keys, full tokens, or full request bodies) — do these match standard security guidance (e.g. OWASP logging cheat sheet, general fail-safe-defaults principle)? Is there anything standard guidance calls for that the draft is missing?
5. **What's missing or weak, compared to known failure modes of similar systems.** Based on common real-world incidents or documented pitfalls in token-based service-to-service authorization (replay, confused-deputy, token substitution/mix-up attacks between services, key compromise/rotation gaps), does the draft address the major ones? Name anything you find under-specified or missing, citing the source describing that failure mode.

## Format

Write `research/identityapi-design-review.md` with: a short plain-language summary up top (the owner is a C#/.NET learner — define any new jargon, e.g. "JOSE," "confused deputy," "token substitution attack," on first use), then the five questions above each with a sourced answer, then a **"Specific findings"** section listing anything the draft does unusually well, anything that deviates from standard practice (flag severity: informational / worth tightening / should fix before production), and anything a standard source would call a gap. End with a "What this does not settle" section — this is industry-pattern and security-literature research, it does not replace the owner's own decision on any of D-9's six items (decisions table, token endpoint, scope split, Closed/Deceased cascade, freeze ownership, one active mandate index), and it is not a cryptography or compliance audit.

Report what you could not find or verify.
