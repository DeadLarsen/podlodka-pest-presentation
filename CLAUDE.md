# CLAUDE.md

Этот файл — контекст проекта TaskFlow для Claude Code и других AI-ассистентов. Claude читает его автоматически при старте сессии в корне проекта.

## Проект

**TaskFlow** — SaaS для управления задачами.

- **Язык:** PHP 8.3 (strict_types=1 во всех файлах)
- **Фреймворк:** Laravel 11
- **Архитектура:** модульный монолит (DDD-lite)
- **Модули:** Auth, Projects, Tasks, Notifications, Billing
- **БД:** PostgreSQL (production), SQLite in-memory (tests)
- **Очереди:** Redis + Laravel Queue
- **Тесты:** Pest 4.x

## Структура

```
app/
├── Models/          # Eloquent-модели — только данные
├── Services/        # Бизнес-логика, НЕ зависят от Http-слоя
├── Http/
│   ├── Controllers/ # Только маршрутизация + валидация
│   └── Requests/    # FormRequest для валидации
├── Actions/         # Single-action классы для команд
├── DataTransferObjects/ # Readonly DTO
├── Enums/           # PHP 8.1+ enums для статусов
└── Exceptions/      # Доменные исключения

tests/
├── Architecture/    # Arch-тесты (запуск первым)
├── Unit/            # Чистая логика без БД
├── Feature/         # API + БД + сервисы
└── Browser/         # E2E через pest-plugin-browser
```

---

## Тестирование: правила для AI

### Всегда используй Pest, не PHPUnit

Мы на Pest 4. Никогда не генерируй тесты в стиле PHPUnit-классов.

**✓ Правильно:**
```php
it('marks task as done', function () {
    $task = Task::factory()->create(['status' => 'in_progress']);
    TaskService::complete($task);
    expect($task->fresh()->status)->toBe('done');
});
```

**✗ Неправильно:**
```php
public function test_it_marks_task_as_done(): void
{
    $this->assertEquals('done', ...);
}
```

### Стиль ожиданий

- Используй `expect()` API, **никогда** `$this->assert*()`
- Используй цепочки: `expect($user)->name->toBe('X')->email->toContain('@')`
- Для отрицаний — модификатор `not->`, не `assertNotEquals`
- Для коллекций — `each->`, не `foreach` внутри теста

### Структура файла

1. `use`-импорты сверху (без `namespace` — Pest автоматически)
2. `beforeEach(function () { ... })` для общего setup
3. `describe('ClassName::method', function () { ... })` — группировка по методам
4. Внутри `describe` — `it()` для отдельных сценариев

### Datasets — обязательны для множественных кейсов

Если тестируешь одно поведение на разных данных — используй `->with()`, не копируй тест.

```php
it('validates status transition', function (string $from, string $to, bool $allowed) {
    // ...
})->with([
    'todo → in_progress' => ['todo', 'in_progress', true],
    'done → todo'        => ['done', 'todo', false],
]);
```

**Всегда именуй кейсы** (ключ массива) — Pest покажет имя в выводе.

### Покрытие для mutation testing

Каждый тест бизнес-логики должен указывать `->covers()`:

```php
it('calculates priority', function () {
    // ...
})->covers(TaskPriorityCalculator::class);
```

Без этого `pest --mutate` будет запускать все тесты на каждую мутацию — это в 10+ раз медленнее.

### Factories, а не new

Модели в тестах создавай через factory:

```php
// ✓
$user = User::factory()->create();
$task = Task::factory()->for($user)->create();

// ✗
$user = new User(['email' => 'test@test.com']);
$user->save();
```

### Helper `actingAs()` для API-тестов

Для Feature-тестов с аутентификацией:

```php
beforeEach(function () {
    $this->user = User::factory()->create();
    $this->actingAs($this->user);
});
```

### Мокать только внешние зависимости

- **Мокай:** HTTP-клиенты, платёжные шлюзы, внешние API
- **Не мокай:** репозитории, Eloquent-модели, свои сервисы
- Для Notification, Mail, Queue используй `::fake()` из Laravel

```php
// ✓
Notification::fake();
Queue::fake();

// ✗ — не мокай Eloquent
$mock = Mockery::mock(Task::class);
```

### Browser-тесты (Pest v4)

Используй `visit()` из `pest-plugin-browser`, а не Dusk.

```php
it('creates task via UI', function () {
    $user = User::factory()->create();

    visit(route('login'))
        ->type('email', $user->email)
        ->type('password', 'password')
        ->press('Sign In')
        ->assertSee('Dashboard');
});
```

**Известные баги Pest v4** (держи в голове):
- Issue #1605: silent crash при навигации — избегай `window.location.href`, используй `->navigate()` или `visit()` заново
- Issue #1593: субдомены не работают — тесты с `admin.taskflow.test` пиши на `localhost`
- Issue #1483: `--parallel` в v4 может давать ложный exit code 1 — в CI используй `--parallel` только в последовательных джобах

---

## Архитектурные правила (см. `tests/Architecture/`)

При генерации кода соблюдай эти инварианты — arch-тесты их проверяют:

1. **Модели — только в `App\Models`** и наследуют `Illuminate\Database\Eloquent\Model`
2. **Сервисы не зависят от `App\Http`** (ни от контроллеров, ни от Request)
3. **Контроллеры не обращаются к моделям напрямую** — только через сервисы или Actions
4. **Никакого `dd()`, `dump()`, `var_dump()`, `die()`, `exit()`** в `app/`
5. **Все доменные исключения** в `App\Exceptions` и наследуют `App\Exceptions\DomainException`
6. **DTO — readonly классы** с `final`, все свойства `public readonly`
7. **Enums — только для статусов** (Task status, Project status, etc.)
8. **Миграции** не содержат `DB::raw()` кроме случаев с комментарием-обоснованием

При добавлении нового правила — добавь arch-тест в `tests/Architecture/RulesTest.php`.

---

## Воркфлоу AI-генерации тестов

Когда я прошу «напиши тесты для X» — делай следующее:

1. **Прочитай сам класс X** и его зависимости. Не угадывай сигнатуры методов.
2. **Проверь существующие тесты** для соседних классов — используй тот же стиль.
3. **Сгенерируй тесты** с покрытием:
    - Happy path (1-2 теста)
    - Edge cases через datasets (3-5 кейсов)
    - Negative scenarios (исключения, ошибки валидации)
4. **Добавь `->covers()`** для каждого теста бизнес-логики.
5. **Запусти `./vendor/bin/pest`** и исправь ошибки самостоятельно.
6. **Запусти `./vendor/bin/pest --mutate --covered-only`** для сгенерированного файла и дополни тесты, если mutation score < 100%.
7. **Сообщи итог:** сколько тестов создано, какой mutation score.

---

## Команды

```bash
# Все тесты параллельно
./vendor/bin/pest --parallel

# Только один модуль
./vendor/bin/pest tests/Feature/Tasks

# С coverage (CI порог 80%)
./vendor/bin/pest --coverage --min=80

# Type coverage
./vendor/bin/pest --type-coverage --min=95

# Mutation testing (медленно, запускай на изменённом)
./vendor/bin/pest --mutate --covered-only

# Architecture-тесты (быстро, первыми в CI)
./vendor/bin/pest --group=architecture

# Browser E2E
./vendor/bin/pest --group=browser

# Watch mode для разработки
./vendor/bin/pest --watch
```

---

## Code style

- **PSR-12** + правила из `.php-cs-fixer.dist.php`
- **Типы обязательны** на всех параметрах и возвращаемых значениях (`--type-coverage --min=95`)
- **Readonly везде, где возможно** (PHP 8.1+)
- **Final классы по умолчанию** — открывай наследование только по необходимости
- **Имена тестов — по-английски** для `it()`, комментарии можно по-русски
- **Строковые ключи datasets** — на английском для CI-логов

---

## Anti-patterns (Claude, не делай так)

1. **Не генерируй `setUp()` и `tearDown()` методы** — используй `beforeEach()`/`afterEach()`
2. **Не пиши `$this->expectException(...)`** — используй `expect(fn() => ...)->toThrow(...)`
3. **Не дублируй тесты** — если 3+ похожих теста, переноси в dataset
4. **Не пиши тесты без `->covers()`** для бизнес-логики
5. **Не используй `Mockery` для своих классов** — мокай только внешние границы
6. **Не создавай классы `SomeTest extends TestCase`** — только функциональный стиль Pest
7. **Не коммить `.phpunit.cache/`** — он в `.gitignore`, но ты его не создавай
8. **Не меняй `phpunit.xml`** без явного запроса — конфигурация уже настроена
9. **Не добавляй `@test` аннотации** — Pest их не требует
10. **Не пиши assert-методы PHPUnit** даже если они доступны через совместимость

---

## Контакты и ресурсы

- **Документация Pest:** https://pestphp.com
- **Внутренний wiki:** `docs/testing-strategy.md`
- **Slack-канал:** #taskflow-dev
- **CODEOWNERS:** см. `.github/CODEOWNERS` для ревью тестов

---

*Файл поддерживается командой. При изменении стратегии тестирования — обновляйте этот файл одновременно с `tests/Architecture/RulesTest.php`.*
