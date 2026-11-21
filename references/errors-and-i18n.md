# Errors and i18n

Keep the public error shape `{ status, code, message }` for new HTTP APIs. `status` is the numeric HTTP status and also sets the response status. `code` is a stable identifier for client decisions. `message` is translated for the request. Keep domain definitions under `src/modules/<feature>/errors/` and catalogs under `src/i18n/locales/<locale>/`. Services and guards throw typed errors. The global filter localizes and serializes them once.

## Definitions and exception

```ts
// src/common/errors/common.errors.ts
import { HttpStatus } from '@nestjs/common';

export type ErrorDefinition = {
  code: string;
  status: number;
  messageKey: `errors.${string}`;
};

export const CommonErrors = {
  BAD_REQUEST: { code: 'BAD_REQUEST', status: HttpStatus.BAD_REQUEST, messageKey: 'errors.common.bad_request' },
  UNAUTHORIZED: { code: 'UNAUTHORIZED', status: HttpStatus.UNAUTHORIZED, messageKey: 'errors.common.unauthorized' },
  FORBIDDEN: { code: 'FORBIDDEN', status: HttpStatus.FORBIDDEN, messageKey: 'errors.common.forbidden' },
  ROUTE_NOT_FOUND: { code: 'ROUTE_NOT_FOUND', status: HttpStatus.NOT_FOUND, messageKey: 'errors.common.route_not_found' },
  PAYLOAD_TOO_LARGE: { code: 'PAYLOAD_TOO_LARGE', status: HttpStatus.PAYLOAD_TOO_LARGE, messageKey: 'errors.common.payload_too_large' },
  RATE_LIMITED: { code: 'RATE_LIMITED', status: HttpStatus.TOO_MANY_REQUESTS, messageKey: 'errors.common.rate_limited' },
  REDIS_UNAVAILABLE: { code: 'REDIS_UNAVAILABLE', status: HttpStatus.SERVICE_UNAVAILABLE, messageKey: 'errors.common.service_unavailable' },
  RATE_LIMIT_UNAVAILABLE: { code: 'RATE_LIMIT_UNAVAILABLE', status: HttpStatus.SERVICE_UNAVAILABLE, messageKey: 'errors.common.service_unavailable' },
  HTTP_CLIENT_ERROR: { code: 'HTTP_CLIENT_ERROR', status: HttpStatus.BAD_REQUEST, messageKey: 'errors.common.client_error' },
  INTERNAL_SERVER_ERROR: {
    code: 'INTERNAL_SERVER_ERROR',
    status: HttpStatus.INTERNAL_SERVER_ERROR,
    messageKey: 'errors.common.internal_server_error',
  },
} as const satisfies Record<string, ErrorDefinition>;
```

```ts
// src/modules/book/errors/book.errors.ts
import { HttpStatus } from '@nestjs/common';
import type { ErrorDefinition } from '../../../common/errors/common.errors';

export const BookErrors = {
  BOOK_TITLE_REQUIRED: {
    code: 'BOOK_TITLE_REQUIRED', status: HttpStatus.BAD_REQUEST,
    messageKey: 'errors.book.title_required',
  },
  BOOK_NOT_ACTIVE_OR_NOT_FOUND: {
    code: 'BOOK_NOT_ACTIVE_OR_NOT_FOUND', status: HttpStatus.NOT_FOUND,
    messageKey: 'errors.book.not_found',
  },
} as const satisfies Record<string, ErrorDefinition>;
```

```ts
// src/common/errors/error-registry.ts
import { BookErrors } from '../../modules/book/errors/book.errors';
import { CommonErrors } from './common.errors';

export const ErrorRegistry = { ...CommonErrors, ...BookErrors } as const;
export type ErrorCode = keyof typeof ErrorRegistry;
```

```ts
// src/common/exceptions/app.exception.ts
import { HttpException } from '@nestjs/common';
import { ErrorRegistry, type ErrorCode } from '../errors/error-registry';

export class AppException extends HttpException {
  constructor(
    public readonly code: ErrorCode,
    public readonly args?: Record<string, string | number>,
  ) {
    super(code, ErrorRegistry[code].status);
  }
}
```

In the service, use `throw new AppException(BookErrors.BOOK_NOT_ACTIVE_OR_NOT_FOUND.code)`. Domain prefixes prevent collisions. Verify registry keys equal their definitions' `code` and that spreads introduce no duplicate keys. Pass only approved public values as translation `args`. Keep provider metadata, validation internals, and exception text out of interpolation. Map known data failures where the domain knows their meaning, such as a targeted uniqueness conflict. Unexpected failures reach the filter as a generic 500.

## Locale config and catalogs

Reuse an existing i18n setup. For the Nest example, use `nestjs-i18n` with `AcceptLanguageResolver`, a configured default locale, and JSON catalogs copied into the compiled application. Install only the package and its missing required peers, including `class-validator` where required by the selected version. The resolver uses supported languages to negotiate the request. Retain an established locale precedence during revisions. Localize framework failures using the request locale when the adapter has resolved it and the default before locale resolution.

```ts
// src/config/i18n.config.ts
import { join } from 'node:path';
import { AcceptLanguageResolver, type I18nOptions } from 'nestjs-i18n';

export const i18nConfig = {
  fallbackLanguage: 'en',
  loaderOptions: { path: join(__dirname, '../i18n/locales'), watch: false },
  resolvers: [AcceptLanguageResolver],
} satisfies I18nOptions;
```

For Nest CLI builds, add `{ "include": "i18n/locales/**/*" }` to `compilerOptions.assets` in `nest-cli.json`. Other builds must copy the same tree relative to the compiled config. Include the assets in the deployed image and run the translation checks against the compiled app.

```json
{
  "common": {
    "bad_request": "Invalid request.",
    "unauthorized": "Authentication required.",
    "forbidden": "Access denied.",
    "route_not_found": "Route not found.",
    "payload_too_large": "Request body is too large.",
    "rate_limited": "Too many requests. Try again later.",
    "service_unavailable": "Service temporarily unavailable.",
    "client_error": "The request could not be completed.",
    "internal_server_error": "An unexpected error occurred."
  },
  "book": {
    "title_required": "A book title is required.",
    "not_found": "Active book not found."
  }
}
```

Place the example in `src/i18n/locales/en/errors.json`. Give each supported locale its own `errors.json` with the same keys. For example, the Turkish `book.not_found` value is `Etkin kitap bulunamadı.` Check catalog coverage for every registered `messageKey` and supported locale, with the default catalog required. A translation key is an internal identifier. Use the configured fallback during resolution and guard against a raw key leaking if catalogs drift.

## One global Nest Fastify filter

```ts
// src/common/filters/http-exception.filter.ts
import { ArgumentsHost, Catch, ExceptionFilter, HttpException, Logger } from '@nestjs/common';
import type { FastifyReply, FastifyRequest } from 'fastify';
import { I18nContext, I18nService } from 'nestjs-i18n';
import { CommonErrors } from '../errors/common.errors';
import { ErrorRegistry, type ErrorCode } from '../errors/error-registry';
import { AppException } from '../exceptions/app.exception';

const frameworkCodes: Partial<Record<number, ErrorCode>> = {
  400: 'BAD_REQUEST',
  401: 'UNAUTHORIZED',
  403: 'FORBIDDEN',
  404: 'ROUTE_NOT_FOUND',
  413: 'PAYLOAD_TOO_LARGE',
  429: 'RATE_LIMITED',
};

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  constructor(private readonly i18n: I18nService) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest<FastifyRequest>();
    const reply = ctx.getResponse<FastifyReply>();
    const domainError = exception instanceof AppException ? exception : undefined;
    const status = exception instanceof HttpException ? exception.getStatus() : 500;
    const fallbackCode = status >= 500 ? 'INTERNAL_SERVER_ERROR' : 'HTTP_CLIENT_ERROR';
    const { code, messageKey } = ErrorRegistry[domainError?.code ?? frameworkCodes[status] ?? fallbackCode];

    const lang = I18nContext.current(host)?.lang;
    const translated = this.i18n.translate(messageKey, { lang, args: domainError?.args });
    const message = translated === messageKey
      ? this.i18n.translate(CommonErrors[fallbackCode].messageKey, {
          lang, defaultValue: 'The request could not be completed.',
        })
      : translated;

    if (status >= 500) this.logger.error({ requestId: request.id, code, status });
    reply.code(status).send({ status, code, message });
  }
}
```

```ts
// src/app.module.ts
import { Module } from '@nestjs/common';
import { APP_FILTER } from '@nestjs/core';
import { I18nModule } from 'nestjs-i18n';
import { HttpExceptionFilter } from './common/filters/http-exception.filter';
import { i18nConfig } from './config/i18n.config';

@Module({
  imports: [I18nModule.forRoot(i18nConfig)], // Also import the feature modules.
  providers: [{ provide: APP_FILTER, useClass: HttpExceptionFilter }],
})
export class AppModule {}
```

If the existing AppBuilder installs the filter with `app.useGlobalFilters(new HttpExceptionFilter(app.get(I18nService)))`, keep that single registration. Use a catch-all filter only for the HTTP transport. In a mixed-transport app, give other transports their own error handling. For direct Node/Fastify, use one `setErrorHandler` and a `setNotFoundHandler` that uses the same serializer and translation catalogs. Preserve known framework statuses on the installed adapter and explicitly normalize native parser failures if its Nest version does not convert them to `HttpException`.

Return a localized `BAD_REQUEST` for built-in DTO validation failures. If field-level details are an established contract, use validated field names and localized constraints through an explicit compatibility mapping. Preserve protocol headers such as `Retry-After` set by the limiter. Framework 401 and 403 keep their status. Redis failures that make a required feature unavailable use a registered 503 error. Existing published envelopes can use the same registry, status and localization internally while keeping client field names until migration.

## Verify the contract

Use the real AppBuilder and Fastify `inject()`. For the book example, assert HTTP 404 and the exact body `{"status":404,"code":"BOOK_NOT_ACTIVE_OR_NOT_FOUND","message":"Etkin kitap bulunamadı."}` under `Accept-Language: tr`. Repeat in English and with an unsupported locale. Check validation 400, auth 401/403, unmatched route 404, oversized body 413, rate limit 429 with `Retry-After`, required dependency 503, and unexpected 500. Assert that framework failures retain their HTTP status and the response omits provider messages, stacks, request data, and translation keys.

See [Nest exception filters](https://docs.nestjs.com/exception-filters), [nestjs-i18n setup](https://nestjs-i18n.com/quick-start), and [locale fallbacks](https://nestjs-i18n.com/guides/fallback-languages).
