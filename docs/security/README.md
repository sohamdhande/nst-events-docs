# Security & Identity Architecture Documentation

This directory contains the authoritative documentation for identity, authentication, authorization, RBAC, JWT, session lifecycle, and PostgreSQL Row Level Security (RLS) for the NST-Events platform.

---

## Authoritative Documentation Suite

The following 7 core documents establish the verified source of truth for all security, authentication, and authorization behaviors:

1. **[Authentication Architecture](authentication.md):** Complete end-to-end authentication sequence, Google OAuth lifecycle, web vs mobile authentication flows, transparent token refresh interceptors, and session termination.
2. **[Google OAuth 2.0 Integration](oauth.md):** Google Identity configuration, OAuth CSRF state cookie, institutional domain restrictions (`@adypu.edu.in`, `@newtonschool.co`), student directory verification, user upserting, and automatic academic batch inference.
3. **[JWT, Refresh Tokens, & Sessions](jwt-and-sessions.md):** Stateless JWT specifications (`sub`, `secVer`, `iat`, `exp`), immediate session revocation via security version increments, database refresh token rotation, concurrent refresh race handling (5-second grace window), and cookie security attributes.
4. **[Authorization Model & Defense-in-Depth](authorization.md):** The multi-layered authorization pipeline (Frontend UX -> Express RBAC -> Service Transaction -> PostgreSQL RLS -> PL/pgSQL RPCs), request tracing, and endpoint security classifications.
5. **[Role-Based Access Control (RBAC)](rbac.md):** Decoupled role taxonomies (`GlobalRole`, `ClubRole`, `ParticipationRole`), the Primary Club Rule for co-hosted events, event lock boundaries, and the complete system authorization matrix.
6. **[Row Level Security (RLS) & Database Context](rls.md):** Zero-trust database principles, transaction-local user context injection (`withUserContext`), recursion breakers (`can_see_user_as_organizer`), and `SECURITY DEFINER` search path hardening.
7. **[Platform Security Model & Threat Analysis](security-model.md):** Tier-by-tier security responsibilities, STRIDE threat model analysis, and verified security implementation risks (e.g. multi-replica mobile auth codes, reverse proxy IP forwarding).

---

## Historical & Topic Deep-Dives
The legacy numbered documents provide historical context on specific security milestones:
- [01. RLS Architecture](01-rls-architecture.md)
- [02. JWT Strategy](02-jwt-strategy.md)
- [03. Permission Enforcement](03-permission-enforcement.md)
- [04. Role Resolution](04-role-resolution.md)
- [05. Table Policy Matrix](05-table-policy-matrix.md)
- [06. Multi-Role Handling](06-multi-role-handling.md)
- [07. Storage RLS](07-storage-rls.md)
- [08. Attendance Security](08-attendance-security.md)
- [09. Threat Model](09-threat-model.md)
- [10. Stress Test Results](10-stress-test-results.md)
- [11. Security Freeze V1](11-security-freeze-v1.md)
