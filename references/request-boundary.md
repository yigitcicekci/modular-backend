# Request guards, middleware, and decorators

When the API has a mobile/device contract, wire a header guard and expose its validated context through a parameter decorator. For protected JWT routes, wire the access guard and use its verified principal. Add role, guest, version, or provider guards when the route has that policy. Finish the actual wiring and test the routes through AppBuilder.

| Layer | Responsibility |
| --- | --- |
| Fastify hook or middleware | Request ID, safe access logging, transport concerns that do not need route metadata. |
| Guard | Validate client headers, authenticate, enforce a route policy or rate limit. |
| Parameter/metadata decorator | Read validated context for controllers or declare policy for a running guard. |
| Service/repository | Apply account/business rules and enforce ownership in data operations. |

## Shared context and header guard

Put shared mobile guards/decorators in `src/common/`. Keep authentication or provider-specific guards in the owning module. Register `INVALID_HEADERS` as a localized 400, `INVALID_TOKEN` and `INVALID_SESSION` as 401, and `FORBIDDEN` as 403 for this example.

```ts
// src/common/http/request-context.ts
import type { FastifyRequest } from 'fastify';

export type DeviceInfo = {
  installationId: string;
  platform: 'ios' | 'android';
  model: string;
  appVersion: string;
};

export type Role = 'user' | 'admin';
export type AuthContext = { userId: string; sessionId: string; role: Role };
export type ApiRequest = FastifyRequest & { deviceInfo?: DeviceInfo; authUser?: AuthContext };

export function singleHeader(request: ApiRequest, name: string): string | undefined {
  const headers = request.raw.rawHeaders;
  let value: string | undefined;
  for (let index = 0; index < headers.length; index += 2) {
    if (headers[index].toLowerCase() !== name) continue;
    if (value !== undefined) return undefined;
    value = headers[index + 1];
  }
  return value?.trim();
}
```

The example uses Node HTTP/1 Fastify. For another adapter, preserve duplicate-header rejection with its request API. Keep header names and aliases from the current public contract. If compatibility requires aliases, reject conflicting values and write one normalized context. Keep `Accept-Language` available to the i18n resolver. Use a client contract to decide whether a specific app locale is mandatory.

```ts
// src/common/decorators/request-context.decorator.ts
import { createParamDecorator, type ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AppException } from '../exceptions/app.exception';
import type { ApiRequest, Role } from '../http/request-context';

export const SkipHeaderValidation = Reflector.createDecorator<boolean>();
export const Public = Reflector.createDecorator<boolean>();
export const RolesMetadata = Reflector.createDecorator<readonly Role[]>();
export const Roles = (...roles: Role[]) => RolesMetadata(roles);

export const CurrentUser = createParamDecorator((_data: unknown, ctx: ExecutionContext) => {
  const user = ctx.switchToHttp().getRequest<ApiRequest>().authUser;
  if (!user) throw new AppException('INVALID_SESSION');
  return user;
});

export const DeviceInfo = createParamDecorator((_data: unknown, ctx: ExecutionContext) => {
  const device = ctx.switchToHttp().getRequest<ApiRequest>().deviceInfo;
  if (!device) throw new AppException('INVALID_HEADERS');
  return device;
});
```

```ts
// src/common/guards/header-validation.guard.ts
import { Injectable, type CanActivate, type ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { SkipHeaderValidation } from '../decorators/request-context.decorator';
import { AppException } from '../exceptions/app.exception';
import { singleHeader, type ApiRequest } from '../http/request-context';

@Injectable()
export class HeaderValidationGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const targets = [ctx.getHandler(), ctx.getClass()];
    if (this.reflector.getAllAndOverride(SkipHeaderValidation, targets)) return true;

    const request = ctx.switchToHttp().getRequest<ApiRequest>();
    const platform = singleHeader(request, 'os')?.toLowerCase();
    const model = singleHeader(request, 'model');
    const appVersion = singleHeader(request, 'version');
    const installationId = singleHeader(request, 'device-id');
    if ((platform !== 'ios' && platform !== 'android') || !model || model.length > 100 ||
        !appVersion || !/^\d{1,8}(\.\d{1,8}){0,2}$/.test(appVersion) ||
        !installationId || !/^[a-z\d._:-]{1,128}$/i.test(installationId)) {
      throw new AppException('INVALID_HEADERS');
    }

    request.deviceInfo = { platform, model, appVersion, installationId };
    return true;
  }
}
```

These formats describe a sample mobile contract. Adapt them to shipped client identifiers and version format. Bind the guard globally through `APP_GUARD` for a mobile-only API or on the relevant controllers for a mixed API. Mark intentional exceptions, such as health, bootstrap config and provider webhooks, with `@SkipHeaderValidation(true)` where the guard applies. A header exemption leaves the authentication and rate-limit policies independent. Respect client upgrade flows and keep update/version gates separate from credential verification. If Swagger is installed, compose an `@ApiClientHeaders()` decorator from Nest Swagger metadata. Keep docs aligned with the active guard.

## JWT guard and authorization

```ts
// src/modules/authentication/guards/jwt-auth.guard.ts
import { Injectable, type CanActivate, type ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { Public } from '../../../common/decorators/request-context.decorator';
import { AppException } from '../../../common/exceptions/app.exception';
import { singleHeader, type ApiRequest } from '../../../common/http/request-context';
import { SessionService } from '../session.service';
import { TokenService } from '../token.service';

@Injectable()
export class JwtAuthGuard implements CanActivate {
  constructor(
    private readonly reflector: Reflector,
    private readonly tokens: TokenService,
    private readonly sessions: SessionService,
  ) {}

  async canActivate(ctx: ExecutionContext): Promise<boolean> {
    if (this.reflector.getAllAndOverride(Public, [ctx.getHandler(), ctx.getClass()])) return true;
    const request = ctx.switchToHttp().getRequest<ApiRequest>();
    const rawToken = /^Bearer ([^\s,]+)$/i.exec(singleHeader(request, 'authorization') ?? '')?.[1];
    if (!rawToken) throw new AppException('INVALID_TOKEN');

    const claims = await this.tokens.verify(rawToken, 'access');
    request.authUser = await this.sessions.requireActiveAccess({
      sessionId: claims.sid,
      userId: claims.sub,
      installationId: request.deviceInfo?.installationId,
      rawToken,
    });
    return true;
  }
}
```

Implement `requireActiveAccess` as the existing or new domain service. Check session owner, expiry and account eligibility in the authoritative path. For a device-bound session, require the validated matching installation ID. If access fingerprints are used, match the verified token to the stored fingerprint. Attach only an allowed principal projection and propagate required-store outages as typed 503. Stateless access follows its documented revocation window instead. See [JWT and session design](identity.md#jwt-passwords-and-fingerprints).

```ts
// src/modules/authentication/guards/roles.guard.ts
import { Injectable, type CanActivate, type ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { RolesMetadata } from '../../../common/decorators/request-context.decorator';
import { AppException } from '../../../common/exceptions/app.exception';
import type { ApiRequest } from '../../../common/http/request-context';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(ctx: ExecutionContext): boolean {
    const roles = this.reflector.getAllAndOverride(RolesMetadata, [ctx.getHandler(), ctx.getClass()]);
    if (!roles?.length) return true;
    const user = ctx.switchToHttp().getRequest<ApiRequest>().authUser;
    if (!user) throw new AppException('INVALID_SESSION');
    if (!roles.includes(user.role)) throw new AppException('FORBIDDEN');
    return true;
  }
}
```

Prefer deny-by-default for new JWT APIs with private routes. Register header validation before device-bound authentication; use `@Public(true)` only on explicitly public routes. Global JWT auth with device binding needs the header guard at global scope before JWT, with explicit scope metadata in a mixed API. A controller-scoped header guard runs too late for global JWT auth. For controller-scoped policies, `@UseGuards(HeaderValidationGuard, JwtAuthGuard, RolesGuard)` orders the three guards together. `@Roles('admin')` requires a wired `RolesGuard`. Controller arguments use `@CurrentUser() user: AuthContext` and `@DeviceInfo() device: DeviceInfo`. Never populate the principal from `req.body` or a header. An early IP budget can protect header validation and authentication; identity budgets need the verified principal or normalized login input, as appropriate.

A guest guard uses the verified account kind and runs after auth on actions limited to full accounts. A provider webhook guard authenticates the provider using the installed integration, required raw body, signature and replay policy. Its header exemption leaves provider authentication in force. Protect manually triggered jobs with explicit operator authorization. An existing endpoint with optional authentication may deliberately accept a missing credential; invalid supplied credentials still fail.

## Middleware and Fastify hooks

Keep one access log path. Reuse Fastify request logging where configured. For custom summary logging, add a Fastify hook once in AppBuilder before initialization. With Nest middleware on Fastify, use the adapter's raw Node request/response and `MiddlewareConsumer` for shared middleware that needs Nest integration. Route-aware authorization belongs in guards.

```ts
// AppBuilder fragment; logger is the application's configured logger.
const fastify = app.getHttpAdapter().getInstance();
fastify.addHook('onResponse', async (request, reply) => {
  logger.log({
    requestId: request.id,
    method: request.method,
    route: request.routeOptions.url ?? 'unmatched',
    status: reply.statusCode,
    durationMs: reply.elapsedTime,
  });
});
```

Use bounded route templates and controlled request IDs. Keep raw addresses, untrusted forwarding headers, query strings, bodies and credentials out of this log. Use [trusted proxy and rate-limit policy](operations.md#shared-rate-limiting) for client IP resolution. Decorator-driven logging needs an active hook/interceptor and a narrow allowlist of metadata. A cache decorator requires the same explicit key ownership, scoping and outage rules as any other Redis read.

Verify missing, duplicate and invalid headers, explicit exceptions, JWT purpose and expiry, device and owner mismatch, and 401 versus 403. Exercise a protected controller with its actual decorators and check that public exemptions do not skip independent guards. Verify logging on both success and failure.

[Nest guards](https://docs.nestjs.com/guards) · [Nest custom decorators](https://docs.nestjs.com/custom-decorators) · [Nest middleware](https://docs.nestjs.com/middleware)
