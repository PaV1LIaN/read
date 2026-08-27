Ниже — большая инструкция “как работать с проектом”. Сохрани её как основу для README/INSTALL, потому что это уже почти документация фреймворка.

1. Что это за проект

Это Laravel-like MVC-фреймворк для разработки внутри коробочного Битрикс24.

Идея такая:

Битрикс24 остаётся основой портала.
Наш фреймворк живёт внутри /local.
Разработчик пишет код почти как в Laravel:
routes → controllers → requests → models → views.

То есть вместо хаоса из отдельных PHP-файлов мы делаем нормальную структуру:

Route::get('/notes', [NoteController::class, 'index'])->name('notes.index');

public function index(): Response
{
    return $this->render('notes/index', [
        'notes' => Note::paginateLatest(),
    ]);
}


---

2. Структура проекта

После распаковки архив выглядит примерно так:

bitrix-laravel-mvc/
  README.md
  INSTALL.md
  composer.json
  .gitignore

  docs/
    urlrewrite.example.php
    nginx-security.example.conf

  local/
    mvc/
      bootstrap.php
      helpers.php

      Core/
        App.php
        Router.php
        Request.php
        Response.php
        Controller.php
        Model.php
        QueryBuilder.php
        Validator.php
        FormRequest.php
        Migration.php
        Migrator.php
        SchemaBuilder.php
        Seeder.php
        Factory.php
        Middleware.php
        Auth.php
        Role.php
        GateManager.php
        SessionManager.php
        LogManager.php
        ErrorHandler.php
        Env.php
        View.php
        ...

      Support/
        Facades/
          Route.php
          Config.php
          Log.php
          Gate.php
          Session.php
          Schema.php
          App.php

    mvc_demo/
      index.php
      config.php
      routes.php
      .env.example

      Controllers/
      Models/
      Requests/
      Policies/
      Providers/
      Services/

      Database/
        Migrations/
        Seeders/
        Factories/

      Views/
        layouts/
        home/
        notes/
        errors/
        partials/

      assets/
        app.css

      storage/
        logs/
        cache/

Главное:

/local/mvc      — ядро фреймворка.
/local/mvc_demo — пример приложения на этом фреймворке.


---

3. Как установить на сервер Битрикс24

Допустим, корень сайта Битрикса:

/srv/bx/docroot

Тогда нужно, чтобы после загрузки получилось:

/srv/bx/docroot/local/mvc
/srv/bx/docroot/local/mvc_demo

То есть папку local из архива нужно совместить с текущей /local.

Пример:

cd /srv/bx/docroot
unzip bitrix-laravel-mvc.zip

Если архив распаковался в папку bitrix-laravel-mvc, то можно скопировать так:

cp -r bitrix-laravel-mvc/local/mvc /srv/bx/docroot/local/
cp -r bitrix-laravel-mvc/local/mvc_demo /srv/bx/docroot/local/


---

4. Настройка .env

В GitHub лежит только пример:

/local/mvc_demo/.env.example

Реальный .env нужно создать на сервере:

cp /srv/bx/docroot/local/mvc_demo/.env.example /srv/bx/docroot/local/mvc_demo/.env

Пример .env для разработки:

APP_ENV=local
APP_DEBUG=true
APP_NAME="MVC Demo"

DB_DEFAULT=projects
DB_PROJECTS_SCHEMA=mvc

LOG_LEVEL=debug

Пример .env для production:

APP_ENV=production
APP_DEBUG=false
APP_NAME="MVC Demo"

DB_DEFAULT=projects
DB_PROJECTS_SCHEMA=mvc

LOG_LEVEL=warning

Самое важное:

APP_DEBUG=true  — можно на разработке.
APP_DEBUG=false — обязательно на production.

Если APP_DEBUG=true, при ошибках будут видны trace, пути файлов и технические детали. На боевом сервере так нельзя.


---

5. Настройка urlrewrite.php

Чтобы красивые URL работали через Битрикс, нужно добавить правило в:

/srv/bx/docroot/urlrewrite.php

Пример правила:

999002 =>
array (
  'CONDITION' => '#^/local/mvc_demo/?(.*)$#',
  'RULE' => 'route=/$1',
  'ID' => '',
  'PATH' => '/local/mvc_demo/index.php',
  'SORT' => 1,
),

Что это значит:

/local/mvc_demo/notes
/local/mvc_demo/admin/users
/local/mvc_demo/debug/routes

все эти адреса будут попадать в:

/local/mvc_demo/index.php

А дальше наш Router сам поймёт, какой контроллер вызвать.


---

6. Настройка Nginx / Angie безопасности

Потому что .env и storage лежат внутри docroot, их нужно закрыть от браузера.

Для Nginx/Angie добавь:

location ~ /\. {
    deny all;
    access_log off;
    log_not_found off;
}

location ^~ /local/mvc_demo/storage/ {
    deny all;
    access_log off;
    log_not_found off;
}

Проверка:

nginx -t
systemctl reload nginx

Если Angie:

angie -t
systemctl reload angie

Нельзя, чтобы открывались такие URL:

/local/mvc_demo/.env
/local/mvc_demo/storage/logs/app-2026-08-27.log


---

7. Как запускается приложение

Главная точка входа demo-проекта:

/local/mvc_demo/index.php

Там задаются константы проекта:

define('LOCAL_MVC_PROJECT_ROOT', __DIR__);
define('LOCAL_MVC_PROJECT_URL', '/local/mvc_demo');
define('LOCAL_MVC_PROJECT_NAMESPACE', 'Local\\MvcDemo\\');

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

\Local\Mvc\Core\App::run();

Простыми словами:

index.php говорит ядру:
1. Где лежит проект.
2. Какой namespace у проекта.
3. Какой URL-префикс у проекта.
4. Запусти приложение.


---

8. Как работает цепочка запроса

Например пользователь открыл:

/local/mvc_demo/notes

Цепочка такая:

Браузер
↓
Битрикс urlrewrite.php
↓
/local/mvc_demo/index.php
↓
/local/mvc/bootstrap.php
↓
App::run()
↓
routes.php
↓
Router
↓
NoteController@index
↓
Model / QueryBuilder
↓
View
↓
HTML пользователю

То есть как в Laravel:

Route → Controller → Model → View


---

9. Основные URL demo-приложения

После установки можно проверить:

/local/mvc_demo
/local/mvc_demo/about
/local/mvc_demo/notes
/local/mvc_demo/notes/trash
/local/mvc_demo/debug/routes
/local/mvc_demo/migrations
/local/mvc_demo/seeders
/local/mvc_demo/form
/local/mvc_demo/method
/local/mvc_demo/ajax

Самые важные:

/local/mvc_demo/debug/routes

показывает зарегистрированные маршруты.

/local/mvc_demo/notes

показывает CRUD заметок.

/local/mvc_demo/migrations

запуск миграций.

/local/mvc_demo/seeders

запуск сидеров.


---

10. Маршруты

Файл маршрутов:

/local/mvc_demo/routes.php

Пример простого маршрута:

use Local\Mvc\Support\Facades\Route;
use Local\MvcDemo\Controllers\HomeController;

Route::get('/', [HomeController::class, 'index'])->name('home');

Пример POST:

Route::post('/form/send', [FormController::class, 'send'])
    ->middleware(['auth', 'csrf'])
    ->name('form.send');

Пример resource:

Route::resource('/notes', NoteController::class, [
    'only' => [
        'index',
        'store',
        'edit',
        'update',
        'destroy',
    ],
    'middleware' => ['csrf'],
]);

Это создаёт маршруты:

GET     /notes                   notes.index
POST    /notes                   notes.store
GET     /notes/{note}/edit       notes.edit
PUT     /notes/{note}            notes.update
DELETE  /notes/{note}            notes.destroy


---

11. Контроллеры

Контроллеры лежат здесь:

/local/mvc_demo/Controllers

Пример:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class HomeController extends Controller
{
    public function index(): Response
    {
        return $this->render('home/index', [
            'title' => 'Главная',
        ]);
    }
}

Главное:

return $this->render('home/index', [...]);

Это значит:

Открой view:
/local/mvc_demo/Views/home/index.php


---

12. Views

Views лежат здесь:

/local/mvc_demo/Views

Например:

/local/mvc_demo/Views/home/index.php
/local/mvc_demo/Views/notes/index.php
/local/mvc_demo/Views/notes/edit.php
/local/mvc_demo/Views/layouts/app.php

В контроллере:

return $this->render('notes/index', [
    'title' => 'Заметки',
    'notes' => $notes,
]);

Во view переменные доступны как обычные PHP-переменные:

<h1><?= e($title) ?></h1>

<?php foreach ($notes as $note): ?>
    <?= e($note['title']) ?>
<?php endforeach; ?>

Для защиты от XSS использовать:

e($value)

Не выводить пользовательские данные напрямую так:

<?= $value ?>

Лучше:

<?= e($value) ?>


---

13. Layout

Главный layout:

/local/mvc_demo/Views/layouts/app.php

Он подключается из Controller::render().

Туда обычно добавляют:

общий HTML
меню
CSS
контейнер страницы
flash-сообщения
footer


---

14. Partial views

Есть helper:

view()

Пример:

<?= view('partials.pagination', [
    'pagination' => $pagination,
    'routeName' => 'notes.index',
    'routeParams' => [],
    'query' => [],
]) ?>

Он откроет файл:

/local/mvc_demo/Views/partials/pagination.php

То есть:

partials.pagination

превращается в:

Views/partials/pagination.php


---

15. Request

Класс запроса:

request()

Пример:

$page = (int)request('page', 1);
$search = trim((string)request('q', ''));

Можно получить объект Request:

$request = request();

Можно читать данные:

request('name')
request('page', 1)


---

16. FormRequest

FormRequest — это Laravel-like валидация формы.

Файлы лежат здесь:

/local/mvc_demo/Requests

Пример:

<?php

namespace Local\MvcDemo\Requests;

use Local\Mvc\Core\FormRequest;

class StoreNoteRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => ['required', 'min:2', 'max:255'],
            'body' => ['required', 'min:2'],
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => 'Введите название.',
            'body.required' => 'Введите текст.',
        ];
    }
}

В контроллере:

public function store(StoreNoteRequest $request): Response
{
    $data = $request->validated();

    Note::create([
        'title' => $data['title'],
        'body' => $data['body'],
    ]);

    return redirect()->route('notes.index');
}

То есть контроллер получает уже проверенные данные.


---

17. Validator

Сейчас поддерживаются базовые правила:

required
min
max
integer
email

Пример:

Validator::validate($data, [
    'email' => ['required', 'email'],
    'age' => ['required', 'integer'],
]);

Дальше для production нужно расширять:

nullable
numeric
string
array
date
in
unique
exists
confirmed
same
different


---

18. CSRF

В формах нужно добавлять:

<?= csrf_field() ?>

Пример:

<form method="post" action="<?= e(route('notes.store')) ?>">
    <?= csrf_field() ?>

    <input type="text" name="title">

    <button type="submit">Создать</button>
</form>

Для PUT/PATCH/DELETE используется:

<?= method_field('DELETE') ?>

Пример:

<form method="post" action="<?= e(route('notes.destroy', ['note' => $noteId])) ?>">
    <?= csrf_field() ?>
    <?= method_field('DELETE') ?>

    <button type="submit">Удалить</button>
</form>

Почему так:

HTML-формы умеют только GET и POST.
Поэтому DELETE передаём через скрытое поле _method.


---

19. Redirect

Можно писать:

return redirect()->route('notes.index');

Или:

return redirect()->to('/local/mvc_demo/notes');

Или назад:

return redirect()->back();


---

20. Flash-сообщения

В контроллере:

Flash::success('Заметка создана.');
Flash::error('Ошибка сохранения.');
Flash::info('Информация.');

После redirect сообщение появится на следующей странице.

Во view $flash уже передаётся автоматически через Controller::render().


---

21. Old values

Если форма не прошла валидацию, старые значения можно вернуть:

value="<?= e(old('title')) ?>"

Пример:

<input
    type="text"
    name="title"
    value="<?= e(old('title')) ?>"
>


---

22. Модели

Модели лежат здесь:

/local/mvc_demo/Models

Пример Note:

class Note extends Model
{
    protected static string $connection = 'projects';
    protected static string $table = 'mvc.mvc_demo_notes';
    protected static string $primaryKey = 'id';

    protected static bool $timestamps = true;
    protected static bool $softDeletes = true;

    protected static array $fillable = [
        'title',
        'body',
        'created_at',
        'updated_at',
        'deleted_at',
    ];
}

Создать запись:

Note::create([
    'title' => 'Тест',
    'body' => 'Текст',
]);

Найти:

$note = Note::findModel(5);

Обновить:

$note->update([
    'title' => 'Новое название',
]);

Удалить:

$note->delete();

Если включён soft delete, физического удаления не будет. Заполнится:

deleted_at


---

23. QueryBuilder

Пример:

$notes = Note::query()
    ->whereLike('title', 'test')
    ->orderBy('id', 'desc')
    ->limit(10)
    ->get();

Пагинация:

$result = Note::query()
    ->orderBy('id', 'desc')
    ->paginate($page, 10);

Условный фильтр через when():

$result = Note::query()
    ->when($search !== '', function ($query) use ($search) {
        $query->whereLike('title', $search);
    })
    ->orderBy('id', 'desc')
    ->paginate($page, 10);

Группировка условий:

$query->where(function ($query) use ($search) {
    $query
        ->whereLike('title', $search)
        ->orWhereLike('body', $search);
});


---

24. Пагинация

В контроллере:

$page = (int)request('page', 1);
$search = trim((string)request('q', ''));

$result = Note::paginateLatest($page, 10, $search);

return $this->render('notes/index', [
    'notes' => $result['items'],
    'pagination' => $result['pagination'],
    'search' => $search,
]);

Во view:

<?= view('partials.pagination', [
    'pagination' => $pagination ?? [],
    'routeName' => 'notes.index',
    'routeParams' => [],
    'query' => !empty($search) ? ['q' => $search] : [],
]) ?>


---

25. Route Model Binding

Это важная Laravel-like фича.

Маршрут:

/notes/{note}/edit

Контроллер:

public function edit(Note $note): Response
{
    return $this->render('notes/edit', [
        'note' => $note->normalized(),
    ]);
}

То есть в URL приходит ID:

/notes/5/edit

А в контроллер приходит уже объект:

Note $note

Если записи нет — будет 404.


---

26. Policies / Gate

Policies лежат здесь:

/local/mvc_demo/Policies

Пример:

class NotePolicy
{
    public function viewAny(string $modelClass): bool
    {
        return Auth::check();
    }

    public function create(string $modelClass): bool
    {
        return Auth::isAdmin();
    }

    public function update(Note|string $note): bool
    {
        return Auth::isAdmin();
    }

    public function delete(Note|string $note): bool
    {
        return Auth::isAdmin();
    }
}

Подключение policy в config.php:

'auth' => [
    'policies' => [
        \Local\MvcDemo\Models\Note::class => \Local\MvcDemo\Policies\NotePolicy::class,
    ],
],

В контроллере:

$this->authorize('update', $note);

Во view:

<?php if (can('delete', \Local\MvcDemo\Models\Note::class)): ?>
    <button>Удалить</button>
<?php endif; ?>

Главная мысль:

Middleware проверяет общий доступ.
Policy проверяет конкретное действие над конкретной моделью.


---

27. Middleware

Middleware aliases задаются в config/App.

Примеры middleware:

auth
admin
csrf
group
groups
role
roles

Использование:

Route::post('/notes', [NoteController::class, 'store'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.store');

Группа:

Route::prefix('/admin')
    ->middleware(['auth', 'admin'])
    ->name('admin.')
    ->group(function () {
        Route::get('/', [AdminController::class, 'dashboard'])->name('dashboard');
        Route::get('/users', [AdminController::class, 'users'])->name('users');
    });


---

28. Service Container

Контейнер позволяет автоматически подставлять зависимости.

Пример:

public function users(UserService $userService): Response
{
    $users = $userService->latest();

    return $this->render('admin/users', [
        'users' => $users,
    ]);
}

Тебе не нужно вручную делать:

$userService = new UserService();

Контейнер сам создаст объект.


---

29. Service Providers

Provider лежит здесь:

/local/mvc_demo/Providers/AppServiceProvider.php

Пример:

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->singleton(DemoGreetingService::class, function () {
            return new DemoGreetingService('Привет из ServiceProvider');
        });

        $this->app->bind(UserService::class, UserService::class);
    }

    public function boot(): void
    {
        //
    }
}

Provider подключается в config.php:

'providers' => [
    \Local\MvcDemo\Providers\AppServiceProvider::class,
],


---

30. Facades

Есть Laravel-like facades:

use Local\Mvc\Support\Facades\Route;
use Local\Mvc\Support\Facades\Log;
use Local\Mvc\Support\Facades\Gate;
use Local\Mvc\Support\Facades\Session;
use Local\Mvc\Support\Facades\Config;
use Local\Mvc\Support\Facades\Schema;

Примеры:

Log::info('Заметка создана');

Gate::allows('update', $note);

Session::put('key', 'value');

Config::get('debug');


---

31. Session

Можно использовать helper:

session()->put('test', 123);

$value = session('test');

Или facade:

Session::put('test', 123);
$value = Session::get('test');

Flash использует SessionManager внутри.


---

32. Логи

Логи пишутся сюда:

/local/mvc_demo/storage/logs/app-YYYY-MM-DD.log

Пример:

Log::info('Заметка создана', [
    'note_id' => $noteId,
]);

Ошибка:

Log::error('Ошибка сохранения заметки', [
    'data' => $data,
]);

Исключение:

try {
    //
} catch (\Throwable $e) {
    Log::exception($e, 'Ошибка обработки');
}

Важно:

password, token, csrf, sessid и похожие ключи маскируются как [hidden].


---

33. Ошибки

Есть production-safe ErrorHandler.

Обрабатываются:

ValidationException → 422 или redirect back
AuthorizationException → 403
ModelNotFoundException → 404
HttpException → нужный HTTP-код
ROUTE_NOT_FOUND → 404
Остальное → 500

Можно вручную вызвать:

abort(404);
abort(403, 'Нет доступа');
abort_if(!$user, 404);
abort_unless(Auth::check(), 403);

Страница ошибок:

/local/mvc_demo/Views/errors/error.php


---

34. Миграции

Миграции лежат здесь:

/local/mvc_demo/Database/Migrations

Пример:

<?php

use Local\Mvc\Core\Migration;
use Local\Mvc\Support\Facades\Schema;
use Local\Mvc\Core\Blueprint;

return new class extends Migration {
    protected string $connection = 'projects';

    public function up(): void
    {
        Schema::connection('projects')->create('mvc_demo_notes', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('body')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::connection('projects')->dropIfExists('mvc_demo_notes');
    }
};

Запуск через страницу:

/local/mvc_demo/migrations

Там есть кнопка запуска.


---

35. Seeders

Seeders лежат здесь:

/local/mvc_demo/Database/Seeders

Пример:

class DemoNotesSeeder extends Seeder
{
    public function run(): void
    {
        if (Note::count() > 0) {
            return;
        }

        Note::factory()->count(3)->create();
    }
}

Запуск через:

/local/mvc_demo/seeders


---

36. Factories

Factories лежат здесь:

/local/mvc_demo/Database/Factories

Пример:

class NoteFactory extends Factory
{
    protected string $model = Note::class;

    public function definition(): array
    {
        return [
            'title' => 'Тестовая заметка ' . random_int(1000, 9999),
            'body' => 'Текст заметки',
        ];
    }
}

Использование:

Note::factory()->count(5)->create();


---

37. Как добавить новую страницу

Допустим, нужна страница:

/local/mvc_demo/reports

1. Добавить маршрут

Файл:

/local/mvc_demo/routes.php

Код:

use Local\MvcDemo\Controllers\ReportController;

Route::get('/reports', [ReportController::class, 'index'])
    ->middleware(['auth'])
    ->name('reports.index');

2. Создать контроллер

Файл:

/local/mvc_demo/Controllers/ReportController.php

Код:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class ReportController extends Controller
{
    public function index(): Response
    {
        return $this->render('reports/index', [
            'title' => 'Отчёты',
        ]);
    }
}

3. Создать view

Файл:

/local/mvc_demo/Views/reports/index.php

Код:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= e($title ?? 'Отчёты') ?>
    </h1>

    <p class="mvc-page-text">
        Здесь будет страница отчётов.
    </p>
</div>

4. Сбросить OPcache

В Битрикс PHP command line:

opcache_reset();
echo 'OPcache reset OK';

5. Проверить

/local/mvc_demo/reports


---

38. Как добавить новый CRUD

Допустим, хотим сущность Task.

Нужны файлы:

routes.php
Controllers/TaskController.php
Models/Task.php
Requests/StoreTaskRequest.php
Requests/UpdateTaskRequest.php
Policies/TaskPolicy.php
Views/tasks/index.php
Views/tasks/edit.php
Database/Migrations/...create_tasks_table.php

Общая схема:

Migration создаёт таблицу.
Model работает с таблицей.
FormRequest проверяет данные.
Policy проверяет права.
Controller управляет логикой.
Views показывают HTML.
Routes связывают URL и controller.


---

39. Как создать новый проект рядом с demo

Сейчас есть:

/local/mvc_demo

Можно создать новый проект:

/local/my_app

Минимальная структура:

/local/my_app/
  index.php
  config.php
  routes.php
  .env
  Controllers/
  Models/
  Requests/
  Policies/
  Providers/
  Services/
  Database/
  Views/
  storage/

В index.php:

<?php

define('LOCAL_MVC_PROJECT_ROOT', __DIR__);
define('LOCAL_MVC_PROJECT_URL', '/local/my_app');
define('LOCAL_MVC_PROJECT_NAMESPACE', 'Local\\MyApp\\');

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

\Local\Mvc\Core\App::run();

Namespace проекта будет:

namespace Local\MyApp\Controllers;

А autoload сам поймёт:

Local\MyApp\ → /local/my_app/


---

40. Работа с Git

Для разработки лучше использовать ветки.

Основная ветка:

main

Новая фича:

git checkout -b feature/reports

После изменений:

git status
git add .
git commit -m "Add reports page"
git push -u origin feature/reports

Потом на GitHub создать Pull Request.

Если работаешь один, можно пушить сразу в main, но для production лучше привыкать к веткам.


---

41. Как обновлять проект на сервере

Если репозиторий уже загружен на сервер:

cd /srv/bx/docroot/bitrix-laravel
git pull

Но если проект лежит прямо в /srv/bx/docroot/local/mvc, лучше делать осторожно:

cd /srv/bx/docroot
cp -r local/mvc local/mvc_backup_$(date +%F_%H-%M)
cp -r local/mvc_demo local/mvc_demo_backup_$(date +%F_%H-%M)

Потом обновлять.


---

42. Что нельзя заливать в GitHub

Нельзя заливать:

.env
storage/logs/*.log
пароли
токены
дампы базы
реальные конфиги подключения к БД
pg_master.php с доступами

В .gitignore это должно быть закрыто:

.env
*.log
local/mvc_demo/storage/logs/*
!local/mvc_demo/storage/logs/.gitkeep


---

43. Частые ошибки

Ошибка: страница не найдена

Проверь:

urlrewrite.php
routes.php
/local/mvc_demo/index.php

Также открой:

/local/mvc_demo/debug/routes

и посмотри, зарегистрирован ли маршрут.

Ошибка: View not found

Значит контроллер вызывает:

$this->render('reports/index')

а файла нет:

/local/mvc_demo/Views/reports/index.php

Ошибка: Route not found

Значит вызываешь:

route('notes.index')

но маршрута с таким именем нет.

Проверь:

/local/mvc_demo/debug/routes

Ошибка: CSRF

Проверь, что в форме есть:

<?= csrf_field() ?>

и маршрут использует middleware:

->middleware(['csrf'])

Ошибка: Argument #1 must be Note, string given

Это значит сломался Route Model Binding или порядок обработки параметров в Container.

Нужна логика:

сначала типизированные классы,
потом простые route-параметры.

Ошибка: пагинация не отображается

Проверь $pagination:

<pre><?= e(print_r($pagination ?? null, true)) ?></pre>

Если total_pages = 1, пагинация не нужна и не показывается.


---

44. Production checklist

Перед использованием на production:

1. APP_DEBUG=false
2. LOG_LEVEL=warning
3. .env закрыт от браузера
4. storage закрыт от браузера
5. urlrewrite.php настроен
6. Права на storage/logs выданы веб-пользователю
7. Все формы с CSRF
8. Все опасные действия через Policy
9. Все пользовательские данные выводятся через e()
10. Ошибки 403/404/500 показываются красиво
11. Реальные доступы не лежат в GitHub
12. Проверены migrations/seeders
13. Сделан backup перед обновлением


---

45. Как мыслить при разработке

Когда делаешь новую функцию, иди в таком порядке:

1. URL — какой адрес нужен?
2. Route — какой маршрут?
3. Controller — какой метод обработает запрос?
4. Request — нужна ли валидация?
5. Policy — нужны ли права?
6. Model — с какой таблицей работаем?
7. View — что показываем?
8. Flash/Redirect — куда отправляем после действия?
9. Logs — что важно записать?
10. Tests/check — как проверить?

Например “создание заявки”:

URL:
GET /requests
POST /requests

Route:
Route::resource('/requests', RequestController::class)

Controller:
RequestController@index
RequestController@store

FormRequest:
StoreRequestRequest

Model:
UserRequest

Policy:
UserRequestPolicy

Views:
requests/index.php
requests/edit.php


---

46. Что ещё нужно доделать, чтобы стало ещё ближе к Laravel

Сейчас фреймворк уже похож на Laravel по основным идеям, но для production дальше нужно развивать:

1. Artisan-аналог:
   mvc make:controller
   mvc make:model
   mvc make:migration
   mvc migrate
   mvc seed

2. Больше Validator-правил:
   nullable
   numeric
   date
   array
   unique
   exists

3. QueryBuilder:
   join
   leftJoin
   whereIn
   whereBetween
   groupBy
   having

4. Model relations:
   hasMany
   belongsTo
   hasOne

5. Cache:
   cache()->get()
   cache()->put()
   cache()->remember()

6. Config cache для production.

7. Нормальные error views:
   errors/403.php
   errors/404.php
   errors/500.php

8. API resources:
   JsonResource-like слой.

9. Тесты.

10. Документация для разработчиков.


---

47. Самое короткое правило работы

Запомни главную схему:

routes.php
  ↓
Controller
  ↓
FormRequest / Policy
  ↓
Model / QueryBuilder
  ↓
View

И главный Laravel-like пример:

Route::resource('/notes', NoteController::class);

public function update(Note $note, UpdateNoteRequest $request): Response
{
    $this->authorize('update', $note);

    $note->update($request->validated());

    Flash::success('Заметка обновлена.');

    return redirect()->route('notes.index');
}

Это и есть стиль, к которому мы ведём проект.