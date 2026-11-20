# Architecture

## Contents

- File organization and composition
- DTO/controller/service/repository example
- Typed errors
- Cross-module use and safe revisions

## File organization and composition

```text
src/
  modules/book/
    dto/create-book.dto.ts
    dto/book-response.dto.ts
    errors/book.errors.ts
    book.controller.ts
    book.service.ts
    book.repository.ts
    book.module.ts
  common/exceptions/app.exception.ts
  common/errors/error-registry.ts
  common/filters/http-exception.filter.ts
  libs/prisma/
  libs/redis/
  config/redis-keys.config.ts
  config/redis-ttl.config.ts
  utils/redis-key.util.ts
```

When services multiply, move them into services/; other layers need not move too. Preserve existing service/repository singular naming. No repository for pure computation or empty controller for a worker. Do not generate all example files if the feature does not need them.

```ts
@Module({
  imports: [PrismaModule],
  controllers: [BookController],
  providers: [BookService, BookRepository],
  exports: [BookService],
})
export class BookModule {}
```

Export BookRepository too only when another module needs data-only access. Import the owning module; do not redeclare its providers to bypass exports. Resolve circular ownership before reaching for forwardRef; do not invent an event bus for a simple call.

## DTO/controller/service/repository example

Illustrative fragments assume project providers/decorators and a Book model with id, ownerId, title, status, createdAt. Bind validation once at bootstrap; DTO decorators alone do not enforce validation.

```ts
import { IsString, MaxLength, MinLength } from 'class-validator';

export class CreateBookDto {
  @IsString()
  @MinLength(1)
  @MaxLength(200)
  title!: string;
}
```

No ownerId, roles, or verification flags in a self-service DTO. Obtain ownership from authenticated context. HTTP update DTOs allow specific fields rather than accepting the ORM's entire update type.

```ts
@UseGuards(JwtAuthGuard)
@Controller('books')
export class BookController {
  constructor(private readonly books: BookService) {}

  @Post()
  create(@CurrentUser() user: AuthUser, @Body() input: CreateBookDto) {
    return this.books.create(user.id, input);
  }

  @Patch(':id/archive')
  archive(
    @CurrentUser() user: AuthUser,
    @Param('id', ParseUUIDPipe) id: string,
  ) {
    return this.books.archive(user.id, id);
  }
}
```

```ts
@Injectable()
export class BookService {
  constructor(private readonly repository: BookRepository) {}

  create(ownerId: string, input: CreateBookDto) {
    const title = input.title.trim();
    if (!title) throw new AppException('BOOK_TITLE_REQUIRED');
    return this.repository.create({ ownerId, title });
  }

  async archive(ownerId: string, id: string) {
    if (!(await this.repository.archiveActive(ownerId, id))) {
      throw new AppException('BOOK_NOT_ACTIVE_OR_NOT_FOUND');
    }
    return { id, status: 'ARCHIVED' as const };
  }
}
```

```ts
@Injectable()
export class BookRepository {
  constructor(private readonly prisma: PrismaService) {}

  create(data: { ownerId: string; title: string }) {
    return this.prisma.book.create({
      data,
      select: { id: true, title: true, status: true, createdAt: true },
    });
  }

  async archiveActive(ownerId: string, id: string): Promise<boolean> {
    const result = await this.prisma.book.updateMany({
      where: { id, ownerId, status: 'ACTIVE' },
      data: { status: 'ARCHIVED' },
    });
    return result.count === 1;
  }
}
```

Service chooses the transition; repository enforces its predicate atomically. Read-then-unconditional-update can race. The combined failure deliberately avoids revealing another user's record. If the product needs distinct already-archived/missing errors or idempotent archive, implement that contract without losing ownership or atomicity. Preserve existing response behavior during refactors.

## Typed errors

Reuse the project's error mechanism. For a new project, this combined example is enough; put domain errors in the module and the registry/exception in common.

```ts
import { HttpException, HttpStatus } from '@nestjs/common';

type ErrorDefinition = { status: HttpStatus; message: string };
export const BookErrors = {
  BOOK_TITLE_REQUIRED: {
    status: HttpStatus.BAD_REQUEST, message: 'Book title is required',
  },
  BOOK_NOT_ACTIVE_OR_NOT_FOUND: {
    status: HttpStatus.NOT_FOUND, message: 'Active book not found',
  },
} as const satisfies Record<string, ErrorDefinition>;

export const ErrorRegistry = { ...BookErrors } as const;
export type ErrorCode = keyof typeof ErrorRegistry;

export class AppException extends HttpException {
  constructor(code: ErrorCode) {
    const { status, message } = ErrorRegistry[code];
    super({ code, message }, status);
  }
}
```

Preserve messageKey localization if present. Prefix codes by domain; check added registry keys for collisions. No subclass per error. The global filter preserves the error envelope, maps validation and known infrastructure errors, and responds generically to unknown failures while logging safe context. Do not add catch blocks throughout services merely to rethrow.

## Cross-module use and safe revisions

LibraryService may read BookRepository in a batch to annotate search results. It calls BookService when adding a book requires domain rules; do not copy those rules into LibraryService. Export only the required provider.

When extracting a service query: find all callers; preserve return types, failures, ownership, soft-delete/tenant scopes, transaction client and locks; move it to the concrete repository; update injection/module wiring/mocks; verify success and relevant rollback/failure. Do not combine the move with unrelated API or schema redesign.

Official references: [Nest modules](https://docs.nestjs.com/modules), [Nest validation](https://docs.nestjs.com/techniques/validation).
