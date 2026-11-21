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

Every feature under `src/modules/` has `dto/`, a controller, a service, a repository, and a Nest module definition. Use `errors/` for module-specific errors when needed. A single controller, service, or repository sits at the feature root. On the second file in a layer, group that layer in `controllers/`, `services/`, or `repositories/`. Apply these names in new code and preserve existing imports for revisions outside the layout scope. Put infrastructure providers in `libs/` and scheduler or worker entry points with their runtime code. Domain operations invoked by a worker still belong in their feature module.

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

These fragments assume the project's auth providers and a book model with id, ownerId, title, status, and createdAt. The request DTO is validated by a Nest pipe attached to the body. This works with Nest's core packages and leaves package-based validation available to projects that already use it.

```ts
import { BadRequestException, Injectable, PipeTransform } from '@nestjs/common';

export type CreateBookDto = { title: string };

@Injectable()
export class CreateBookBodyPipe implements PipeTransform<unknown, CreateBookDto> {
  transform(value: unknown): CreateBookDto {
    if (value === null || typeof value !== 'object' || Array.isArray(value)) {
      throw new BadRequestException('Expected an object');
    }
    const fields = Object.keys(value);
    if (fields.length !== 1 || fields[0] !== 'title') {
      throw new BadRequestException('Expected only a title field');
    }
    const { title } = value as Record<string, unknown>;
    if (typeof title !== 'string' || title.length < 1 || title.length > 200) {
      throw new BadRequestException('Title must contain 1 to 200 characters');
    }
    return { title };
  }
}
```

No ownerId, roles, or verification flags in a self-service DTO. Obtain ownership from authenticated context. HTTP update DTOs allow specific fields rather than accepting the ORM's entire update type.

```ts
@UseGuards(JwtAuthGuard)
@Controller('books')
export class BookController {
  constructor(private readonly books: BookService) {}

  @Post()
  create(@CurrentUser() user: AuthUser, @Body(CreateBookBodyPipe) input: CreateBookDto) {
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

Define errors in the owning module and combine the definitions in a typed registry. Services throw an `AppException` with the registered code. The shared HTTP filter resolves and translates it. Use the complete [error and i18n setup](errors-and-i18n.md) for the definitions, catalogs, filter, and response contract.

## Cross-module use and safe revisions

LibraryService may read BookRepository in a batch to annotate search results. It calls BookService when adding a book requires domain rules; do not copy those rules into LibraryService. Export only the required provider.

When extracting a service query: find all callers; preserve return types, failures, ownership, soft-delete/tenant scopes, transaction client and locks; move it to the concrete repository; update injection/module wiring/mocks; verify success and relevant rollback/failure. Do not combine the move with unrelated API or schema redesign.

Official references: [Nest modules](https://docs.nestjs.com/modules), [Nest validation](https://docs.nestjs.com/techniques/validation).
