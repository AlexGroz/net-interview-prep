# Подготовка к собеседованию (.NET) — конспекты

Разборы вопросов по резюме, уровень **продвинутый middle**. Веб-версия (удобно с телефона):

- **[Вопрос 1 — Высокая пропускная способность и низкая задержка в .NET](q1-high-load.html)**
- **[Вопрос 2 — Миграция DI и жизненных циклов: .NET Framework → .NET Core](q2-di-lifetimes.html)**

Ниже — те же материалы в формате Markdown.

---

# Вопрос 1. Высокая нагрузка в .NET (продвинутый middle)

Две метрики, которые стоит назвать сразу:

- **Пропускная способность (throughput)** — сколько запросов в секунду держит сервис.
- **Задержка (latency)** — сколько идёт один запрос; важны «хвосты» `p95 / p99`, а не средние.

---

## 0. Что реально ждут от продвинутого middle

Ответ строится из четырёх мыслей:

1. **Не блокировать потоки** — правильно использовать async/await.
2. **Не создавать лишний мусор** — лишние аллокации нагружают GC.
3. **Не ходить в БД и по сети лишний раз** — кэш и нормальные запросы.
4. **Опираться на измерения** — смотреть на цифры, а не угадывать.

> 💡 Козырь: под нагрузкой узкое место чаще всего **не C#, а БД** (твой опыт Oracle/PostgreSQL, EF Core).

---

## 1. Асинхронность — главное

**Суть:** пока ждём I/O (БД, внешний сервис, файл), поток не занят и обслуживает другие запросы — отсюда высокая пропускная способность.

**Правило №1: async до конца.**

```csharp
// ✘ Плохо: поток застревает в ожидании
var user = GetUserAsync(id).Result;   // или .Wait()

// ✔ Хорошо
public async Task<IActionResult> Get(int id, CancellationToken ct)
{
    var user = await _service.GetUserAsync(id, ct);
    return Ok(user);
}
```

Что полезно знать:

- Компилятор делает из `async`-метода **машину состояний**; в точке `await` поток освобождается.
- `async void` — только для UI-событий, в серверном коде — нет (исключения не поймать).
- `Task.Run` «чтобы было асинхронно» на сервере не нужен — это лишняя работа.
- Независимые запросы запускать параллельно через `Task.WhenAll`.

```csharp
// ✘ Последовательно: 2 × 100 мс = 200 мс
var a = await GetAAsync(ct);
var b = await GetBAsync(ct);

// ✔ Параллельно: ~100 мс
var taskA = GetAAsync(ct);
var taskB = GetBAsync(ct);
await Task.WhenAll(taskA, taskB);
```

| Вопрос | Что ответить |
|---|---|
| Зачем `ValueTask`? | Экономит аллокацию, когда результат часто готов сразу (кэш). В обычном коде хватает `Task`. |
| Зачем `ConfigureAwait(false)`? | В библиотеках — чтобы не возвращать продолжение в исходный контекст. В ASP.NET Core своего контекста нет, поэтому там не критично. |
| Параллелизм? | Ограничивать! Неограниченный веер запросов исчерпает пул соединений к БД и обрушит задержки. |

---

## 2. Пул потоков простыми словами

Потоки переиспользуются из **пула** (по умолчанию — примерно по числу ядер). Блокировать их нельзя.

**Thread pool starvation (голодание пула):** запросы заняли все потоки и «зависли» на `.Result`/`.Wait()`,
ожидая ответа БД. Потоки заняты, но бесполезны; новые пул добавляет медленно (примерно по одному за 500 мс).
Итог — задержка в секунды при свободном CPU.

- **Признак:** растёт очередь пула и число потоков, а задержка огромная.
- **Лечение:** убрать блокирующие вызовы. Поднимать `MinThreads` — только как временная мера (это симптом, а не причина).

> Фраза: «Я не блокирую потоки пула: I/O — только асинхронно, чтобы поток не ждал, а работал».

---

## 3. Память и GC — база

Ключевая мысль: важна **не память, а скорость аллокаций**. Чем больше мусора в секунду — тем чаще работает GC
и тем больше CPU уходит не на работу.

- GC **поколенческий**: Gen0 собирается часто и дёшево, Gen2 — редко и дороже (и даёт паузы).
- Крупные объекты (массивы от ~85 КБ) попадают в **LOH** — лучше переиспользовать.
- На сервере обычно включён **Server GC**.

Откуда «лишний» мусор:

| Источник | Чем заменить |
|---|---|
| **Boxing** (`ArrayList`, упаковка struct в object) | Generic-коллекции: `List<int>` |
| Конкатенация строк в цикле | `StringBuilder` |
| Тяжёлые LINQ-цепочки в горячем коде | обычный цикл `for` |
| Исключения как поток управления | методы `Try*` (`TryParse`) |
| Частые крупные буферы | `ArrayPool<T>` |

```csharp
// ✘ ArrayList упаковывает каждый int в объект → мусор
var list = new ArrayList();
list.Add(42);

// ✔ Generic-коллекция
var list2 = new List<int>();
list2.Add(42);
```

> `Span<T>` и `ArrayPool<T>` — инструменты для горячих путей. На middle достаточно знать, что это и зачем; в обычном коде не усложнять.

---

## 4. Данные и кэш — где на самом деле живёт нагрузка

Сильный раздел (EF Core, Oracle/PostgreSQL, оптимизация SQL).

| Что делаю | Зачем |
|---|---|
| `AsNoTracking()` для чтения | EF не тратит CPU на отслеживание изменений |
| Проекция в DTO через `Select` | В БД уходит только нужный SQL |
| Борьба с N+1 (`Include`, проекции) | Вместо 1000 запросов — один |
| `ExecuteUpdate/ExecuteDelete` | Один UPDATE на стороне БД вместо загрузки тысяч строк |
| Индексы + `EXPLAIN ANALYZE` | Убираю Seq Scan и медленные запросы |

```csharp
// ✔ Проекция выполняется в БД, только нужные колонки, без tracking
var dtos = await _db.Users
    .AsNoTracking()
    .Where(u => u.IsActive)
    .Select(u => new UserDto(u.Id, u.Name))   // SELECT Id, Name FROM Users WHERE IsActive
    .ToListAsync(ct);
```

**Кэширование:**

- In-memory (`IMemoryCache`) — быстро, в пределах одного инстанса.
- Распределённый (`IDistributedCache`, обычно Redis) — общий для всех инстансов.
- Ошибки: нет **инвалидации** (устаревшие данные) и **cache stampede** (ключ истёк — все ломятся в БД).

**Пул соединений и backpressure:** соединения к БД — ограниченный ресурс; лишние параллельные запросы создают очередь и таймауты. Поэтому число одновременных операций ограничивают:

```csharp
private static readonly SemaphoreSlim Gate = new(8); // не более 8 сразу
await Gate.WaitAsync(ct);
try   { await ProcessOneAsync(id, ct); }
finally { Gate.Release(); }
```

---

## 5. Измеряемость — коротко, но обязательно

- **Latency** смотреть по `p95 / p99`, а не по среднему.
- **`dotnet-counters`** — «посмотреть живьём»: очередь потоков, скорость аллокаций, счётчики GC.
- **BenchmarkDotNet** — сравнить два варианта кода и увидеть аллокации.
- **k6** (или подобное) — нагрузочный тест: сколько реально держит.
- **`EXPLAIN ANALYZE`** (PostgreSQL) — когда тормозит БД.

Порядок разбора «тормозит под нагрузкой»:

1. Растёт p99 → смотрим `dotnet-counters`.
2. Растёт очередь потоков → ищем `.Result`/`.Wait()`.
3. Высокая скорость аллокаций / частые Gen0-Gen2 → ищем лишний мусор.
4. Ждём соединения из пула → проблема в БД или нет кэша.

> Фраза: «Сначала измеряю — baseline и узкое место (чаще БД или I/O). Потом оптимизирую и измеряю снова».

---

## 6. Готовый ответ (2–3 минуты)

**1. Асинхронность.** «Главное — не занимать поток, пока ждём I/O. Держу весь путь асинхронным: нет
`.Result` и `.Wait()`. Пока ждём БД или сервис, поток свободен и обслуживает другие запросы — отсюда
высокая пропускная способность».

**2. Потоки.** «Потоки берутся из пула, и я их не блокирую: блокировка ведёт к голоданию пула — потоки
заняты, но простаивают, задержка резко растёт. Правило: I/O только асинхронно».

**3. Память.** «Под нагрузкой важна не память, а скорость аллокаций: больше мусора — чаще GC — больше CPU
не на работу. Поэтому избегаю boxing, использую generic-коллекции и `StringBuilder`, не использую исключения
как поток управления. `Span` и пулы — точечно, где нужно».

**4. Данные и измерения.** «Узкое место чаще всего в БД. Поэтому `AsNoTracking` и проекции в EF Core, борьба
с N+1, массовые операции через set-based SQL, проверка плана запроса, плюс кэш на чтение. И всё по
измерениям: p95/p99, счётчики через `dotnet-counters`, а не «на глаз».»

---

## 7. Шпаргалка

**Четыре мысли, покрывающие вопрос:**

| Слой | Что сказать |
|---|---|
| Асинхронность | async/await до конца, без `.Result`/`.Wait()` |
| Потоки | не блокировать пул → нет thread pool starvation |
| Память | меньше аллокаций → меньше работы GC |
| Данные | кэш, проекции, `AsNoTracking`, борьба с N+1, план запроса |

**Ассоциации «проблема → приём»:**

- Поток простаивает → **async/await**.
- Пул «голодает» → убрать **блокирующие вызовы**.
- Много мусора → generic-коллекции, `StringBuilder`, меньше LINQ в горячем коде.
- Много запросов в БД → **кэш**, проекции, `AsNoTracking`, борьба с N+1.
- Перегрузка → **ограничение параллелизма** и rate limiting.
- «Где тормозит?» → **p95/p99**, `dotnet-counters`, `EXPLAIN ANALYZE`.

**Чего НЕ надо на этом уровне:** устройство GC/LOH-компакция, `IValueTaskSource`, Pipelines, native AOT,
fine-tuning GC. Достаточно знать, что это и зачем.

---

# Вопрос 2. Миграция DI и жизненных циклов: .NET Framework → .NET Core

## 0. Столкновение двух миров

В .NET Framework DI был **сторонним и опциональным** (Unity, Autofac, Ninject через `IDependencyResolver`),
а в .NET Core DI **встроенный и обязательный** — через Generic Host проходит всё приложение.

| Аспект | .NET Framework (MVC/WPF) | .NET Core |
|---|---|---|
| DI | не встроен, сторонний | встроенный `Microsoft.Extensions.DependencyInjection`, Generic Host |
| Получение зависимости | `new`, service locator | constructor injection |
| HttpContext | `HttpContext.Current` | `IHttpContextAccessor` |
| Конфигурация | Web.config, `ConfigurationManager` | `IConfiguration` + `IOptions*` |
| «На запрос» | `InstancePerRequest` | `Scoped` |
| Точка старта | Global.asax / OWIN | `Program` + Host |
| Валидация графа | нет | `ValidateScopes` / `ValidateOnBuild` |

## 1. Главные боли миграции

- Разбросанный `new` и статика → заводим в DI.
- **Service Locator** (`DependencyResolver.Current`) → constructor injection.
- `HttpContext.Current` больше нет → `IHttpContextAccessor`.
- Возможности сторонних контейнеров (property injection, child-контейнеры, интерцепторы) во встроенном DI отсутствуют.
- Конфигурация: `Web.config`/`ConfigurationManager` → `IConfiguration` + `IOptions*`.
- **WPF**: не было хоста → поднимали Generic Host, view-модели в DI.
- Осознанно разложить lifetimes.

## 2. Жизненные циклы

| Lifetime | Сколько живёт | Пример |
|---|---|---|
| **Transient** | каждый раз новый | мапперы, лёгкие сервисы |
| **Scoped** | один на запрос/scope | `DbContext`, репозитории |
| **Singleton** | один на приложение | конфиг, кэши, инфраструктура |

> ★ Золотое правило: **Singleton не зависит от Scoped**.

`DbContext` по умолчанию — Scoped. В фоне/консоли scope создаём сами.

## 3. Captive Dependency

Долгоживущий держит короткоживущего. Самый частый случай — **Singleton → `DbContext` (Scoped)**.

```csharp
// ✘ Singleton зависит от Scoped → captive
builder.Services.AddSingleton<ReportService>();
builder.Services.AddDbContext<AppDbContext>(o => o.UseNpgsql(cs));

public sealed class ReportService(AppDbContext db) { /* ... */ }
```

Чем плохо: рост памяти, ошибки конкурентности (`A second operation started on this context...`), «старые» данные.

Правильно — scope на операцию:

```csharp
public sealed class ReportService(IServiceScopeFactory scopeFactory)
{
    public async Task<Report> BuildAsync(CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        return await db.Reports.AsNoTracking().FirstAsync(ct);
    }
}
```

## 4. Как отлавливать

1. **Валидация графа на старте:**
```csharp
builder.Host.UseDefaultServiceProvider(o => { o.ValidateScopes = true; o.ValidateOnBuild = true; });
```
`ValidateScopes` ловит Singleton→Scoped; `ValidateOnBuild` проверяет весь граф при старте.

2. **Тест-«сторож»**, который резолвит все сервисы в scope.

3. **Код-ревью и правила**: Singleton — только stateless; красные флаги — `IServiceProvider`/`DbContext` в Singleton.

Симптомы в рантайме: рост памяти, ошибки конкурентности, устаревшие данные.

## 5. Приёмы

- `BackgroundService` (Singleton): **scope на каждую итерацию** через `IServiceScopeFactory`.
- `IOptions` (singleton) / `IOptionsSnapshot` (scoped) / `IOptionsMonitor` (singleton).
- `IHttpClientFactory` вместо `new HttpClient()` в Singleton.
- WPF: Generic Host + DI; VM/окна — не Singleton (иначе утечки подписок).

## 6. Готовый ответ (2–3 минуты)

1. В .NET Core DI встроенный и обязательный; незарегистрированная зависимость = падение на старте.
2. Боли: ручные `new`/статика/service locator → DI; `HttpContext.Current` → `IHttpContextAccessor`; Web.config → IOptions; WPF → Generic Host.
3. Lifetimes: Transient/Scoped/Singleton; правило — Singleton не зависит от Scoped.
4. Captive Dependency (#1 — Singleton→DbContext): ловим `ValidateScopes`/`ValidateOnBuild` + тестом; чиним через `IServiceScopeFactory`.

## 7. Шпаргалка

- Transient — новый каждый раз; Scoped — один на запрос (DbContext); Singleton — один на приложение.
- Captive = Singleton → Scoped.
- Поймать: `ValidateScopes` + `ValidateOnBuild`; тест, резолвящий всё.
- Починить: `IServiceScopeFactory.CreateScope()`; scope на операцию/итерацию.
- Ассоциации: `HttpContext.Current` → `IHttpContextAccessor`; `InstancePerRequest` → `Scoped`; `Web.config` → `IOptions*`.


