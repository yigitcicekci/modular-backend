# User, auth, session, and device workflows

## Contents

- Responsibilities and feature selection
- JWT, passwords, and fingerprints
- User and authorization
- Registration and login
- Refresh rotation and concurrency
- Recovery and account lifecycle
- Session storage and revocation
- Web and OAuth
- Device and push tokens
- Behavior checks

## Responsibilities and feature selection

| Owner | Responsibilities |
| --- | --- |
| User | Profile, allowed field updates, account state and lifecycle |
| Auth | Registration/login, identity verification, recovery and orchestration |
| Session (initially inside auth) | Session creation, refresh rotation, expiry, revocation |
| Device | User/device association, platform metadata and push token lifecycle |
| Guard/policy | Identity validation and route/record permission enforcement |

Do not generate every feature automatically. Use the requested login methods and platforms. Email/password implies safe hashing and the appropriate recovery/verification flows. OAuth-only products do not need unused password endpoints. Device/push features belong only in products using them. Keep session logic within auth until a separate module has a concrete consumer.

In new token-based APIs, short-lived access tokens with separately managed rotating refresh sessions are a useful starting design. Preserve a sound existing cookie-session or external identity system; JWT is not mandatory. Never hand-roll signatures or password hashing. Select maintained installed libraries, adapting to actual versions. For a new password hasher, prefer Argon2id with parameters checked against current OWASP guidance and measured on deployment hardware. Retain existing secure hashes; migrate legacy hashes on verified login when necessary. Never reduce hashing work to minimize complexity.

## JWT, passwords, and fingerprints

For a Nest JWT API, use `@nestjs/jwt` via `JwtModule.register({})` when `TokenService` supplies explicit purpose-specific options. Use existing `@nestjs/config` or validated config providers. A direct Nest `CanActivate` guard can use `TokenService` without Passport. Keep Passport if the application already uses its strategy flow. Add a provider SDK or a Fastify cookie integration only for an actual provider flow or browser cookie contract. Use `argon2` for new Argon2id password verification when there is no suitable installed hasher. Check that its native module loads in the release image.

```ts
// src/modules/authentication/token.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { JwtService } from '@nestjs/jwt';
import { randomUUID } from 'node:crypto';
import { AppException } from '../../common/exceptions/app.exception';

type Purpose = 'access' | 'refresh';
type Claims = { sub: string; sid: string; purpose: Purpose; exp: number };
type JwtPolicy = { secret: string; issuer: string; audience: string; expiresInSeconds: number };
type AuthConfig = { AUTH_TOKENS: Record<Purpose, JwtPolicy> };

@Injectable()
export class TokenService {
  private readonly policy: AuthConfig['AUTH_TOKENS'];

  constructor(private readonly jwt: JwtService, config: ConfigService<AuthConfig, true>) {
    this.policy = config.get('AUTH_TOKENS', { infer: true });
  }

  sign(userId: string, sessionId: string, purpose: Purpose) {
    const { secret, expiresInSeconds, issuer, audience } = this.policy[purpose];
    return this.jwt.signAsync({ sub: userId, sid: sessionId, purpose }, {
      secret, algorithm: 'HS256', issuer, audience, expiresIn: expiresInSeconds,
      jwtid: randomUUID(),
    });
  }

  async verify(token: string, purpose: Purpose): Promise<Claims> {
    const { secret, issuer, audience } = this.policy[purpose];
    const claims = await this.jwt.verifyAsync<Claims>(token, {
      secret, issuer, audience, algorithms: ['HS256'],
    }).catch(() => { throw new AppException('INVALID_TOKEN'); });
    if (claims.purpose !== purpose || typeof claims.sub !== 'string' || !claims.sub ||
        typeof claims.sid !== 'string' || !claims.sid || !Number.isFinite(claims.exp)) {
      throw new AppException('INVALID_TOKEN');
    }
    return claims;
  }
}
```

Load `AUTH_TOKENS` from validated startup config. Validate separate signing keys, issuers, audiences and positive TTLs before bootstrap. Keep the issuer, audience, signing algorithm, expiration, and token purpose checks in the active verification path. Map expired access/refresh to separate typed codes when clients need that distinction. Keep existing claim meanings during migration. Refresh may instead use the opaque secret below. When signing JWT refresh tokens, store their digest and use the same atomic rotation as for opaque refresh. [Nest JWT](https://github.com/nestjs/jwt)

```ts
// src/modules/authentication/password.service.ts
import { Injectable, type OnModuleInit } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import argon2 from 'argon2';

@Injectable()
export class PasswordService implements OnModuleInit {
  private readonly pepper: Buffer;
  private dummyHash!: string;

  constructor(config: ConfigService) {
    this.pepper = Buffer.from(config.getOrThrow<string>('PASSWORD_PEPPER'), 'base64');
  }

  async onModuleInit() {
    this.dummyHash = await this.hash('password-verification-placeholder');
  }

  hash(password: string) {
    return argon2.hash(password, {
      type: argon2.argon2id, memoryCost: 19_456, timeCost: 2,
      parallelism: 1, hashLength: 32, secret: this.pepper,
    });
  }

  verify(password: string, hash?: string | null) {
    return argon2.verify(hash ?? this.dummyHash, password, { secret: this.pepper });
  }
}
```

Validate a dedicated high-entropy `PASSWORD_PEPPER` in shared config for this peppered design. Store it in the secret store and plan rotation before rollout. Existing unpeppered hashes need a migration before switching verification. Let Argon2 generate a fresh salt per password. Apply independent IP/account budgets before hashing and bound inputs without silently truncating passwords. Never lower the work factor to speed up a controller. Tune the example settings on the deployment hardware. [OWASP password storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)

| Value | Representation and use |
| --- | --- |
| Password | Salted Argon2id hash. Verify with the hasher and avoid sending or logging it. |
| Access, refresh, or reset bearer credential | Full SHA-256 digest of the high-entropy token when persistence or revocation needs comparison. Verify a JWT before using its digest to match state. Use constant-time comparison for in-memory secret-digest checks; atomic datastore equality remains the rotation predicate. |
| IP, normalized login identifier, or validated installation ID | HMAC-SHA-256 with a dedicated secret and versioned, purpose-specific input for rate limiting or scoped pseudonymous identifiers. Store only where needed with a defined TTL. An IP fingerprint is a signal for rate limiting, since NAT, proxies and network changes make it unsuitable as an account credential. |

```ts
// src/common/utils/fingerprint.util.ts
import { createHash, createHmac, timingSafeEqual } from 'node:crypto';

export function tokenFingerprint(token: string): string {
  return createHash('sha256').update(token).digest('hex');
}

export function matchesTokenFingerprint(token: string, storedHex: string): boolean {
  if (!/^[0-9a-f]{64}$/i.test(storedHex)) return false;
  return timingSafeEqual(Buffer.from(tokenFingerprint(token), 'hex'), Buffer.from(storedHex, 'hex'));
}

export function keyedFingerprint(
  key: Buffer, purpose: 'login-ip' | 'login-account' | 'installation', value: string,
): string {
  return createHmac('sha256', key)
    .update(JSON.stringify(['fingerprint-v1', purpose, value]))
    .digest('hex');
}
```

For a login IP key, call `keyedFingerprint(config.rateLimitFingerprintKey, 'login-ip', canonicalIp)` using the trusted-proxy `request.ip` from Fastify. Normalize equivalent address forms for the actual ingress, including mapped IPv4 when present. Account keys use the same normalized identity as credential lookup and a separate purpose. Bind installation fingerprints to the verified owner when used across accounts, for example by fingerprinting `JSON.stringify([userId, installationId])`. Do not build a browser fingerprint from IP or device model by default. Avoid writing raw addresses, credentials, or fingerprint values to request logs. [Node.js crypto](https://nodejs.org/api/crypto.html)

## User and authorization

Use response DTOs/projections; never return a full user persistence object to HTTP. Separate credential reads from public profile reads. Keep roles, blocked/enabled/deleted/verified flags and password fields out of self-service update DTOs. Domain fields and role models come from the product, not a generic enterprise identity system.

```ts
// UserRepository: explicit public projection; preserve existing field names.
findProfile(id: string) {
  return this.prisma.user.findUnique({
    where: { id },
    select: { id: true, username: true, language: true, createdAt: true },
  });
}

// UserService: input is a validated UpdateProfileDto with only these fields.
updateProfile(userId: string, input: UpdateProfileDto) {
  return this.repository.updateProfile(userId, {
    ...(input.username !== undefined ? { username: input.username.trim() } : {}),
    ...(input.language !== undefined ? { language: input.language } : {}),
  });
}
```

Repositories apply owner/tenant predicates to reads and writes, not only guards. Do not treat possession of a resource ID as permission. Prefer deny-by-default protected routes with explicit public login/recovery/health endpoints; respect an existing equivalent mechanism.

```ts
// NotesRepository; userId comes from the verified server context, not body.
findOwned(userId: string, noteId: string) {
  return this.prisma.note.findFirst({
    where: { id: noteId, userId, deletedAt: null },
    select: { id: true, title: true, content: true },
  });
}
```

Use roles only when the product needs them; record ownership does not disappear because a role check exists. Never spread an HTTP body into Prisma update data, even if a TypeScript cast appears to constrain it.

Let `UserService` own account eligibility, reserved names, profile edit cooldowns, language/preferences, and account lifecycle where applicable. Return typed errors and limited DTOs. Use a repository constraint for unique public names and a conditional write for a real cooldown. Use an explicit account kind for guest restrictions. Revoking, disabling or deleting a user must take effect in the authoritative session path. Read [Request boundary](request-boundary.md) for guards and parameter decorators.

## Registration and login

Registration: validate -> normalize identity according to the existing policy -> hash password and prepare session -> create user/session in a shared transaction -> return only after commit. Use DB uniqueness to resolve concurrent registrations; a precheck alone is insufficient. Map the specific identity uniqueness violation, not every database error, to the chosen safe registration response. Do not lowercase passwords or invent provider-specific email rewrites.

Use the service/repository transaction example in [Data and Redis](data-and-redis.md). If a required session is Redis-primary, SQL user creation and session creation are not one atomic transaction; define orphan-user cleanup/retry or explicit login-after-register behavior instead of claiming cross-store atomicity. Email delivery happens after commit, with an honest response/retry contract.

```ts
// AuthService fragment; PasswordService uses its startup dummy hash on a miss.
async login(input: LoginDto, context: AuthContext) {
  await this.loginLimits.check(input.email, context.ipAddress);
  const user = await this.users.findCredentials(normalizeEmail(input.email));
  const matches = await this.passwords.verify(input.password, user?.passwordHash);
  if (!user || !matches || !user.enabled || user.blocked || user.deletedAt) {
    throw new AppException('INVALID_CREDENTIALS');
  }
  return this.sessions.createForUser(user.id, context);
}
```

The credentials repository should select only needed internal fields. AuthService's response exposes no hash. Session creation verifies any required email-verification policy. Use shared Redis rate limits from operations; the example helper names are responsibilities to implement/reuse, not libraries that already exist. Normalize email consistently for storage, lookup and rate-limit keys; product-required case rules must not change during refactors. Generic responses avoid account enumeration; avoid a trivially different fast path for unknown identities.

## Refresh rotation and concurrency

For a DB-owned refresh session, store a hash of a high-entropy refresh secret, session/user association, expiry, absolute lifetime when needed, and revocation state. Never store plaintext bearer credentials in logs or ordinary database columns. A hash of a random high-entropy token is distinct from password hashing.

```ts
// example: token-secret
import { createHash, randomBytes } from 'node:crypto';

export function tokenDigest(raw: string) {
  return createHash('sha256').update(raw).digest('hex');
}
export function newRefreshSecret() {
  const raw = randomBytes(32).toString('base64url');
  return { raw, digest: tokenDigest(raw) };
}
```

An opaque refresh credential can carry a validated session ID plus a random secret; authenticate using the stored digest, not the ID alone. If refresh is JWT, verify signature, allowed algorithm, issuer, audience, expiry and explicit token purpose; do not allow an access token as refresh or vice versa.

```ts
// AuthSessionRepository fragment. nextExpiresAt is capped by the unchanged
// absolute expiry by the service. Pass tx when part of a larger transaction.
async rotate(input: {
  id: string; userId: string; expectedHash: string; nextHash: string;
  now: Date; nextExpiresAt: Date;
}) {
  const result = await this.prisma.authSession.updateMany({
    where: {
      id: input.id, userId: input.userId,
      refreshTokenHash: input.expectedHash, revokedAt: null,
      expiresAt: { gt: input.now }, absoluteExpiresAt: { gt: input.now },
    },
    data: {
      refreshTokenHash: input.nextHash,
      expiresAt: input.nextExpiresAt, lastUsedAt: input.now,
    },
  });
  return result.count === 1;
}
```

Service sequence: verify token/purpose -> load authoritative active session and eligible user -> prepare next credentials -> conditionally rotate using the presented digest -> return next credentials only if the conditional write succeeds. If Redis owns sessions, enforce equivalent hash/state/expiry checks in one atomic Redis operation; do not use GET followed by unconditional SET.

Define replay policy explicitly. A failed conditional write is not permission to accept a stale cached record or overwrite a newer hash. Re-read authoritative state if needed to distinguish expiry, revocation, a concurrent refresh and reuse. For strict one-time rotation, detect validly identified reuse and revoke the affected session/family; clients serialize refresh. For a product requiring parallel refresh, use an existing bounded, tested replay/grace mechanism that returns the same successor without minting independent branches. Do not add a naive grace window that accepts arbitrary old tokens; do not copy encryption/replay machinery unless the concurrency requirement warrants it.

## Recovery and account lifecycle

Recovery requests respond generically whether an account exists. Use unpredictable, short-lived, purpose-bound one-time tokens, stored as digests. Verify permitted link origins; never derive a reset URL from an untrusted Host header. Do not place raw reset credentials in logs or debug API responses.

```ts
// AuthTokenRepository; supplied tx must be used, not a global client.
async consumeReset(hash: string, userId: string, now: Date, tx: Prisma.TransactionClient) {
  const result = await tx.authOneTimeToken.updateMany({
    where: {
      tokenHash: hash, userId, purpose: 'PASSWORD_RESET',
      consumedAt: null, expiresAt: { gt: now },
    },
    data: { consumedAt: now },
  });
  return result.count === 1;
}
```

```ts
// AuthRecoveryService; hash before opening the transaction.
const passwordHash = await this.passwords.hash(input.password);
await this.prisma.$transaction(async (tx) => {
  const token = await this.tokens.findReset(tokenDigest(input.token), tx);
  if (!token || !(await this.tokens.consumeReset(token.tokenHash, token.userId, now, tx))) {
    throw new AppException('INVALID_OR_EXPIRED_TOKEN');
  }
  await this.users.updatePassword(token.userId, passwordHash, tx);
  await this.sessions.revokeAll(token.userId, tx);
});
// Apply the documented Redis revocation/invalidation policy after commit.
```

The lookup is not the single-use guarantee; conditional consumption is. If password/session update fails, token consumption rolls back. Email verification uses its own token purpose and atomic consumption. Password change checks current credentials or recent reauthentication; account deletion and sensitive identity changes follow the product's reauthentication policy.

Account disable/delete resets effective access according to the revocation contract. Decide soft/hard delete, identifier reuse, retention and anonymization from product needs; do not apply soft delete to every table or silently reactivate deleted users on social login. Clean up/revoke relevant sessions, device associations, push tokens and Redis projections without deleting unrelated shared data.

## Session storage and revocation

Before implementation, choose one authoritative strategy and state effective revocation behavior:

| Strategy | Rule |
| --- | --- |
| Redis-primary active sessions | Guards/refresh check authoritative Redis state; absence rejects, outage is explicit unavailable. Define durable recovery and absolute/idle TTLs. Logout removes/revokes the session there. |
| DB sessions plus Redis projection | Define freshness and invalidation failure behavior. Cached active state must not silently outlive a promised immediate revoke. A projection outage never means implicitly authenticated. |
| Stateless access JWT plus stateful refresh | Logout revokes refresh; existing access JWT remains valid until expiry unless a guard checks session/version/revocation state. Never claim instant logout from deleting the refresh record alone. |

For new token APIs needing immediate revocation, prefer a checked authoritative session design. Do not invent a DB/Redis dual-write consistency protocol when a simpler primary-session design meets requirements. If the product accepts bounded access-token validity, state that bound and keep tokens short-lived according to its risk/UX needs. Existing projects retain their contract unless the requested change includes migration.

```ts
// SessionService method for a device-bound, fingerprint-checked access session.
async requireActiveAccess(input: {
  sessionId: string; userId: string; installationId?: string; rawToken: string;
}): Promise<AuthContext> {
  // Returns active state with current account eligibility, or propagates store outage.
  const session = await this.repository.findActiveAccess(input.sessionId);
  if (!session || session.userId !== input.userId || session.accessExpiresAt <= Date.now() ||
      !input.installationId || session.installationId !== input.installationId ||
      !matchesTokenFingerprint(input.rawToken, session.accessFingerprint)) {
    throw new AppException('INVALID_SESSION');
  }
  return { userId: session.userId, sessionId: input.sessionId, role: session.role };
}
```

`findActiveAccess` must enforce revocation and needed current account state. Disable/delete must invalidate the affected active sessions, including any account-status cache. Define races and recovery rather than claiming SQL+Redis atomicity. Avoid a DB read/write on every request purely for lastSeen: throttle activity persistence with Redis only if required and define outage behavior. User-wide revocation can use a user/session index or version if needed; bound cleanup and account for concurrent login/revoke.

## Web and OAuth

Web refresh cookies use HttpOnly, Secure in deployed HTTPS environments, and SameSite/path/domain appropriate to the actual origin topology. Cookie auth needs an explicit CSRF strategy for state-changing requests; CORS is not authentication or a universal CSRF defense. Existing web/mobile transports may share services with separate thin controllers. Do not expose refresh credentials in a JSON response when the web cookie contract intentionally hides them.

```ts
// WebAuthController fragment; cookie options come from validated shared config.
const result = await this.auth.login(input, context);
this.authCookies.setRefresh(response, result.refreshToken);
return { user: result.user, accessToken: result.accessToken };
```

For provider sign-in, use the provider library/JWKS and validate issuer, audience, expiry and required nonce; authorization-code redirects use state and PKCE as applicable. Identify accounts using provider/issuer plus subject, never client-supplied profile fields. Do not merge accounts merely because emails match. Linking an existing identity requires an authenticated, verified linking flow.

```ts
// AuthService; provider.verify validates token claims with the provider SDK.
const identity = await this.google.verify(input.idToken);
return this.socialLogin.authenticate({
  provider: 'google', issuer: identity.issuer,
  subject: identity.subject, verifiedEmail: identity.verifiedEmail,
}, context);
```

The identity repository enforces a unique provider/issuer/subject relation. Product policy controls registration/linking; verifiedEmail is not automatic permission to claim an existing local account. Use the existing provider verifier or its supported production library when the provider flow is requested. Apply a bounded timeout to external exchange/verification calls, and configure allowlisted client IDs for each supported app. Do not add mock authentication bypasses to production routes.

## Device and push tokens

Device identity is untrusted metadata, not proof of authentication. Associate devices using the authenticated user and a validated installation ID. Do not assume one device per OS or one session per device unless required. IP and device headers are not stable credentials.

For device-bound mobile sessions, keep the device row's server ID stable. Store installation ID, owner, platform, model, app version and optional notification settings. Associate the session with the row and verify both owner and installation on protected requests and refresh. Reuse the same installation row across logins. A one-device-per-platform policy needs an explicit replacement and old-session revocation contract. Keep device metadata durable even if Redis is authoritative for active sessions. Use a shared read contract for account/session status and logout, with no implicit SQL fallback for Redis-owned auth state.

```ts
// DeviceRepository; schema has @@unique([ownerId, installationId]).
upsertMetadata(ownerId: string, input: DeviceMetadataDto) {
  return this.prisma.device.upsert({
    where: { ownerId_installationId: { ownerId, installationId: input.installationId } },
    create: {
      ownerId, installationId: input.installationId,
      platform: input.platform, model: input.model, appVersion: input.appVersion,
    },
    update: { platform: input.platform, model: input.model, appVersion: input.appVersion },
    select: { id: true, platform: true, appVersion: true },
  });
}
```

If push is required, define whether an installation/token may be active for more than one account. For a single-owner token policy, enforce ownership with a unique DB constraint (or equivalent atomic Redis mapping), not only clear-others-then-update; parallel requests can race. Handle reassignment from an authenticated device workflow without trusting a userId in the body. Clear stale push mappings on account switches/logout according to product behavior. Process invalid provider tokens conditionally so a delayed provider response does not delete a newer token. Fetch tokens in batches for notification jobs, not one query per recipient. Never treat a push token as an auth credential.

## Behavior checks

Test the paths changed by the task, not every feature in this reference:

- Allowed profile fields; cross-user/tenant read and update denial; safe response projection.
- Registration uniqueness race; failed session creation does not return usable credentials.
- Unknown/wrong/disabled login; limits shared across instances; expired and wrong-purpose tokens.
- Fresh password salts and valid/invalid verification; HMAC stability and purpose isolation; changed token rejected by its stored digest.
- Refresh atomic winner, replay policy, concurrent requests, absolute expiry and revoked users.
- Reset/verification replay and rollback; generic recovery response.
- Logout/revoke-all effect on the actual guard; Redis outage and invalidation failure.
- Device ownership, multiple devices, push reassignment and delayed invalid-token cleanup.

Sources for security details: [OWASP authentication](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html), [password storage](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html), [session management](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html), [OAuth security RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html). These guide security details, not permission to generate every identity feature.
