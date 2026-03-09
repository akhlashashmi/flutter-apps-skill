# Drift Persistence

Drift is a reactive SQLite toolkit for Dart and Flutter. This guide reflects Drift docs and package releases available in March 2026.

## Core Stack (March 2026)

| Package | Version | Purpose |
|---------|---------|---------|
| drift | 2.32.0+ | ORM, type-safe queries, streams |
| drift_dev | 2.32.0+ | Code generation, schema tools |
| drift_flutter | 0.3.0+ | Flutter database bootstrap helpers |
| sqlite3_flutter_libs | latest | Native SQLite binaries for mobile/desktop |
| build_runner | latest | Generates Drift and Riverpod files |

## Why Drift

- **Type-safe SQL and Dart APIs** with compile-time validation.
- **Reactive streams** for UI updates without manual event wiring.
- **Scalable schema tooling** (generated migration steps + tests).
- **Cross-platform support** for mobile, desktop, and web.

## Scalable Organization

Keep Drift code in the feature `data` layer, but keep one app-level database entrypoint.

```text
lib/
  core/
    database/
      app_database.dart
      app_database_provider.dart
  features/
    products/
      data/
        persistence/
          drift/
            tables/
              products_table.dart
            daos/
              product_dao.dart
            queries/
              product_search.drift
```

### Structure Rules

1. **One table per file** (`*_table.dart`).
2. **One DAO per aggregate/use-case area**, not one mega-DAO.
3. **Complex joins/search SQL in `.drift` files** to keep Dart DAOs small.
4. **Repository layer maps DB models to domain entities**. Do not leak Drift row classes to presentation.
5. **Keep app-wide DB wiring in `core/database`** and feature logic in feature folders.

## Table Pattern

`products_table.dart`

```dart
import 'package:drift/drift.dart';

class Products extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get name => text().withLength(min: 1, max: 100)();
  IntColumn get quantity => integer().withDefault(const Constant(0))();
  DateTimeColumn get updatedAt => dateTime()();
}
```

## DAO Pattern

`product_dao.dart`

```dart
import 'package:drift/drift.dart';

import 'package:my_app/core/database/app_database.dart';
import '../tables/products_table.dart';

part 'product_dao.g.dart';

@DriftAccessor(tables: [Products])
class ProductDao extends DatabaseAccessor<AppDatabase> with _$ProductDaoMixin {
  ProductDao(super.db);

  Stream<List<Product>> watchAll() =>
      (select(products)..orderBy([(t) => OrderingTerm.desc(t.updatedAt)])).watch();

  Future<List<Product>> page({required int limit, required int offset}) {
    return (select(products)
          ..orderBy([(t) => OrderingTerm.desc(t.updatedAt)])
          ..limit(limit, offset: offset))
        .get();
  }

  Future<int> upsert(ProductsCompanion row) {
    return into(products).insertOnConflictUpdate(row);
  }

  Future<void> deleteById(int id) {
    return (delete(products)..where((t) => t.id.equals(id))).go();
  }
}
```

## Database Entrypoint (Flutter)

`app_database.dart`

```dart
import 'package:drift/drift.dart';
import 'package:drift_flutter/drift_flutter.dart';

import 'package:my_app/features/products/data/persistence/drift/daos/product_dao.dart';
import 'package:my_app/features/products/data/persistence/drift/tables/products_table.dart';

part 'app_database.g.dart';

@DriftDatabase(tables: [Products], daos: [ProductDao])
class AppDatabase extends _$AppDatabase {
  AppDatabase([QueryExecutor? executor]) : super(executor ?? _openConnection());

  @override
  int get schemaVersion => 1;

  static QueryExecutor _openConnection() => driftDatabase(name: 'app_db');
}
```

## Riverpod Providers

`app_database_provider.dart`

```dart
import 'package:riverpod_annotation/riverpod_annotation.dart';

import 'app_database.dart';

part 'app_database_provider.g.dart';

@Riverpod(keepAlive: true)
AppDatabase appDatabase(Ref ref) {
  final db = AppDatabase();
  ref.onDispose(db.close);
  return db;
}

@Riverpod(keepAlive: true)
ProductDao productDao(Ref ref) {
  return ref.watch(appDatabaseProvider).productDao;
}
```

## Code Generation

```bash
dart run build_runner build --delete-conflicting-outputs
```

Use watch mode during development:

```bash
dart run build_runner watch --delete-conflicting-outputs
```

## Migration Workflow (Recommended for Production)

1. Add `schemaVersion` bump and migration logic in `AppDatabase`.
2. Configure `build.yaml` with a named database.
3. Run Drift migration generator.
4. Commit generated migration steps and tests.

`build.yaml` example:

```yaml
targets:
  $default:
    builders:
      drift_dev:
        options:
          databases:
            app_db: lib/core/database/app_database.dart
          schema_dir: test/drift/schemas/
          test_dir: test/drift/
```

Generate migration steps and tests:

```bash
dart run drift_dev make-migrations
```

### Minimal Migration Strategy

```dart
@override
MigrationStrategy get migration => MigrationStrategy(
      onCreate: (m) async => m.createAll(),
      onUpgrade: (m, from, to) async {
        if (from < 2) {
          await m.addColumn(products, products.updatedAt);
        }
      },
      beforeOpen: (details) async {
        await customStatement('PRAGMA foreign_keys = ON');
      },
    );
```

## Modular Code Generation (Large Apps)

For large codebases, Drift recommends modular generation to reduce rebuild scope and keep generated files colocated.

When using modular generation:

1. Disable the default `drift_dev` builder and enable `drift_dev:modular`.
2. Remove `part` statements from Drift files.
3. Import generated `.drift.dart` files directly.
4. Update generated base class names from `_$ClassName` to `$ClassName`.

`build.yaml` starter:

```yaml
targets:
  $default:
    builders:
      drift_dev:
        enabled: false
      drift_dev:analyzer:
        enabled: true
        options: &drift_options
          sql:
            dialect: sqlite
      drift_dev:modular:
        enabled: true
        options: *drift_options
```

Adopt this once schema growth makes full project regeneration noticeably slow.

## Testing

Use an in-memory database with the same DAO APIs:

```dart
import 'package:drift/native.dart';
import 'package:flutter_test/flutter_test.dart';

import 'package:my_app/core/database/app_database.dart';

void main() {
  late AppDatabase db;
  late ProductDao dao;

  setUp(() {
    db = AppDatabase(NativeDatabase.memory());
    dao = db.productDao;
  });

  tearDown(() async {
    await db.close();
  });

  test('upsert then read', () async {
    await dao.upsert(
      ProductsCompanion.insert(
        name: 'Notebook',
        quantity: const Value(3),
        updatedAt: DateTime(2026, 3, 6),
      ),
    );

    final rows = await dao.page(limit: 10, offset: 0);
    expect(rows, isNotEmpty);
  });
}
```

## Critical Rules

1. **Never skip migration tests** for production schema changes.
2. **Keep DAO methods use-case oriented** (`watchCart`, `upsertInventory`) rather than generic dumps.
3. **Use transactions for multi-step writes** to avoid partial updates.
4. **Prefer stream queries for live UI screens** and one-shot futures for commands.
5. **Do not expose Drift row classes outside data layer**.
6. **Close database instances** in tests and app shutdown paths.

## References

- [Drift docs](https://drift.simonbinder.eu/)
- [Drift setup guide](https://drift.simonbinder.eu/setup/)
- [Drift migrations](https://drift.simonbinder.eu/migrations/)
- [Drift modular code generation](https://drift.simonbinder.eu/generation_options/modular/)
- [drift on pub.dev](https://pub.dev/packages/drift)
- [drift_flutter on pub.dev](https://pub.dev/packages/drift_flutter)
