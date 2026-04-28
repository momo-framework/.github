# Momo Framework — Полная Техническая Спецификация

> **Версия документа:** 1.0.0-draft
> **Статус:** Living Document — обновляется по мере развития фреймворка
> **Принцип:** Event-Driven Modular Monolith → Microservices-Ready
> **Стек:** PHP 8.5+ → C-расширения → паритет с Go при TTM ×3–4 быстрее

---

## Содержание

1. [Кому читать этот документ](#1-кому-читать-этот-документ)
2. [Философия и принципы](#2-философия-и-принципы)
3. [Стратегия производительности: PHP → C-расширения → Go-паритет](#3-стратегия-производительности-php--c-расширения--go-паритет)
4. [Структура проекта](#4-структура-проекта)
5. [Пакеты ядра — подробно](#5-пакеты-ядра--подробно)
   - [momo-framework/kernel](#51-momo-frameworkkernel)
   - [momo-framework/bus](#52-momo-frameworkbus)
   - [momo-framework/events](#53-momo-frameworkevents)
   - [momo-framework/http](#54-momo-frameworkhttp)
   - [momo-framework/routing](#55-momo-frameworkrouting)
   - [momo-framework/modules](#56-momo-frameworkmodules)
   - [momo-framework/server](#57-momo-frameworkserver)
   - [momo-framework/swoole](#58-momo-frameworkswoole)
   - [momo-framework/discovery](#59-momo-frameworkdiscovery)
6. [ORM — выбор и обоснование](#6-orm--выбор-и-обоснование)
7. [Архитектура модуля (DDD)](#7-архитектура-модуля-ddd)
8. [Database — Migrations, Entities, Repositories](#8-database--migrations-entities-repositories)
9. [Bus — Command / Query / Event](#9-bus--command--query--event)
10. [Kernel и жизненный цикл запроса](#10-kernel-и-жизненный-цикл-запроса)
11. [HTTP-слой](#11-http-слой)
12. [Routing](#12-routing)
13. [Events (Event-Driven Architecture)](#13-events-event-driven-architecture)
14. [Discovery Plugin](#14-discovery-plugin)
15. [Swoole и конкурентность](#15-swoole-и-конкурентность)
16. [Тестирование](#16-тестирование)
17. [CLI — momo console](#17-cli--momo-console)
18. [Версионирование и релизы](#18-версионирование-и-релизы)
19. [Roadmap по версиям](#19-roadmap-по-версиям)
20. [Золотые правила Momo](#20-золотые-правила-momo)
21. [Глоссарий](#21-глоссарий)

---

## 1. Кому читать этот документ

| Роль                  | Что читать            | Зачем                                           |
|-----------------------|-----------------------|-------------------------------------------------|
| CEO / Product         | §2, §3, §19 (Roadmap) | Понять горизонт, TTM, конкурентное преимущество |
| Tech Lead / Architect | Весь документ         | Зафиксировать архитектурные решения на старте   |
| Backend-разработчик   | §5–§17                | Писать код по стандартам фреймворка             |
| DevOps / Infra        | §15, §17, §19         | Деплой, мониторинг, CI/CD                       |
| QA / Тестировщик      | §16, §8, §9           | Стратегия тестирования                          |

**Главная идея в двух предложениях:**
Momo — это PHP-фреймворк для модульных монолитов, где каждый бизнес-модуль живёт изолированно и общается с остальными только через шину сообщений (Bus). Сначала всё пишется на PHP — быстро, дёшево, понятно любому PHP-разработчику — затем критические пакеты переписываются на C-расширения, и производительность выходит на паритет с Go при TTM в **3–4 раза быстрее**.

---

## 2. Философия и принципы

### 2.1 Три кита Momo

| Кит | Принцип                                | Что это значит на практике                                                                               |
|-----|----------------------------------------|----------------------------------------------------------------------------------------------------------|
| ①   | **Модуль как единица деплоя**          | Любой модуль можно вынуть из монолита и запустить как отдельный микросервис без переписывания кода       |
| ②   | **Общение через Bus, не через импорт** | Модули не импортируют классы друг друга. Они посылают Commands, Queries и Events — как письма на почту   |
| ③   | **Производительность по умолчанию**    | Swoole держит процесс живым между запросами. Нет накладных расходов на запуск PHP при каждом HTTP-вызове |

### 2.2 Дизайн-решения

| Решение | Обоснование | Отвергнутая альтернатива | Почему нет |
|---------|-------------|--------------------------|-----------|
| Swoole вместо FPM | Постоянные процессы, корутины, WebSocket из коробки | ReactPHP | Меньше экосистема |
| Bus вместо прямых вызовов | Модули не зависят на уровне кода | Контракты/интерфейсы | Создают compile-time зависимость |
| Flat-root структура | Прозрачность, нет вложенности, легко извлекать модули | `app/modules/` | Запутывает неймспейсы |
| Composer Discovery Plugin | Авторегистрация без ручной правки конфигов | Ручная регистрация | Ошибки, забывчивость |
| PHP 8.5+ | Readonly, Fibers, Property hooks, новые типы | Старые версии | Компромисс по архитектуре |
| Cycle ORM v3 (Data Mapper) | Строгая типизация, нет Active Record, явные Entity | Eloquent, Doctrine | Eloquent = Active Record = анти-паттерн при DDD; Doctrine = Symfony-first, хуже Swoole |
| DDD внутри модулей | Чёткое разделение Domain/Application/Infrastructure | Только MVC | Смешивает бизнес и техническую логику |

### 2.3 Что Momo НЕ делает

- ❌ Не диктует шаблонизатор — Blade, Twig, чистый PHP, JSON API — всё нейтрально
- ❌ Не тянет Symfony Components как зависимость первого уровня
- ❌ Не позволяет модулям импортировать классы друг из друга — **только через Bus**
- ❌ Не пытается заменить Laravel — Momo решает другую задачу

---

## 3. Стратегия производительности: PHP → C-расширения → Go-паритет

### 3.1 Логика эволюции

```
Этап 1 (сейчас)         Этап 2                    Этап 3
─────────────────        ─────────────────          ─────────────────
100% PHP 8.5+            Профилирование под          Горячие пакеты → C-ext
                         реальной нагрузкой
Быстрая разработка       Выявляем 2–3 пакета-        95% кодовой базы
TTM × 3–4 быстрее Go    бутылочных горлышка          не трогаем

PHP-разработчики         bus, routing, http,
сразу продуктивны        events — первые кандидаты
```

> **Аналогия:** Строим автомобиль. Сначала рабочий прототип из дерева (PHP) — он едет.
> Потом заменяем только **двигатель и трансмиссию** на металл (C-расширения).
> Кузов, салон, электрика остаются деревянными.
> Итог: **скорость металлического** автомобиля, **скорость сборки** деревянного.

### 3.2 Конкурентная позиция

| Параметр | Go | Momo PHP | Momo PHP + C-ext |
|----------|----|----------|-----------------|
| Скорость разработки | Средняя | **Высокая** | **Высокая** |
| Онбординг PHP-разработчика | Нужно учить Go | Сразу продуктивен | Сразу продуктивен |
| Throughput (rps) | ~50 000+ | ~15 000 (Swoole) | **~45 000+** |
| Latency (p99) | < 5ms | < 20ms | **< 7ms** |
| TTM до Production | Базовый | **× 3–4 быстрее** | **× 3–4 быстрее** |
| Переход к микросервисам | Требует рефакторинг | Встроен в архитектуру | Встроен в архитектуру |

### 3.3 Кандидаты на замену C-расширениями (по приоритету)

| Пакет | Текущий (PHP) | После (C-ext) | Ожидаемый прирост |
|-------|--------------|---------------|-------------------|
| `momo/bus` | PHP-диспетчер | C-расширение | ×5–8 на горячем пути |
| `momo/routing` | PHP-матчинг | C-расширение | ×3–5 на роутинге |
| `momo/http` | PHP PSR-7 | C-расширение | ×4–6 на парсинге |
| `momo/events` | PHP pub/sub | C-расширение | ×3–4 на диспатче |

> **Вывод:** Мы доставляем продукт на рынок в 3–4 раза быстрее чем Go-команда,
> а затем **догоняем по производительности** через C-расширения — **без переписывания бизнес-логики**.

---

## 4. Структура проекта

```
/
├── app/                          # Глобальный клей (только это, ничего бизнесового)
│   ├── Bootstrap/
│   │   └── Application.php       # Entry point, собирает Kernel + модули
│   ├── Http/
│   │   └── Middleware/           # Глобальные middleware (Auth, CORS, RateLimit)
│   └── Console/
│       └── Kernel.php            # Регистрация глобальных консольных команд
│
├── modules/                      # Бизнес-модули (каждый — самодостаточный мир)
│   └── Shop/                     # Пример: e-commerce модуль
│
├── packages/                     # Системные пакеты ядра (монорепо)
│   ├── bus/                      # momo-framework/bus
│   ├── discovery/                # momo-framework/discovery
│   ├── events/                   # momo-framework/events
│   ├── http/                     # momo-framework/http
│   ├── kernel/                   # momo-framework/kernel
│   ├── modules/                  # momo-framework/modules
│   ├── routing/                  # momo-framework/routing
│   ├── server/                   # momo-framework/server
│   └── swoole/                   # momo-framework/swoole
│
├── vendor/                       # Composer deps + marketplace-модули
├── bootstrap/
│   └── cache/
│       ├── packages.php          # @generated — ServiceProviders + configs
│       └── modules-autoload.php  # @generated — PSR-4 пути модулей
│
├── config/                       # Глобальные конфиги (override defaults)
│   ├── server.php
│   └── app.php
│
├── storage/
│   ├── logs/
│   └── cache/
│
├── momo                          # CLI entry point (php momo ...)
└── composer.json
```

### 4.1 Правило «Глобального клея» (app/)

В `app/` разрешено держать **только**:

| Что | Примеры | Почему здесь |
|-----|---------|-------------|
| Global Middleware | `AuthMiddleware`, `CorsMiddleware` | Применяются ко всем модулям |
| Bootstrap/Application | `Application.php` | Собирает Kernel + модули |
| Global Console Kernel | `Console/Kernel.php` | Глобальные команды |
| Cross-cutting config | `config/app.php` | Настройки всего приложения |

> ⛔ **Запрещено в app/:** бизнес-логика, Entity, репозитории, контроллеры конкретных фич.

### 4.2 Правило неймспейсов

```
Momo\Core\        →  app/             (глобальный клей)
Momo\Module\*\    →  modules/*/src/   (бизнес-модули)
Momo\*\           →  packages/*/src/  (ядро фреймворка)
```

### 4.3 Карта зависимостей между пакетами

```
momo/server
    └── momo/swoole
            └── momo/kernel
                    ├── momo/http
                    ├── momo/routing
                    ├── momo/bus
                    │       └── momo/events
                    └── momo/modules
                            └── momo/discovery  (Composer Plugin — compile-time only)
```

> Зависимости строго **однонаправлены** — никаких циклов.
> Это обеспечивает возможность тестировать и заменять каждый пакет независимо.

---

## 5. Пакеты ядра — подробно

### 5.1 momo-framework/kernel

**Ответственность:** IoC-контейнер, регистрация ServiceProvider-ов, бутстрап приложения.

**Публичный API:**

```php
interface KernelInterface
{
    public function boot(): void;
    public function handle(Request $request): Response;
    public function terminate(Request $request, Response $response): void;
    public function getContainer(): ContainerInterface;
}
```

**Что делает `boot()`:**

1. Читает `bootstrap/cache/packages.php`
2. Инстанциирует каждый `ServiceProvider`
3. Вызывает `register()` на всех провайдерах — **только биндинги в контейнер**
4. Вызывает `boot()` на всех провайдерах — **использует уже зарегистрированные сервисы**
5. Регистрирует роуты из всех модулей

> ⚠️ `register()` и `boot()` — два **отдельных** прохода.
> Нельзя использовать сервис из контейнера в `register()`, только в `boot()`.

**IoC-контейнер:** использует Symfony DependencyInjection — самый зрелый и типизированный DI в PHP-экосистеме. Поддерживает автовайринг, теги, декораторы.

**CI требования:**
- PHPStan level 10
- 100% coverage
- php-cs-fixer
- rector

---

### 5.2 momo-framework/bus

**Ответственность:** синхронная in-process диспетчеризация Commands и Queries внутри одного request cycle. Основа CQRS в Momo.

```
HTTP / gRPC / CLI / GraphQL
           │
           ▼
    Controller / Resolver      (delivery layer — знает о протоколе)
           │
           │  new CreateOrderCommand(...)
           ▼
       CommandBus               (этот пакет)
           │
           ▼
    CreateOrderHandler          (application layer — знает о domain)
           │
           ▼
    Domain / Infrastructure     (чистая бизнес-логика)
```

**Два автобуса:**

| Bus | Метод диспатча | Возвращает | Назначение |
|-----|---------------|-----------|-----------|
| `CommandBus` | `dispatch(CommandInterface)` | `void` | Write: create, update, delete |
| `QueryBus` | `ask(QueryInterface)` | `mixed` | Read: fetch, list, search |

**Контракт — один Handler на одно сообщение:**

```php
// Регистрация второго Handler-а для того же Command → silent overwrite.
// Это намеренное решение — Fan-out запрещён на уровне Bus.
```

**Оба Bus — всегда синхронные.** Для async используется `momo-framework/queue` (будущий пакет) или Swoole Task Workers напрямую через `dispatchAsync()`.

**Инициализация:**

```php
// Composer Discovery авторегистрирует пакет — ручная регистрация не нужна.
composer require momo-framework/bus
```

**Регистрация в ServiceProvider:**

```php
final class ShopServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->singleton(CreateOrderHandler::class);
        $this->singleton(GetOrderHandler::class);
    }

    public function boot(): void
    {
        $this->app->make(CommandBusInterface::class)
            ->register(CreateOrderCommand::class, $this->app->make(CreateOrderHandler::class));

        $this->app->make(QueryBusInterface::class)
            ->register(GetOrderQuery::class, $this->app->make(GetOrderHandler::class));
    }
}
```

**Command — readonly value object:**

```php
final readonly class CreateOrderCommand implements CommandInterface
{
    public function __construct(
        public string $customerId,
        public array  $items,
    ) {}
}
```

**CommandHandler:**

```php
final class CreateOrderHandler implements CommandHandlerInterface
{
    public function __construct(
        private readonly OrderRepository $orders,
        private readonly EventBusInterface $eventBus,
    ) {}

    public function handle(CommandInterface $command): void
    {
        assert($command instanceof CreateOrderCommand);

        $order = Order::create($command->customerId, $command->items);
        $this->orders->save($order);
        // Domain Events публикуются автоматически через EventBus после save()
    }
}
```

**Query:**

```php
final readonly class GetOrderQuery implements QueryInterface
{
    public function __construct(public string $orderId) {}
}

// Диспатч:
$order = $queryBus->ask(new GetOrderQuery($orderId));
```

**CI требования:** PHPStan level 10, 100% coverage.

---

### 5.3 momo-framework/events

**Ответственность:** pub/sub между модулями, базовые классы `DomainEvent` и `AggregateRoot`, асинхронные listeners через Swoole.

**Два типа событий:**

| Тип | Источник | Когда публикуется |
|-----|---------|------------------|
| `DomainEvent` | Внутри `AggregateRoot` через `recordEvent()` | Автоматически после `Repository::save()` |
| `IntegrationEvent` | Application layer напрямую | Вручную через `EventBus::publish()` |

**Базовый класс DomainEvent:**

```php
namespace Momo\Events;

abstract class DomainEvent
{
    public readonly DateTimeImmutable $occurredAt;

    public function __construct()
    {
        $this->occurredAt = new DateTimeImmutable();
    }
}
```

**AggregateRoot:**

```php
namespace Momo\Events;

abstract class AggregateRoot
{
    private array $domainEvents = [];

    protected function recordEvent(DomainEvent $event): void
    {
        $this->domainEvents[] = $event;
    }

    public function pullDomainEvents(): array
    {
        $events = $this->domainEvents;
        $this->domainEvents = [];
        return $events;
    }
}
```

**Публичный API EventBus:**

```php
interface EventBusInterface
{
    public function publish(DomainEvent ...$events): void;

    public function subscribe(
        string $eventClass,
        EventListenerInterface $listener,
        bool $async = false,
    ): void;
}
```

**Подписка с async-флагом:**

```php
// В ServiceProvider::boot()
$eventBus->subscribe(
    ProductCreated::class,
    $this->app->make(SendProductNotificationListener::class),
    async: true  // Swoole Task Worker — не блокирует HTTP-ответ
);
```

**Правило именования событий — только прошедшее время:**

```php
// ✅ Правильно — факт свершился
final class ProductCreated extends DomainEvent {}
final class OrderPlaced extends DomainEvent {}
final class PaymentFailed extends DomainEvent {}

// ❌ Неправильно — намерение, а не факт
final class CreateProductEvent {}
final class OnProductCreated {}
```

---

### 5.4 momo-framework/http

**Ответственность:** PSR-7-совместимые Request/Response объекты, Middleware pipeline.

**Публичный API:**

```php
interface HttpKernelInterface
{
    public function handle(Request $request): Response;
    public function addMiddleware(MiddlewareInterface $middleware): void;
    public function addMiddlewareBefore(string $before, MiddlewareInterface $middleware): void;
}
```

**Pipeline:**

```
Request → [GlobalMiddleware] → [RouteMiddleware] → Controller → Response
```

**Request:**

```php
// Создание из Swoole (делает SwooleAdapter автоматически)
$request = Request::fromSwoole($swooleRequest);

// Методы
$request->method();        // GET, POST, etc.
$request->uri();           // /api/v1/products
$request->header('Authorization');
$request->body();          // raw body
$request->json();          // decoded JSON body
$request->query('page');   // query string param
$request->input('name');   // body param
$request->file('avatar');  // uploaded file
```

**Response:**

```php
Response::json(['id' => $id], 201);
Response::html($htmlContent, 200);
Response::redirect('/login', 302);
Response::noContent();         // 204
Response::notFound();          // 404
Response::unprocessable($errors); // 422
```

**FormRequest — валидация входных данных:**

```php
final class CreateProductRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'sku'            => ['required', 'string', 'max:64'],
            'name'           => ['required', 'string', 'max:255'],
            'price_amount'   => ['required', 'integer', 'min:1'],
            'price_currency' => ['required', 'string', 'size:3'],
            'stock'          => ['required', 'integer', 'min:0'],
        ];
    }
}
// При невалидных данных — автоматически 422 Unprocessable Entity
```

---

### 5.5 momo-framework/routing

**Ответственность:** регистрация маршрутов, их матчинг, передача в контроллер.

**Принципы:**
- Роуты декларируются в `routes/` каждого модуля
- `ServiceProvider::boot()` регистрирует роуты через `Router::loadRoutesFrom()`
- Поддержка групп, prefix, middleware-стека на группу
- Route caching для продакшена

**Пример файла routes/api.php:**

```php
use Momo\Routing\Router;

Router::group(['prefix' => 'api/v1', 'middleware' => ['auth']], function (Router $router) {

    $router->get('/products', [ProductController::class, 'index']);
    $router->post('/products', [ProductController::class, 'store']);
    $router->get('/products/{id}', [ProductController::class, 'show']);
    $router->put('/products/{id}', [ProductController::class, 'update']);
    $router->delete('/products/{id}', [ProductController::class, 'destroy']);

});
```

**Route параметры:**

```php
// {id} — обязательный
// {slug?} — опциональный
// {id:\d+} — с regex-ограничением

$router->get('/products/{id:\d+}', [ProductController::class, 'show']);
```

**Именованные роуты:**

```php
$router->get('/products/{id}', [ProductController::class, 'show'])
    ->name('products.show');

// Генерация URL:
$url = route('products.show', ['id' => 42]); // /api/v1/products/42
```

---

### 5.6 momo-framework/modules

**Ответственность:** базовый класс `ServiceProvider`, управление lifecycle модулей, базовые трейты и контракты.

**Базовый ServiceProvider:**

```php
namespace Momo\Modules;

abstract class ServiceProvider
{
    public function __construct(
        protected readonly Application $app,
    ) {}

    // ПЕРВЫЙ ПРОХОД: только биндинги в IoC-контейнер
    // Нельзя вызывать $this->app->make() здесь
    abstract public function register(): void;

    // ВТОРОЙ ПРОХОД: роуты, Bus, Events, миграции, команды
    // Все сервисы уже зарегистрированы
    abstract public function boot(): void;

    // Хелперы
    protected function singleton(string $abstract, ?string $concrete = null): void;
    protected function bind(string $abstract, string $concrete): void;
    protected function loadRoutesFrom(string $path): void;
    protected function loadMigrationsFrom(string $path): void;
    protected function commands(array $commands): void;
    protected function publishes(array $paths, string $group = 'default'): void;
}
```

**Module Lifecycle Hooks (v0.3.x):**

```php
interface ModuleLifecycleInterface
{
    public function onInstall(): void;    // первая установка
    public function onEnable(): void;     // включение
    public function onDisable(): void;    // отключение
    public function onUpgrade(string $fromVersion, string $toVersion): void;
    public function onUninstall(): void;  // удаление
}
```

---

### 5.7 momo-framework/server

**Ответственность:** транспортная абстракция — единый интерфейс для HTTP/1.1, HTTP/2, WebSocket, независимо от конкретного драйвера.

**Публичный API:**

```php
interface ServerInterface
{
    public function start(): void;
    public function stop(): void;
    public function reload(bool $reloadWorkers = true): void;
    public function getOptions(): ServerOptions;
}
```

**ServerOptions:**

```php
final class ServerOptions
{
    public function __construct(
        public readonly string  $host             = '0.0.0.0',
        public readonly int     $port             = 8080,
        public readonly string  $mode             = 'production', // production | development
        public readonly int     $workerProcesses  = 4,
        public readonly int     $maxRequest       = 10_000,
        public readonly bool    $enableStaticHandler = false,
        public readonly ?string $documentRoot     = null,
        public readonly bool    $enableHttp2      = false,
        public readonly ?string $sslCertFile      = null,
        public readonly ?string $sslKeyFile       = null,
    ) {}

    public static function fromArray(array $config): self;
    public static function fromEnv(): self;
}
```

| Опция | Default | Описание |
|-------|---------|---------|
| `host` | `0.0.0.0` | Bind address |
| `port` | `8080` | Порт сервера |
| `mode` | `production` | `production` или `development` |
| `workerProcesses` | `4` | Число worker-процессов |
| `maxRequest` | `10 000` | Макс. запросов до рестарта worker-а |
| `enableStaticHandler` | `false` | Раздача статических файлов |
| `documentRoot` | `null` | Папка для статики |
| `enableHttp2` | `false` | HTTP/2 (требует SSL) |

---

### 5.8 momo-framework/swoole

**Ответственность:** Swoole-специфичный HTTP Server, Worker Pool, Coroutine контекст, адаптеры Request/Response.

**Архитектура пакета:**

```
Swoole\Http\Request
        │
        ▼
SwooleRequestAdapter          конвертирует в Momo\Http\Request
        │
        ▼
KernelInterface::handle()     передаёт в ядро приложения
        │
        ▼
SwooleResponseAdapter         конвертирует Momo\Http\Response → Swoole\Http\Response
        │
        ▼
Swoole\Http\Response::end()   отправляет клиенту
```

**Запуск через CLI:**

```bash
# Продакшн
php momo server:start

# Dev-режим с авторелоадом при изменении файлов
php momo server:start --dev

# Кастомные параметры
php momo server:start --host=127.0.0.1 --port=3000 --workers=8
```

**Программный запуск:**

```php
use Momo\Swoole\SwooleServer;
use Momo\Swoole\ServerFactory;
use Momo\Server\ServerOptions;

// Простой запуск
$server = SwooleServer::withKernel($kernel, $options);
$server->start();

// Через фабрику
$server = ServerFactory::create($kernel, $options);
$server->start();

// Dev-режим с file watcher
ServerFactory::runWithWatch($kernel, $options, watchDirectories: [
    __DIR__ . '/app',
    __DIR__ . '/modules',
]);
```

**CLI Options:**

| Опция | Описание |
|-------|---------|
| `--host` | Bind address (default: 0.0.0.0) |
| `--port` | Порт (default: 8080) |
| `--workers` | Число workers (default: 4) |
| `--dev` | Dev-режим с авторелоадом |

**Dev-режим:**
- Авторелоад при изменении PHP-файлов
- Полные stack traces на ошибки
- 1 worker по умолчанию (легче дебажить)

**CoroutineContext — состояние запроса:**

```php
// ✅ Правильно — состояние изолировано в корутине
$context = CoroutineContext::current();
$context->set('user', $authenticatedUser);
$user = $context->get('user');

// ❌ НИКОГДА не делать — утечка между параллельными запросами
static $currentUser = null;
$currentUser = $authenticatedUser; // следующий запрос увидит чужого пользователя!
```

> ⚠️ **Критично:** в Swoole один Worker обрабатывает **множество запросов конкурентно** через корутины.
> `static` переменные — это утечка данных между запросами. Всё состояние запроса — **только в CoroutineContext**.

---

### 5.9 momo-framework/discovery

**Ответственность:** Composer Plugin, автоматическая регистрация локальных модулей после `composer dump-autoload`.

**Принцип работы:**

```
composer dump-autoload
        │
        ▼
POST_AUTOLOAD_DUMP event
        │
        ├─ ModuleScanner::scan()
        │    читает modules/*/composer.json
        │    собирает autoload.psr-4 записи
        │    резолвит относительные пути в абсолютные
        │
        └─ AutoloadPatcher::patch()
             перезаписывает vendor/composer/autoload_psr4.php
             инжектирует addPsr4() вызовы в vendor/autoload_real.php
```

**Что генерируется:**

| Файл | Содержимое |
|------|-----------|
| `vendor/composer/autoload_psr4.php` | Перезаписывается со смерженной namespace map |
| `vendor/autoload_real.php` | Инжектируются `addPsr4()` вызовы перед `return $loader` |

**Формат module's composer.json:**

```json
{
  "name": "momo-module/shop",
  "type": "momo-module",
  "autoload": {
    "psr-4": {
      "Momo\\Module\\Shop\\": "src/"
    }
  },
  "extra": {
    "momo": {
      "providers": [
        "Momo\\Module\\Shop\\Application\\Providers\\ShopServiceProvider"
      ]
    }
  }
}
```

**Локальные vs Vendor-модули:**

| | Локальный модуль | Vendor-модуль |
|--|-----------------|--------------|
| Расположение | `modules/` | `vendor/` |
| Автозагрузка | инжектируется этим плагином | стандартный Composer |
| Зависимости | через корневой `composer.json` | собственный `composer.json` |
| Редактируемый | ✅ да, `make:*` работают напрямую | ❌ нет, сначала `module:publish` |

> **Идемпотентность:** патчи защищены guard-комментарием — повторные `dump-autoload` не создают дублей.

> ⛔ `bootstrap/cache/` — файлы `@generated`. Никогда не редактировать вручную.
> После любых изменений модулей: `composer dump-autoload`.

---

## 6. ORM — выбор и обоснование

### 6.1 Почему не Active Record

Active Record (Eloquent и подобные) — анти-паттерн при Domain-Driven Design:

```php
// Active Record: Entity == Row. Domain-логика живёт рядом с SQL.
class Product extends Model  // ← extends Model — это смерть Domain
{
    public function calculateDiscount(): Money  // бизнес-логика
    {
        $this->save(); // ← побочный эффект ВНУТРИ бизнес-метода — кошмар
    }
}

// В Momo: Entity ничего не знает о БД
final class Product extends AggregateRoot  // ← только бизнес-контракт
{
    public function applyDiscount(Discount $discount): void
    {
        // чистая логика, нет save(), нет SQL, нет зависимостей
        $this->recordEvent(new DiscountApplied($this->id, $discount));
    }
}
```

### 6.2 Почему не Doctrine

Doctrine — правильный Data Mapper, но:
- Разработан для Symfony-экосистемы, несёт лишние зависимости
- DBAL/DBAL2 плохо совместимы с Swoole coroutines
- Connection pooling через Swoole требует глубоких патчей
- Proxies и lazy loading конфликтуют с Coroutine Context

### 6.3 Выбор: Cycle ORM v3

**[Cycle ORM](https://cycle-orm.app/)** — Data Mapper ORM, написанный с нуля для современного PHP:

| Критерий | Cycle ORM v3 |
|---------|-------------|
| Паттерн | Data Mapper (не Active Record) |
| Типизация | PHPStan level 10 из коробки |
| PHP 8.5+ | ✅ полная поддержка readonly, fibers |
| Swoole совместимость | ✅ Connection pooling через `spiral/roadrunner-bridge` |
| Миграции | ✅ встроенные через `cycle/migrations` |
| Schema generation | ✅ из PHP-аттрибутов или отдельных схем |
| Coroutine-safe | ✅ отдельный UoW на корутину |
| Связи | all (has one, has many, many-to-many, morphic) |
| Raw queries | ✅ через `cycle/database` (DBAL) |
| Custom типы | ✅ typecast через атрибуты |

### 6.4 Как Cycle ORM вписывается в архитектуру Momo

```
Domain/
    Entities/
        Product.php        ← чистый PHP, НЕТ атрибутов ORM внутри
    Repositories/
        ProductRepositoryInterface.php  ← интерфейс в Domain

Infrastructure/
    Persistence/
        Schema/
            ProductSchema.php   ← Cycle schema (атрибуты или Array Schema)
        Records/
            ProductRecord.php   ← опциональный Record слой (если нужна доп. изоляция)
        Hydrators/
            ProductHydrator.php ← Record → Domain Entity
        ProductRepository.php   ← реализует ProductRepositoryInterface через Cycle
```

**Вариант 1 — Schema через аттрибуты (на отдельном класс-маппере):**

```php
namespace Momo\Module\Shop\Infrastructure\Persistence\Schema;

use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Relation\HasMany;

// Это НЕ Domain Entity — это схема для маппинга
#[Entity(table: 'products', repository: ProductRepository::class)]
final class ProductMapper
{
    #[Column(type: 'uuid', primary: true)]
    public string $id;

    #[Column(type: 'string', length: 64)]
    public string $sku;

    #[Column(type: 'string', length: 255)]
    public string $name;

    #[Column(type: 'integer')]
    public int $priceAmount;

    #[Column(type: 'char', length: 3)]
    public string $priceCurrency;

    #[Column(type: 'integer')]
    public int $stock;

    #[Column(type: 'datetime')]
    public DateTimeImmutable $createdAt;
}
```

**Вариант 2 — Array Schema (строгий, без аннотаций):**

```php
namespace Momo\Module\Shop\Infrastructure\Persistence\Schema;

use Cycle\ORM\Schema;
use Cycle\ORM\Relation;

return [
    'product' => [
        Schema::ENTITY     => ProductMapper::class,
        Schema::MAPPER     => \Cycle\ORM\Mapper\StdMapper::class,
        Schema::REPOSITORY => ProductRepository::class,
        Schema::TABLE      => 'products',
        Schema::PRIMARY_KEY => 'id',
        Schema::COLUMNS    => [
            'id'             => 'id',
            'sku'            => 'sku',
            'name'           => 'name',
            'price_amount'   => 'priceAmount',
            'price_currency' => 'priceCurrency',
            'stock'          => 'stock',
            'created_at'     => 'createdAt',
        ],
        Schema::TYPECAST    => [
            'id'         => 'uuid',
            'created_at' => 'datetime',
        ],
    ],
];
```

**Repository реализация через Cycle:**

```php
namespace Momo\Module\Shop\Infrastructure\Persistence;

use Cycle\ORM\ORMInterface;
use Cycle\ORM\Transaction;
use Momo\Module\Shop\Domain\Entities\Product;
use Momo\Module\Shop\Domain\Repositories\ProductRepositoryInterface;
use Momo\Module\Shop\Domain\ValueObjects\ProductId;

final class ProductRepository implements ProductRepositoryInterface
{
    public function __construct(
        private readonly ORMInterface $orm,
    ) {}

    public function findById(ProductId $id): ?Product
    {
        // Достаём mapper-объект, конвертируем в Domain Entity через Hydrator
        $mapper = $this->orm->getRepository(ProductMapper::class)->findByPK((string) $id);
        return $mapper !== null ? ProductHydrator::toDomain($mapper) : null;
    }

    public function save(Product $product): void
    {
        $mapper = ProductHydrator::fromDomain($product);

        $transaction = new Transaction($this->orm);
        $transaction->persist($mapper);
        $transaction->run();

        // Публикуем накопленные Domain Events
        foreach ($product->pullDomainEvents() as $event) {
            $this->eventBus->publish($event);
        }
    }
}
```

**Unit of Work — Cycle автоматически:**

```php
// Cycle ORM поддерживает UoW из коробки
// Один Transaction = одна единица работы
// Все изменения применяются атомарно

$transaction = new Transaction($orm);
$transaction->persist($productMapper);
$transaction->persist($inventoryMapper);
$transaction->run(); // один COMMIT
```

### 6.5 Миграции через cycle/migrations

```bash
# Генерация миграции на основе изменений схемы
php momo migrate:diff --module Shop

# Применение миграций
php momo migrate:run

# Откат
php momo migrate:rollback

# Статус
php momo migrate:status
```

**Структура файла миграции:**

```php
namespace Momo\Module\Shop\Database\Migrations;

use Cycle\Migrations\Migration;

final class CreateProductsTable extends Migration
{
    public function up(): void
    {
        $this->table('products')
            ->addColumn('id',             'uuid',     ['nullable' => false])
            ->addColumn('sku',            'string',   ['nullable' => false, 'size' => 64])
            ->addColumn('name',           'string',   ['nullable' => false, 'size' => 255])
            ->addColumn('price_amount',   'integer',  ['nullable' => false])
            ->addColumn('price_currency', 'char',     ['nullable' => false, 'size' => 3])
            ->addColumn('stock',          'integer',  ['nullable' => false, 'default' => 0])
            ->addColumn('created_at',     'datetime', ['nullable' => false])
            ->addIndex(['sku'], ['unique' => true])
            ->setPrimaryKeys(['id'])
            ->create();
    }

    public function down(): void
    {
        $this->table('products')->drop();
    }
}
```

### 6.6 Swoole + Cycle: Connection Pooling

Cycle ORM работает с Swoole через корутинный connection pool. Каждая корутина получает изолированное соединение:

```php
// В ServiceProvider::register()
$this->singleton(DatabaseInterface::class, function () {
    return DatabaseManager::fromConfig([
        'default' => [
            'driver'   => 'pgsql',  // или mysql, sqlite
            'host'     => env('DB_HOST'),
            'port'     => env('DB_PORT', 5432),
            'database' => env('DB_DATABASE'),
            'username' => env('DB_USERNAME'),
            'password' => env('DB_PASSWORD'),
            // Swoole coroutine-safe pool
            'pool' => [
                'minActive'         => 4,
                'maxActive'         => env('DB_POOL_MAX', 64),
                'waitTimeout'       => 3.0,
                'idleTimeout'       => 60.0,
            ],
        ],
    ]);
});
```

---

## 7. Архитектура модуля (DDD)

### 7.1 Четыре слоя

Зависимости идут **только снаружи внутрь**. Domain ничего не знает о фреймворке:

| Слой | Папка | Знает о | Содержит |
|------|-------|---------|---------|
| Domain | `src/Domain/` | Только себя | Entity, Value Object, Domain Event, Repository Interface, Exceptions |
| Application | `src/Application/` | Domain | Command, Query, Handler, Listener, DTO, ServiceProvider |
| Infrastructure | `src/Infrastructure/` | Domain + Application | Repository реализация, HTTP-клиенты, Console-команды, ORM-схемы |
| Presentation | `src/Presentation/` | Application (DTO) | Controller, FormRequest, Resource (JSON-трансформер) |

> 🏆 Главное правило: ни одного `use Momo\*` в `src/Domain/`.
> Domain — чистая бизнес-логика, переносимая в любой фреймворк.

### 7.2 Полная структура модуля

```
modules/Shop/
├── composer.json                       # type: momo-module
├── module.json                         # Манифест модуля (версия, deps, publishes)
├── config/
│   └── shop.php                        # Дефолтные конфиги модуля
│
├── database/
│   ├── migrations/                     # Миграции ТОЛЬКО этого модуля
│   │   ├── 0001_create_products_table.php
│   │   └── 0002_add_sku_index.php
│   ├── seeders/
│   │   └── ProductSeeder.php
│   └── factories/
│       └── ProductFactory.php
│
├── routes/
│   ├── api.php
│   └── web.php
│
├── src/
│   ├── Domain/                         # ← НОЛЬ зависимостей от фреймворка
│   │   ├── Entities/
│   │   │   └── Product.php             # AggregateRoot
│   │   ├── ValueObjects/
│   │   │   ├── ProductId.php
│   │   │   ├── Money.php
│   │   │   └── Sku.php
│   │   ├── Events/
│   │   │   ├── ProductCreated.php
│   │   │   ├── ProductPriceChanged.php
│   │   │   └── ProductDeleted.php
│   │   ├── Repositories/
│   │   │   └── ProductRepositoryInterface.php  # только интерфейс
│   │   └── Exceptions/
│   │       ├── ProductNotFoundException.php
│   │       └── InvalidSkuException.php
│   │
│   ├── Application/
│   │   ├── Commands/
│   │   │   ├── CreateProduct.php       # readonly, без логики
│   │   │   ├── UpdateProductPrice.php
│   │   │   └── DeleteProduct.php
│   │   ├── Handlers/
│   │   │   ├── CreateProductHandler.php
│   │   │   ├── UpdateProductPriceHandler.php
│   │   │   └── DeleteProductHandler.php
│   │   ├── Queries/
│   │   │   ├── GetProduct.php
│   │   │   └── ListProducts.php
│   │   ├── QueryHandlers/
│   │   │   ├── GetProductHandler.php
│   │   │   └── ListProductsHandler.php
│   │   ├── DTO/
│   │   │   ├── ProductDTO.php          # результат Query — не Entity
│   │   │   └── ProductListDTO.php
│   │   ├── Listeners/
│   │   │   └── ReserveStockOnOrderCreated.php
│   │   └── Providers/
│   │       └── ShopServiceProvider.php
│   │
│   ├── Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── Schema/
│   │   │   │   └── ProductMapper.php   # Cycle ORM маппинг
│   │   │   ├── Hydrators/
│   │   │   │   └── ProductHydrator.php # Mapper → Domain Entity
│   │   │   └── ProductRepository.php   # реализует интерфейс через Cycle
│   │   ├── Http/
│   │   │   └── PaymentGatewayClient.php
│   │   └── Console/
│   │       └── ImportProductsCommand.php
│   │
│   └── Presentation/
│       ├── Controllers/
│       │   └── ProductController.php
│       ├── Requests/
│       │   ├── CreateProductRequest.php
│       │   └── UpdateProductRequest.php
│       └── Resources/
│           └── ProductResource.php     # DTO → JSON
│
└── tests/
    ├── Unit/
    │   ├── Domain/                     # Entity, Value Objects — без фреймворка
    │   └── Application/                # Handlers с in-memory репозиториями
    └── Integration/
        ├── Persistence/                # Repository с реальной БД
        └── Http/                       # Feature-тесты HTTP-эндпоинтов
```

### 7.3 Domain Entity — AggregateRoot

```php
namespace Momo\Module\Shop\Domain\Entities;

use Momo\Events\AggregateRoot; // единственная зависимость — базовый класс из ядра

final class Product extends AggregateRoot
{
    private function __construct(
        public readonly ProductId $id,
        private Sku    $sku,
        private string $name,
        private Money  $price,
        private int    $stock,
    ) {}

    // Фабричный метод — единственный способ создать продукт
    public static function create(Sku $sku, string $name, Money $price, int $stock): self
    {
        if ($stock < 0) {
            throw new \InvalidArgumentException('Stock cannot be negative');
        }

        $product = new self(
            id: ProductId::generate(),
            sku: $sku, name: $name, price: $price, stock: $stock,
        );

        $product->recordEvent(new ProductCreated(
            productId: $product->id,
            sku: $product->sku,
            price: $product->price,
        ));

        return $product;
    }

    public function changePrice(Money $newPrice): void
    {
        if ($newPrice->equals($this->price)) {
            return; // нет изменений — нет события
        }

        $oldPrice  = $this->price;
        $this->price = $newPrice;

        $this->recordEvent(new ProductPriceChanged(
            productId: $this->id,
            oldPrice: $oldPrice,
            newPrice: $newPrice,
        ));
    }

    public function price(): Money  { return $this->price; }
    public function sku(): Sku      { return $this->sku; }
    public function name(): string  { return $this->name; }
    public function stock(): int    { return $this->stock; }
}
```

### 7.4 Value Objects

```php
// Value Object: readonly, сравнение по значению, не по ссылке
final readonly class Money
{
    public function __construct(
        public readonly int    $amount,   // всегда в минимальных единицах (центы)
        public readonly string $currency, // ISO 4217: USD, EUR, AMD
    ) {
        if ($this->amount < 0) {
            throw new \InvalidArgumentException("Money amount cannot be negative");
        }
        if (strlen($this->currency) !== 3) {
            throw new \InvalidArgumentException("Currency must be ISO 4217 (3 chars)");
        }
    }

    public function add(self $other): self
    {
        $this->assertSameCurrency($other);
        return new self($this->amount + $other->amount, $this->currency);
    }

    public function equals(self $other): bool
    {
        return $this->amount === $other->amount && $this->currency === $other->currency;
    }

    public function format(): string
    {
        return number_format($this->amount / 100, 2) . ' ' . $this->currency;
    }

    private function assertSameCurrency(self $other): void
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException(
                "Currency mismatch: {$this->currency} vs {$other->currency}"
            );
        }
    }
}

// Типобезопасный ID — не просто string
final readonly class ProductId
{
    public function __construct(public readonly string $value)
    {
        if (empty($this->value)) {
            throw new \InvalidArgumentException('ProductId cannot be empty');
        }
    }

    public static function generate(): self
    {
        return new self(\Ramsey\Uuid\Uuid::uuid7()->toString());
    }

    public function equals(self $other): bool { return $this->value === $other->value; }
    public function __toString(): string       { return $this->value; }
}
```

### 7.5 module.json — манифест

```json
{
  "name": "momo-module/shop",
  "version": "1.2.0",
  "momo-version": ">=0.2.0 <1.0.0",
  "provider": "Momo\\Module\\Shop\\Application\\Providers\\ShopServiceProvider",
  "depends": ["momo-module/catalog"],
  "publishes": {
    "events":   ["ProductCreated", "ProductDeleted", "ProductPriceChanged"],
    "commands": ["CreateProduct", "UpdateProductPrice", "DeleteProduct"],
    "queries":  ["GetProduct", "ListProducts"]
  }
}
```

> `publishes` — это **публичный контракт** модуля.
> Другие модули могут слушать только опубликованные события и диспатчить только опубликованные команды.
> Всё остальное — приватные детали реализации.

---

## 8. Database — Migrations, Entities, Repositories

### 8.1 Принцип изоляции миграций

Каждый модуль управляет своими миграциями независимо. Нет центральной `database/migrations/`:

```
modules/Shop/database/migrations/
    0001_create_products_table.php
    0002_create_categories_table.php
    0003_add_sku_index_to_products.php

modules/Order/database/migrations/
    0001_create_orders_table.php
    0002_create_order_items_table.php
```

### 8.2 Record vs Domain Entity (опционально)

При необходимости дополнительной изоляции от ORM-деталей:

| Слой | Что это | Зачем |
|------|---------|-------|
| `ProductMapper` | Cycle ORM schema class | Изолирует Domain Entity от ORM-атрибутов |
| `ProductHydrator` | Конвертер Mapper ↔ Domain Entity | Единственное место, знающее оба мира |
| `Product` (Entity) | Чистая бизнес-логика | Тестируется без БД. Независим от ORM |

### 8.3 Repository Interface в Domain

```php
namespace Momo\Module\Shop\Domain\Repositories;

// Интерфейс живёт в Domain — он часть бизнес-контракта
// Реализация живёт в Infrastructure — там знают про ORM и БД
interface ProductRepositoryInterface
{
    public function findById(ProductId $id): ?Product;
    public function findBySku(Sku $sku): ?Product;

    /** @return list<Product> */
    public function findAll(int $limit = 50, int $offset = 0): array;

    public function save(Product $product): void;   // insert или update
    public function delete(ProductId $id): void;
}
```

---

## 9. Bus — Command / Query / Event

### 9.1 Три типа сообщений

| Тип | Назначение | Handler-ов | Пример |
|-----|-----------|-----------|--------|
| **Command** | Изменить состояние системы | Ровно один | `CreateProduct`, `PlaceOrder`, `CancelSubscription` |
| **Query** | Прочитать данные без побочных эффектов | Ровно один | `GetProduct`, `ListOrders`, `SearchUsers` |
| **Event** | Оповестить что что-то случилось | 0 или много | `ProductCreated`, `OrderPlaced`, `PaymentFailed` |

### 9.2 Полный путь Command

```
1. ProductController::store()
   → new CreateProductCommand(sku, name, price, stock)

2. CommandBus::dispatch(command)
   → TransactionMiddleware::process()
   → LoggingMiddleware::process()
   → CreateProductHandler::handle()

3. CreateProductHandler
   → Product::create(...)              ← Domain Entity
   → recordEvent(ProductCreated)       ← накапливается внутри
   → productRepository->save(product)  ← запись в БД через Cycle ORM

4. ProductRepository::save()
   → ProductHydrator::fromDomain()     ← Entity → Mapper
   → Cycle Transaction::persist()      ← UoW
   → Transaction::run()                ← COMMIT
   → EventBus::publish(...pullDomainEvents()) ← публикуем накопленные события

5. EventBus
   → CreateInventoryRecordListener     ← синхронно
   → SendProductNotificationListener   ← async (Swoole Task Worker)

6. Handler возвращает ProductId → Controller → Response::json(['id' => '...'], 201)
```

### 9.3 Async dispatch

```php
// Синхронный — ждём результата, транзакция гарантирована
$productId = $this->commandBus->dispatch(new CreateProductCommand(...));

// Асинхронный — не ждём, Swoole Task Worker подхватит
$this->commandBus->dispatchAsync(new SendWelcomeEmailCommand($userId));
$this->commandBus->dispatchAsync(new GenerateInvoicePdfCommand($orderId));
```

### 9.4 Bus Middleware

```php
// Глобальные middleware для всех Command-ов:
$commandBus->addMiddleware(new TransactionMiddleware($db));
$commandBus->addMiddleware(new LoggingMiddleware($logger));
$commandBus->addMiddleware(new ValidationMiddleware());

// Специфичный для конкретного Command-а — через аттрибут:
#[WithMiddleware(RetryMiddleware::class)]
final readonly class ProcessPaymentCommand implements CommandInterface { ... }
```

---

## 10. Kernel и жизненный цикл запроса

### 10.1 Старт приложения (один раз)

```
php momo server:start
    │
    ▼
SwooleServer::start()
    │
    ▼
Kernel::boot()
    ├── читает bootstrap/cache/packages.php
    ├── инициализирует IoC-контейнер
    ├── ServiceProvider::register() × все провайдеры   ← только биндинги
    ├── ServiceProvider::boot()   × все провайдеры   ← роуты, bus, events, migrations
    └── Сервер готов — процесс живёт неограниченно
```

### 10.2 Жизненный цикл запроса

| Шаг | Компонент | Что происходит |
|-----|----------|---------------|
| 1 | Swoole | Принимает TCP-соединение |
| 2 | SwooleRequestAdapter | Конвертирует `Swoole\Http\Request` → `Momo\Http\Request` |
| 3 | HttpKernel | Запускает Global Middleware pipeline |
| 4 | CorsMiddleware | Добавляет CORS-заголовки |
| 5 | RateLimitMiddleware | Проверяет лимиты (429 если превышен) |
| 6 | AuthMiddleware | Валидирует JWT, кладёт User в CoroutineContext |
| 7 | Router | Матчит URL → `Controller::action` + Route Middleware |
| 8 | Route Middleware | `ThrottleMiddleware` и другие специфичные для роута |
| 9 | FormRequest | Валидация. При ошибке — автоматически 422 |
| 10 | Controller | Создаёт Command/Query, диспатчит в Bus |
| 11 | Handler | Бизнес-логика, Domain Events, сохранение |
| 12 | SwooleResponseAdapter | Конвертирует `Momo\Http\Response` → Swoole, отправляет |

---

## 11. HTTP-слой

> Подробно — раздел [5.4 momo-framework/http](#54-momo-frameworkhttp)

Ключевые точки:
- Request/Response — PSR-7 совместимые объекты
- FormRequest — валидация, автоматический 422 при ошибках
- Resource — трансформация DTO → JSON для API-ответов

---

## 12. Routing

> Подробно — раздел [5.5 momo-framework/routing](#55-momo-frameworkrouting)

Ключевые точки:
- Роуты декларируются в `routes/api.php` каждого модуля
- Именованные роуты через `->name()`
- Route caching для продакшена: `php momo route:cache`

---

## 13. Events (Event-Driven Architecture)

> Подробно — раздел [5.3 momo-framework/events](#53-momo-frameworkevents)

Ключевые точки:
- Domain Events накапливаются внутри AggregateRoot до `Repository::save()`
- После save() — автоматически публикуются через EventBus
- Listeners могут быть sync или async (Swoole Task Workers)
- Naming convention: **прошедшее время** (`ProductCreated`, не `CreateProductEvent`)

---

## 14. Discovery Plugin

> Подробно — раздел [5.9 momo-framework/discovery](#59-momo-frameworkdiscovery)

Ключевые точки:
- `POST_AUTOLOAD_DUMP` — хук Composer
- Автоматически патчит `vendor/composer/autoload_psr4.php`
- Идемпотентен — повторные запуски безопасны
- `bootstrap/cache/` — никогда не редактировать вручную

---

## 15. Swoole и конкурентность

> Подробно — раздел [5.8 momo-framework/swoole](#58-momo-frameworkswoole)

### 15.1 Worker Model

```
Swoole HTTP Server
    ├── Master Process        (управляет worker-ами)
    ├── Manager Process       (мониторит worker-ов)
    └── Worker Process × N   (обрабатывают запросы)
         ├── Coroutine 1   ← запрос A
         ├── Coroutine 2   ← запрос B
         └── Coroutine N   ← запрос N (конкурентно, не параллельно)
```

### 15.2 Graceful Restart

```bash
# Zero-downtime рестарт — старые workers дожидаются завершения текущих запросов
php momo server:restart

# Или через сигнал
kill -USR1 $(cat storage/server.pid)
```

### 15.3 Connection Pooling

При Swoole каждое соединение с БД должно быть привязано к корутине.
Cycle ORM с `SwooleDriver` управляет pool автоматически:

```
Worker Process
    ├── Coroutine 1 → Connection из pool (занята)
    ├── Coroutine 2 → Connection из pool (занята)
    └── Coroutine 3 → ждёт освобождения (waitTimeout: 3s)
```

---

## 16. Тестирование

### 16.1 Пирамида тестов

| Уровень | Папка | Скорость | Что тестируют |
|---------|-------|---------|--------------|
| Unit/Domain | `tests/Unit/Domain/` | Мгновенно | Entity, Value Objects, Domain Events. БЕЗ фреймворка |
| Unit/Application | `tests/Unit/Application/` | Очень быстро | Handlers с in-memory репозиториями |
| Integration/Persistence | `tests/Integration/Persistence/` | Медленно | Repository с реальной тестовой БД |
| Integration/Http | `tests/Integration/Http/` | Медленно | HTTP Feature-тесты от запроса до ответа |

### 16.2 Принципы

- Domain-тесты — **ноль зависимостей от фреймворка**
- Handlers тестируются с **in-memory реализациями** репозиториев (не mock через Mockery)
- Feature-тесты используют отдельную тестовую БД, сбрасываемую после каждого теста
- Factories используются везде — никаких голых SQL INSERT в тестах

### 16.3 Цели покрытия

| Слой | Покрытие | Приоритет |
|------|---------|---------|
| Domain (Entity + Value Objects) | 100% | Критично |
| Application (Handlers) | 90%+ | Высокий |
| Infrastructure (Repository) | 80%+ | Средний |
| Presentation (Controllers) | 70%+ | Через Feature-тесты |

### 16.4 CI Pipeline

```bash
composer ci
  ├── lint          php-cs-fixer --dry-run
  ├── stan          phpstan level 10
  ├── rector:check  rector --dry-run
  └── test          phpunit --coverage-text
```

---

## 17. CLI — momo console

### 17.1 Сервер

```bash
php momo server:start                            # запуск
php momo server:start --dev                      # dev-режим с авторелоадом
php momo server:start --host=127.0.0.1 --port=3000 --workers=8
php momo server:stop                             # graceful shutdown
php momo server:restart                          # zero-downtime рестарт
```

### 17.2 База данных

```bash
php momo migrate:run                             # все миграции
php momo migrate:run --module Shop               # миграции модуля Shop
php momo migrate:rollback                        # откат последней
php momo migrate:status                          # статус
php momo migrate:diff --module Shop              # diff схемы (Cycle ORM)
php momo db:seed --module Shop                   # сиды
```

### 17.3 Модули и скаффолдинг

```bash
php momo module:create MyModule                  # скаффолдинг нового модуля

php momo make:command MyCommand --module Shop    # Console Command
php momo make:handler MyHandler --module Shop    # Command Handler
php momo make:query MyQuery --module Shop        # Query + Handler
php momo make:event MyEvent --module Shop        # Domain Event
php momo make:listener MyListener --module Shop  # Event Listener
php momo make:controller MyCtrl --module Shop    # Controller
php momo make:entity MyEntity --module Shop      # Domain Entity
php momo make:value-object Money --module Shop   # Value Object
php momo make:repository MyRepo --module Shop    # Repository (interface + impl)
```

### 17.4 Конфигурация и кэш

```bash
php momo config:publish momo-module/shop         # публикация дефолтных конфигов
php momo config:cache                            # кэш для продакшена
php momo config:clear                            # очистка кэша конфигов

php momo route:cache                             # кэш роутов для продакшена
php momo route:list                              # список всех маршрутов
php momo route:clear                             # очистка кэша роутов

php momo cache:clear                             # полная очистка кэша
```

---

## 18. Версионирование и релизы

### 18.1 SemVer

```
MAJOR.MINOR.PATCH[-prerelease]

1.0.0          стабильный релиз
1.1.0          новая функциональность (обратно совместима)
1.1.1          исправление багов
2.0.0          breaking changes
0.1.0-alpha.1  текущая стадия
```

### 18.2 Совместимость модулей

```json
{
  "momo-version": ">=0.2.0 <1.0.0"
}
```

Discovery Plugin проверяет совместимость при `composer install` и предупреждает о конфликтах.

### 18.3 Changelog

Каждый пакет ведёт `CHANGELOG.md` в формате [Keep a Changelog](https://keepachangelog.com):

```markdown
## [Unreleased]

## [0.2.0] - 2025-06-01
### Added
- Command Bus middleware pipeline
- Async command dispatch via Swoole Task Workers
- Cycle ORM integration package

### Changed
- ServiceProvider::boot() теперь получает Container как аргумент

### Fixed
- Memory leak in Router cache
```

---

## 19. Roadmap по версиям

### v0.1.x — Foundation *(текущее)*

- [x] Flat-root структура проекта
- [x] Discovery Plugin + авторегистрация модулей
- [x] Базовый HTTP Kernel + Swoole
- [x] DDD-структура модулей
- [x] PSR-4 автозагрузка модулей
- [ ] Базовый CommandBus (синхронный)
- [ ] Базовый QueryBus
- [ ] Cycle ORM интеграция (базовый пакет `momo-framework/orm`)

### v0.2.x — Bus & Events

- [ ] **CommandBus** — полная реализация с middleware
- [ ] **QueryBus** — полная реализация
- [ ] **EventBus** — pub/sub между модулями
- [ ] Domain Events + AggregateRoot base class
- [ ] Async dispatch через Swoole Task Workers
- [ ] `module.json` манифест + обновление Discovery
- [ ] Cycle ORM migrations через `php momo migrate:*`

### v0.3.x — DX & Lifecycle

- [ ] Module Lifecycle Hooks (onInstall/onEnable/onDisable/onUpgrade/onUninstall)
- [ ] CLI scaffolding: `momo module:create`, `momo make:*`
- [ ] Hot Reload в dev-режиме (Swoole file watcher)
- [ ] SemVer для всех `packages/` + CHANGELOG.md
- [ ] PHPUnit integration tests для пакетов ядра
- [ ] Cycle ORM полная интеграция + connection pool для Swoole

### v0.4.x — Production Hardening

- [ ] Connection Pool для БД (Swoole-совместимый, через Cycle)
- [ ] Coroutine Context management
- [ ] Config caching для продакшена
- [ ] Route caching
- [ ] Health check endpoint
- [ ] Graceful restart (`server:restart` без потери запросов)
- [ ] Metrics endpoint (Prometheus-совместимый)

### v0.5.x — ACL & Security

- [ ] Декларативные permissions в ServiceProvider
- [ ] Gate/Policy система
- [ ] Rate Limiting middleware
- [ ] Request signing

### v1.0.0 — Stable Release

- [ ] Полная документация
- [ ] 100% PHPStan level 10 по всем пакетам
- [ ] Все пакеты с SemVer
- [ ] Backward compatibility guarantees
- [ ] `momo-framework/orm` — стабильный Cycle ORM пакет

### v1.x.x — Marketplace

- [ ] Module Registry (приватный Packagist-совместимый)
- [ ] `momo module:pack` — упаковка для маркетплейса
- [ ] Module signing и верификация
- [ ] Web UI для управления модулями

### v2.0.x — C-расширения: Первая волна 🚀

- [ ] Профилирование под реальной нагрузкой → определение горячих пакетов
- [ ] `momo/bus` → C-расширение (горячий диспатч-путь)
- [ ] `momo/routing` → C-расширение (матчинг)
- [ ] Benchmark: первые результаты против Go

### v2.1.x — C-расширения: Вторая волна

- [ ] `momo/http` → C-расширение (парсинг Request/Response)
- [ ] `momo/events` → C-расширение (pub/sub диспатч)
- [ ] Throughput цель: **~40 000+ rps**

### v3.0.x — Go Parity 🏁

- [ ] Benchmark: throughput и latency вплотную к Go
- [ ] p99 latency < 7ms
- [ ] Throughput > 45 000 rps
- [ ] Объявление: "PHP + C-ext дышит в спину Go"

---

## 20. Золотые правила Momo

Эти правила — квинтэссенция всей архитектуры. Нарушение подрывает гарантии фреймворка:

| # | Правило | Почему важно |
|---|---------|-------------|
| 1 | **Модуль НЕ импортирует классы из другого модуля** | Только через Bus (Commands, Queries, Events). Иначе — сильная связность, невозможно извлечь модуль |
| 2 | **Domain НЕ зависит от фреймворка** | Ни одного `use Momo\*` в `src/Domain/`. Domain живёт вечно, фреймворки меняются |
| 3 | **Handler — один на Command/Query** | Никаких fan-out. Одна команда — одна точка изменения |
| 4 | **Listeners — много на Event** | Один факт — много реакций. Расширяемость без изменения существующего кода |
| 5 | **Состояние запроса — только в CoroutineContext** | Никаких `static` переменных. В Swoole один Worker = много параллельных корутин |
| 6 | **`register()` — биндинги, `boot()` — всё остальное** | Нарушение ломает порядок инициализации |
| 7 | **Публичный API модуля — только в `module.json` секции `publishes`** | Остальное — приватные детали. Контракт явный и версионированный |
| 8 | **Версии модулей — SemVer, breaking changes — только в MAJOR** | Предсказуемые обновления для всей команды |
| 9 | **Entity НЕ расширяет ORM-класс** | Data Mapper, не Active Record. Cycle ORM работает через отдельный Mapper/Schema |
| 10 | **Миграции — внутри модуля, не в глобальной папке** | Модуль самодостаточен. При извлечении в микросервис берёт миграции с собой |

---

## 21. Глоссарий

| Термин | Техническое значение | Простыми словами |
|--------|---------------------|-----------------|
| **Swoole** | PHP C-расширение для асинхронного программирования | Позволяет PHP держать один процесс постоянно живым, как Node.js |
| **Bus (Шина)** | Посредник для передачи сообщений между модулями | Корпоративная почта — отправитель и получатель не знают друг о друге |
| **Command** | Объект-намерение изменить состояние системы | Запрос на действие: «создай товар», «удали заказ» |
| **Query** | Объект-запрос на чтение данных без изменений | Вопрос к системе: «покажи товар», «список заказов» |
| **Domain Event** | Факт о произошедшем бизнес-событии | «Товар создан» — свершившийся факт, нельзя отменить |
| **DDD** | Domain-Driven Design — код вокруг бизнес-логики | Код отражает бизнес-процессы, а не техническую структуру БД |
| **AggregateRoot** | Корневой Entity, управляющий группой связанных объектов | «Корень» бизнес-объекта — через него происходят все изменения |
| **Data Mapper** | ORM-паттерн: Entity ≠ строка в БД, маппинг отдельный | Entity — это бизнес-объект. Mapper знает как его сохранить в БД |
| **Active Record** | ORM-паттерн: Entity == строка в БД, смешивает всё | Anti-pattern при DDD: бизнес-логика перемешана с SQL |
| **Cycle ORM** | PHP Data Mapper ORM с PHPStan level 10 | Выбранный ORM для Momo: строгий, типизированный, Swoole-совместимый |
| **PSR-7** | PHP-стандарт для HTTP-объектов | Любой PHP-фреймворк понимает эти объекты — Request и Response |
| **IoC Container** | Инверсия управления — контейнер, создающий объекты с зависимостями | «Фабрика всего» — говоришь «дай мне ProductHandler», он сам подтягивает зависимости |
| **ServiceProvider** | Класс, регистрирующий модуль в системе | Инструкция по установке — говорит фреймворку, что модуль предоставляет |
| **C-расширение** | Код на C, компилируемый в бинарный PHP-модуль | Заменяем деревянный мотор на металлический — тот же интерфейс, другая скорость |
| **Coroutine** | Лёгкий поток выполнения внутри одного OS-потока | Жонглёр с несколькими мячами — один человек, несколько задач одновременно |
| **TTM** | Time-to-Market — время от идеи до продакшена | Насколько быстро новая фича доходит до пользователей |
| **SemVer** | Semantic Versioning — MAJOR.MINOR.PATCH | 2.1.3: крупное.новая фича.баг-фикс |
| **UoW** | Unit of Work — все изменения применяются атомарно | Один «пакет» изменений в БД: либо всё сохранилось, либо ничего |
| **CQRS** | Command Query Responsibility Segregation | Разделение на «запись» (Command) и «чтение» (Query) — разные модели |

---

*Momo Framework — построен для тех, кто хочет скорость Go при скорости разработки PHP.*