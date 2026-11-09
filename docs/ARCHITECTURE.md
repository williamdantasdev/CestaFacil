# Arquitetura — CestaFácil

Documento complementar ao [README](../README.md) com detalhes técnicos das decisões.

---

## 1. Feature-first + Clean Architecture (simplificada)

Cada feature é organizada em três camadas com dependências unidirecionais:

```
presentation  →  domain  ←  data
```

- **domain** é o núcleo: entidades + interfaces. Não depende de nada.
- **data** depende de `domain` (implementa interfaces) e de infraestrutura (Firebase, HTTP, Drift).
- **presentation** depende de `domain` (via providers), nunca de `data`.

> Não usamos "use cases" como camada separada enquanto não houver regra de negócio complexa. Quando houver (ex.: cálculo do comparador, match de receitas), mover para `domain/use_cases/` como funções puras testáveis.

---

## 2. Injeção de dependência com Riverpod

Cada repositório existe como interface no domínio. O provider central decide a implementação:

```dart
// core/di/repositories.dart
final productRepositoryProvider = Provider<ProductRepository>((ref) {
  // Fase atual: mock
  return MockProductRepository(ref.watch(mockProductDataSourceProvider));

  // Fase 4 (backend): trocar apenas esta linha
  // return FirebaseProductRepository(ref.watch(firestoreProductDataSourceProvider));
});
```

**Regra**: nenhum arquivo em `presentation/` pode importar implementações concretas. Só a interface.

---

## 3. Datasources

Um repositório pode combinar múltiplos datasources:

```dart
class DefaultProductRepository implements ProductRepository {
  final RemoteProductDataSource remote;   // Firestore ou HTTP
  final LocalProductCache cache;          // Drift
  ...
}
```

Contrato do datasource remoto é independente do backend:

```dart
abstract class RemoteProductDataSource {
  Future<List<ProductDto>> search({required String query, String? category, int page, int pageSize});
  Future<ProductDto?> getById(String id);
  Stream<List<ProductDto>> watch({String? category});
}
```

---

## 4. Tratamento de erros

```dart
sealed class AppException implements Exception {
  const AppException(this.message, {this.cause, this.stackTrace});
  final String message;
  final Object? cause;
  final StackTrace? stackTrace;
}

final class NetworkException extends AppException { ... }
final class AuthException extends AppException { ... }
final class NotFoundException extends AppException { ... }
final class ValidationException extends AppException { ... }
final class UnknownException extends AppException { ... }
```

- Datasources lançam/capturam exceções específicas do backend.
- Repositórios traduzem para `AppException`.
- `presentation` consome via `AsyncValue` (Riverpod) e mostra mensagens localizadas.

---

## 5. Conversão de unidades (`core/units/`)

Toda unidade de produto é um enum com dimensionalidade:

```dart
enum UnitDimension { mass, volume, count }

enum Unit {
  g,        // massa
  kg,
  ml,       // volume
  l,
  un,       // contagem
  dz,
  pack,
  ...
}
```

- Fator para a unidade-base por dimensão: `mass → g`, `volume → ml`, `count → un`.
- Conversão só entre unidades da **mesma dimensão** (kg↔g, L↔ml).
- Comparador de preços só compara unidades da mesma dimensão; caso contrário, sinaliza "não comparável".

Funções:

```dart
double toBase(double value, Unit unit);
double fromBase(double value, Unit target, UnitDimension dimension);
```

---

## 6. Offline-first e sincronização (Fase 4)

- SQLite é a **fonte de leitura da UI**.
- Escritas vão para o SQLite imediatamente e enfileiram um evento de sync.
- `sync_status` por entidade: `local`, `pendingUpload`, `synced`, `conflict`, `rejected`.
- Executor de sync:
  - Escuta mudanças locais → envia ao Firestore.
  - Escuta snapshots do Firestore → aplica ao SQLite (last-write-wins por `updatedAt` + `updatedByUid`).
- Conflitos resolvem por `updatedAt` (MVP); em seguida, merge por campo se necessário.

---

## 7. Contratos dos repositórios (referência)

```dart
abstract class ProductRepository {
  Future<List<Product>> search(ProductSearchQuery query);
  Stream<List<Product>> watchCatalog({String? category});
  Future<Product?> getById(String id);
}

abstract class LocalProductRepository {
  Future<List<LocalProduct>> getAll();
  Future<LocalProduct> create(LocalProduct input);
  Future<void> delete(String localId);
  Future<List<LocalProduct>> listPendingUpload();
  Future<void> markSynced(String localId, String catalogId);
}

abstract class ShoppingListRepository {
  Stream<ShoppingList?> watchList(String listId);
  Future<ShoppingList> createList({required String name, required String ownerUid});
  Future<void> addItem(String listId, ShoppingItem item);
  Future<void> updateItem(String listId, ShoppingItem item);
  Future<void> toggleChecked(String listId, String itemId, {String? byUid});
  Future<void> removeItem(String listId, String itemId);
  Stream<List<ShoppingList>> watchMyLists(String uid);
}

abstract class AuthRepository {
  Stream<AppUser?> watchAuthState();
  Future<AppUser> signInWithEmail(String email, String password);
  Future<AppUser> signUp(String email, String password);
  Future<AppUser> signInWithGoogle();
  Future<void> signOut();
}
```

---

## 8. i18n

- Tudo via `flutter gen-l10n` (arquivos `l10n/app_pt.arb`, `app_en.arb`, ...).
- Locale base: `pt_BR`.
- Datas: `DateFormat.yMd(locale)`.
- Moeda: `NumberFormat.currency(locale: 'pt_BR', symbol: 'R\$')`.
- Unidades: símbolo não traduzido (kg, g, L); nomes por extenso localizados.
- Produtos e receitas carregam `names: Map<String, String>` com fallback para `pt`.

---

## 9. Comparador de preços — algoritmo

Para cada `PriceOffer`:

1. Validar que `size > 0` e `price >= 0`.
2. Converter `(size, unit)` para base: `(baseValue, dimension)`.
3. Custo por unidade base: `price / baseValue`.
4. Agrupar ofertas comparáveis (mesma `dimension`).
5. Ordenar por custo por unidade base.
6. Calcular `percentSavings` vs. oferta mais cara.
7. Sugerir `unitsToMatch = ceil(baseValueMax / baseValueMin)` para igualar volumes.
8. Custo equivalente: `unitsToMatch * cheaper.price`.
9. Sinalizar quando `unitsToMatch * cheaper.price < expensive.price`.

Casos de teste obrigatórios:
- 1 kg R$ 10 vs. 500 g R$ 4,50 → 500 g compensa (2× = R$ 9,00).
- L vs. ml.
- Unidades incompatíveis (kg vs. L) → não comparável.
