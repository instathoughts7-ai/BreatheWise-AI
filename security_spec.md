# BreatheWise AI — Firestore Security Specification (Phase 0)

## 1. Data Invariants
1. **Master Gate & PII Isolation**: All persistent user records (`UserProfile`, `SavedLocation`, `SavedPlan`) are strictly scoped under `/users/{userId}`. Email addresses and auth metadata remain in Firebase Auth and are never duplicated in Firestore.
2. **Verified Identity**: Write operations require an authenticated, email-verified user (`request.auth != null && request.auth.token.email_verified == true`).
3. **Owner-Only Access**: Every `get`, `list`, `create`, `update`, and `delete` operation requires `request.auth.uid == userId` and `incoming().userId == userId`.
4. **Master Gate Relational Sync**: Subcollections (`/users/{userId}/locations/{locationId}` and `/users/{userId}/plans/{planId}`) validate parent existence via `exists(/databases/$(database)/documents/users/$(userId))` on create.
5. **Path Variable Hardening**: Every document ID (`userId`, `locationId`, `planId`) must match `^[a-zA-Z0-9_\-]+$` and length <= 128.
6. **Strict Blueprint Synchronicity & No Shadow Fields**: Every write validates exact required and allowed keys (`hasAll` + `hasOnly`), string bounds, numeric coordinate bounds (`latitude` in `[-90, 90]`, `longitude` in `[-180, 180]`), bounded arrays (`recommendations.size() <= 10`), and server-enforced timestamps (`request.time`).
7. **Immortal Fields**: `userId`, `locationId`, `planId`, and `createdAt` are immutable on update.
8. **SavedPlan Immutability**: Saved plans are point-in-time snapshots (`allow update: if false`, `allow delete: if isOwner(userId)`).

## 2. "Dirty Dozen" Adversarial Payloads (Must All Yield `PERMISSION_DENIED`)
1. **Unauthenticated read/write**: Any read or write to `/users/uid1` with `auth == null`.
2. **Cross-user profile read**: User `uid2` attempts `get(/users/uid1)`.
3. **Cross-user list scraping**: User `uid2` attempts `list(/users/uid1/locations)`.
4. **Unverified email write**: Authenticated user with `email_verified: false` creates `/users/uid1`.
5. **Identity spoofing**: User `uid1` writes `/users/uid1` with `userId: "uid2"`.
6. **Shadow field injection**: User `uid1` creates `/users/uid1` with extra field `isAdmin: true`.
7. **ID poisoning**: User `uid1` creates `/users/uid1/locations/${'a'.repeat(500)}`.
8. **Coordinate out-of-bounds**: User `uid1` saves location with `latitude: 120.5` or `longitude: -220`.
9. **Client timestamp forgery**: User `uid1` sets `createdAt` to a past timestamp instead of `request.time`.
10. **Immutable field tampering**: User `uid1` updates `/users/uid1` changing `createdAt`.
11. **Unbounded array / Denial of Wallet**: User `uid1` writes `/users/uid1/plans/p1` with 25 items in `recommendations` or >600 chars in `summary`.
12. **Orphaned subcollection write**: User `uid1` creates `/users/uid1/locations/loc1` before `/users/uid1` profile document exists.
