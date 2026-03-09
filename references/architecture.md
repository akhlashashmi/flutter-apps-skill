# Architecture

Use a feature-first layered architecture. MVVM applies inside each feature's `presentation/` layer, not across the entire project.

## What MVVM Means Here

- `presentation/screens/` and `presentation/widgets/` are the View.
- `presentation/notifiers/` are the ViewModel.
- `domain/`, `repositories/`, and `data/` together make up the Model side of the feature.

This is not a strict three-folder MVVM app. It is a feature-first structure that uses MVVM for UI state while keeping dedicated repository, domain, and data layers for scalability.

## Dependency Rules

Keep imports flowing inward:

- `presentation/` may depend on `repositories/` and `domain/`
- `repositories/` may depend on `data/` and `domain/`
- `data/` may depend on `domain/` for mapping targets, but never on `presentation/`
- `domain/` should stay UI-agnostic and free of Flutter imports

Keep UI error state in the ViewModel. Repositories and datasources should usually throw typed errors upward instead of converting them into UI state.

## Full Directory Structure

```text
lib/
|-- core/
|   |-- config/
|   |   `-- app_config.dart              # Environment variables, API URLs
|   |-- domain/
|   |   `-- errors/
|   |       `-- app_error.dart           # Shared cross-feature error types
|   |-- extensions/
|   |   |-- extensions.dart              # Barrel export for all extensions
|   |   |-- context_extensions.dart      # Theme, media, breakpoints, feedback
|   |   |-- string_extensions.dart       # capitalize, truncate, initials
|   |   |-- date_time_extensions.dart    # timeAgo, isToday, startOfDay
|   |   |-- iterable_extensions.dart     # firstWhereOrNull, groupBy
|   |   `-- widget_extensions.dart       # separatedBy
|   |-- navigation/
|   |   |-- routes.dart                  # Typed GoRouter route classes
|   |   |-- routes.g.dart
|   |   `-- router_provider.dart         # GoRouter provider with auth redirect
|   |-- services/
|   |   |-- http_service.dart            # HTTP client wrapper
|   |   |-- storage_service.dart         # Local persistence helpers
|   |   `-- database_service.dart
|   |-- theme/
|   |   |-- app_colors.dart
|   |   |-- spacing.dart                 # Spacing constants
|   |   |-- radii.dart                   # Border radius constants
|   |   `-- icon_sizes.dart
|   |-- utils/
|   |   |-- batch_utils.dart             # Parallel batch processing
|   |   |-- debouncer.dart               # Timer-based debouncer
|   |   |-- snack_bar_utils.dart         # Centralized feedback helpers
|   |   |-- validators.dart              # Form validation functions
|   |   `-- date_formatter.dart
|   `-- widgets/
|       |-- atoms/                       # Buttons, badges, indicators
|       |-- molecules/                   # Avatar tiles, stat cards
|       |-- organisms/                   # Data grids, navigation headers
|       `-- templates/                   # Dashboard layouts, list-detail
|-- features/
|   |-- auth/
|   |   |-- data/
|   |   |   |-- datasources/
|   |   |   |   `-- auth_remote_datasource.dart
|   |   |   `-- models/
|   |   |       `-- auth_model.dart
|   |   |-- domain/
|   |   |   `-- entities/
|   |   |       `-- user.dart
|   |   |-- repositories/
|   |   |   `-- auth_repository.dart
|   |   `-- presentation/
|   |       |-- notifiers/
|   |       |   `-- auth_notifier.dart
|   |       |-- screens/
|   |       |   `-- login_screen.dart
|   |       `-- widgets/
|   |           `-- login_form.dart
|   |-- products/
|   |   |-- data/
|   |   |   |-- datasources/
|   |   |   |   |-- product_remote_datasource.dart
|   |   |   |   `-- product_local_datasource.dart
|   |   |   `-- models/
|   |   |       `-- product_model.dart
|   |   |-- domain/
|   |   |   `-- entities/
|   |   |       `-- product.dart
|   |   |-- repositories/
|   |   |   `-- product_repository.dart
|   |   `-- presentation/
|   |       |-- notifiers/
|   |       |   `-- product_notifier.dart
|   |       |-- screens/
|   |       |   |-- product_list_screen.dart
|   |       |   `-- product_detail_screen.dart
|   |       `-- widgets/
|   |           |-- product_card.dart
|   |           `-- product_filter.dart
|   `-- home/
|       `-- presentation/
|           |-- notifiers/
|           |   `-- home_notifier.dart
|           |-- screens/
|           |   `-- home_screen.dart
|           `-- widgets/
|               `-- home_section.dart
`-- main.dart
```

Not every feature needs every folder on day one. A simple feature may start with `presentation/` only and grow into `repositories/`, `domain/`, and `data/` when complexity justifies it.

## Layer Responsibilities

### Presentation Layer

This is the MVVM layer.

- Views live in `presentation/screens/` and `presentation/widgets/`
- ViewModels live in `presentation/notifiers/`
- Views watch provider state and send user actions to notifiers
- ViewModels call repositories, map failures into UI state, and expose immutable state objects

See [state-management.md](state-management.md) for full notifier patterns.

```dart
// features/products/presentation/notifiers/product_notifier.dart
@freezed
sealed class ProductState with _$ProductState {
  const factory ProductState({
    @Default([]) List<Product> items,
    @Default(false) bool isLoading,
    String? error,
  }) = _ProductState;
}

@Riverpod(keepAlive: true)
class ProductNotifier extends _$ProductNotifier {
  @override
  ProductState build() {
    _load();
    return const ProductState();
  }

  Future<void> _load() async {
    state = state.copyWith(isLoading: true, error: null);
    try {
      final items = await ref.read(productRepositoryProvider).fetchAll();
      if (!ref.mounted) return;
      state = state.copyWith(items: items, isLoading: false);
    } catch (e) {
      if (!ref.mounted) return;
      state = state.copyWith(isLoading: false, error: e.toString());
    }
  }
}
```

```dart
// features/products/presentation/screens/product_list_screen.dart
class ProductListScreen extends ConsumerWidget {
  const ProductListScreen({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final isLoading = ref.watch(
      productProvider.select((s) => s.isLoading),
    );

    if (isLoading) {
      return const Center(child: CircularProgressIndicator());
    }

    return const ProductListView();
  }
}

class ProductListView extends ConsumerWidget {
  const ProductListView({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final items = ref.watch(
      productProvider.select((s) => s.items),
    );

    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) => ProductCard(id: items[index].id),
    );
  }
}
```

### Repository Layer

Repositories are the feature boundary between ViewModels and raw data access.

- Read from one or more datasources
- Write to cache or persistence layers
- Map data models into domain entities
- Expose feature-focused methods such as `fetchAll`, `save`, `delete`, or `sync`

Repositories should not hold widget state or UI flags like `isLoading`. Keep them focused on business data flow.

```dart
// features/products/repositories/product_repository.dart
@Riverpod(keepAlive: true)
ProductRepository productRepository(Ref ref) {
  return ProductRepository(
    ref.read(productRemoteDatasourceProvider),
    ref.read(productLocalDatasourceProvider),
  );
}

class ProductRepository {
  ProductRepository(this._remote, this._local);

  final ProductRemoteDatasource _remote;
  final ProductLocalDatasource _local;

  Future<List<Product>> fetchAll() async {
    final cached = await _local.getAll();
    if (cached.isNotEmpty) {
      return cached.map((m) => m.toEntity()).toList();
    }

    final models = await _remote.fetchAll();
    await _local.cacheAll(models);
    return models.map((m) => m.toEntity()).toList();
  }

  Future<Product> fetchById(String id) async {
    final model = await _remote.fetchById(id);
    return model.toEntity();
  }
}
```

### Domain Layer

Domain types define the business shape of the feature.

- Use pure Dart entities and value objects
- Avoid Flutter imports
- Keep UI formatting, JSON, and storage concerns out of this layer
- Put derived business behavior on the entity itself when it comes from the entity's own fields

See [freezed-sealed.md](freezed-sealed.md#rich-models) for rich model guidance.

```dart
// features/products/domain/entities/product.dart
@freezed
sealed class Product with _$Product {
  const Product._();

  const factory Product({
    required String id,
    required String name,
    required double price,
    @Default(0) int quantity,
    @Default(true) bool isActive,
  }) = _Product;

  double get totalValue => price * quantity;
  bool get inStock => quantity > 0;
}
```

Entities should not contain `fromJson` or `toJson`. That belongs in the Data layer.

### Data Layer

The Data layer deals with external representations.

- Models handle JSON and storage serialization
- Datasources perform API, database, and local cache operations
- Mapping helpers convert data models into domain entities

```dart
// features/products/data/models/product_model.dart
@freezed
sealed class ProductModel with _$ProductModel {
  const ProductModel._();

  const factory ProductModel({
    required String id,
    required String name,
    required double price,
    @Default(0) int quantity,
    @JsonKey(name: 'is_active') @Default(true) bool isActive,
  }) = _ProductModel;

  factory ProductModel.fromJson(Map<String, dynamic> json) =>
      _$ProductModelFromJson(json);

  Product toEntity() => Product(
        id: id,
        name: name,
        price: price,
        quantity: quantity,
        isActive: isActive,
      );

  Map<String, dynamic> toNameOnlyRequestBody() => {
        'id': id,
        'name': name,
      };
}
```

```dart
// features/products/data/datasources/product_remote_datasource.dart
@Riverpod(keepAlive: true)
ProductRemoteDatasource productRemoteDatasource(Ref ref) {
  return ProductRemoteDatasource(ref.read(httpServiceProvider));
}

class ProductRemoteDatasource {
  ProductRemoteDatasource(this._http);

  final HttpService _http;

  Future<List<ProductModel>> fetchAll() async {
    final response = await _http.get('/products');
    return (response as List)
        .map((json) => ProductModel.fromJson(json))
        .toList();
  }

  Future<ProductModel> fetchById(String id) async {
    final json = await _http.get('/products/$id');
    return ProductModel.fromJson(json);
  }

  Future<void> create(ProductModel model) async {
    await _http.post('/products', body: model.toJson());
  }
}
```

## Complexity Tiers

| Tier | Data | Auth | Example | Implementation |
|------|------|------|---------|----------------|
| 1 | Simple, no PII | None | To-do lists, notes | `presentation/` + one repository, optional Hive |
| 2 | Public or user-owned data | Basic | Social, catalogs | Full feature structure with remote/local datasources |
| 3 | PII or regulated workflows | Full | Banking, health | Full structure, typed errors, stronger validation and auditing |

Start with Tier 2 for most production apps. Simplify to Tier 1 only for trivial features. Use Tier 3 for regulated or high-risk domains.

## Design Tokens

Do not hardcode spacing, colors, radii, or icon sizes. See [atomic-design.md](atomic-design.md) for token definitions such as `Spacing`, `Radii`, `IconSizes`, typography, `ColorScheme`, and semantic colors.

```dart
Padding(padding: const EdgeInsets.all(Spacing.s16))
Text('Title', style: Theme.of(context).textTheme.titleMedium)
Container(color: Theme.of(context).colorScheme.primary)
```

## Atomic Design for Widgets

Shared widgets in `core/widgets/` follow atomic design:

- atoms
- molecules
- organisms
- templates

See [atomic-design.md](atomic-design.md) for placement rules and examples.

Feature-specific widgets belong in `features/x/presentation/widgets/`, not `core/widgets/`.
