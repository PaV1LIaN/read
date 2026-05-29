Идём дальше. После миграций в Laravel обычно идут Seeders.

Простыми словами:

Migration — создаёт таблицу.
Seeder — заполняет таблицу тестовыми данными.

Например:

Создать таблицу notes
↓
Добавить 3 тестовые заметки


---

1. Создай /local/mvc/Core/Seeder.php

<?php

namespace Local\Mvc\Core;

/**
 * Seeder
 *
 * Laravel-like наполнитель базы тестовыми данными.
 */
abstract class Seeder
{
    abstract public function run(): void;

    /**
     * Запустить другие seeder-классы.
     *
     * Пример:
     * $this->call([
     *     DemoNotesSeeder::class,
     * ]);
     */
    protected function call(array|string $seeders): void
    {
        if (!is_array($seeders)) {
            $seeders = [$seeders];
        }

        foreach ($seeders as $seederClass) {
            $seeder = App::make((string)$seederClass);

            if (!$seeder instanceof Seeder) {
                throw new \RuntimeException('SEEDER_MUST_EXTEND_SEEDER: ' . $seederClass);
            }

            $seeder->run();
        }
    }
}


---

2. Создай /local/mvc/Core/SeederRunner.php

<?php

namespace Local\Mvc\Core;

/**
 * SeederRunner
 *
 * Запускает seeders из config.php.
 */
class SeederRunner
{
    public function run(?array $seeders = null): array
    {
        if ($seeders === null) {
            $seeders = Config::get('database.seeders', []);
        }

        if (!is_array($seeders)) {
            $seeders = [];
        }

        $results = [];

        foreach ($seeders as $seederClass) {
            $seeder = App::make((string)$seederClass);

            if (!$seeder instanceof Seeder) {
                throw new \RuntimeException('SEEDER_MUST_EXTEND_SEEDER: ' . $seederClass);
            }

            $seeder->run();

            $results[] = [
                'seeder' => (string)$seederClass,
                'status' => 'done',
                'message' => 'Seeder выполнен',
            ];
        }

        return $results;
    }
}


---

3. Зарегистрируй SeederRunner в /local/mvc/Core/App.php

Найди блок:

$container->singleton(\Local\Mvc\Core\SchemaBuilder::class, \Local\Mvc\Core\SchemaBuilder::class);

Ниже добавь:

$container->singleton(\Local\Mvc\Core\SeederRunner::class, \Local\Mvc\Core\SeederRunner::class);


---

4. Создай папку seeders

/local/mvc_demo/Database/Seeders/


---

5. Создай /local/mvc_demo/Database/Seeders/DemoNotesSeeder.php

<?php

namespace Local\MvcDemo\Database\Seeders;

use Local\Mvc\Core\Seeder;
use Local\MvcDemo\Models\Note;

class DemoNotesSeeder extends Seeder
{
    public function run(): void
    {
        /**
         * Чтобы не плодить одинаковые записи каждый раз,
         * добавляем тестовые заметки только если таблица пустая.
         */
        if (Note::count() > 0) {
            return;
        }

        Note::create([
            'title' => 'Первая тестовая заметка',
            'body' => 'Эту запись добавил DemoNotesSeeder.',
        ]);

        Note::create([
            'title' => 'Вторая тестовая заметка',
            'body' => 'Seeder нужен, чтобы быстро заполнить таблицу начальными данными.',
        ]);

        Note::create([
            'title' => 'Laravel-like подход',
            'body' => 'Migration создаёт таблицу, Seeder заполняет её данными.',
        ]);
    }
}


---

6. Создай /local/mvc_demo/Database/Seeders/DatabaseSeeder.php

<?php

namespace Local\MvcDemo\Database\Seeders;

use Local\Mvc\Core\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            DemoNotesSeeder::class,
        ]);
    }
}

Это как в Laravel:

DatabaseSeeder
  ↓
вызывает другие seeders


---

7. Обнови /local/mvc_demo/config.php

В блок database добавь seeders.

Должно быть примерно так:

'database' => [
    'default' => 'bitrix',

    'connections' => [
        'bitrix' => [
            'driver' => 'bitrix',
        ],

        'projects' => [
            'driver' => 'pg_master',
            'schema' => 'mvc',
        ],
    ],

    'migrations' => [
        'connection' => 'projects',
        'table' => 'mvc.migrations',
    ],

    'seeders' => [
        \Local\MvcDemo\Database\Seeders\DatabaseSeeder::class,
    ],
],


---

8. Создай /local/mvc_demo/Controllers/SeederController.php

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\Mvc\Core\SeederRunner;

class SeederController extends Controller
{
    public function index(): Response
    {
        return $this->render('seeders/index', [
            'title' => 'Seeders',
            'seeders' => config('database.seeders', []),
        ]);
    }

    public function run(SeederRunner $runner): Response
    {
        $results = $runner->run();

        Flash::success('Seeders выполнены: ' . count($results));

        return redirect()->route('seeders.index');
    }
}


---

9. Создай view /local/mvc_demo/Views/seeders/index.php

Сначала папка:

/local/mvc_demo/Views/seeders/

Файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= e($title ?? 'Seeders') ?>
    </h1>

    <p class="mvc-page-text">
        Seeder заполняет таблицы начальными или тестовыми данными.
        Это похоже на Laravel <span class="mvc-code">php artisan db:seed</span>.
    </p>

    <?php if (!empty($flash)): ?>
        <?php foreach ($flash as $item): ?>
            <div class="mvc-info" style="border-color:#bbf7d0;background:#f0fdf4;color:#166534;">
                <?= e($item['message'] ?? '') ?>
            </div>
        <?php endforeach; ?>
    <?php endif; ?>

    <div class="mvc-info">
        <form method="post" action="<?= e(route('seeders.run')) ?>">
            <?= csrf_field() ?>

            <button
                type="submit"
                style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#2563eb;color:#fff;font-weight:600;cursor:pointer;"
            >
                Запустить seeders
            </button>
        </form>
    </div>

    <div class="mvc-info">
        <?php if (empty($seeders)): ?>
            <p style="margin:0;">Seeders не настроены.</p>
        <?php else: ?>
            <table style="width:100%;border-collapse:collapse;">
                <thead>
                    <tr>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Seeder</th>
                    </tr>
                </thead>

                <tbody>
                    <?php foreach ($seeders as $seeder): ?>
                        <tr>
                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <span class="mvc-code">
                                    <?= e($seeder) ?>
                                </span>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>

    <div class="mvc-info" style="border-color:#fde68a;background:#fffbeb;color:#92400e;">
        <b>Важно:</b>
        перед запуском seeders таблицы должны быть созданы миграциями.
        Сначала проверь страницу <span class="mvc-code">/migrations</span>.
    </div>
</div>


---

10. Добавь маршруты в /local/mvc_demo/routes.php

Вверху добавь:

use Local\MvcDemo\Controllers\SeederController;

Ниже добавь:

Route::get('/seeders', [SeederController::class, 'index'])
    ->middleware(['auth', 'admin'])
    ->name('seeders.index');

Route::post('/seeders/run', [SeederController::class, 'run'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('seeders.run');


---

11. Добавь ссылку в меню

Файл:

/local/mvc_demo/Views/layouts/app.php

Добавь:

<a href="<?= e(route('seeders.index')) ?>">Seeders</a>


---

12. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/seeders

Нажми:

Запустить seeders

Потом открой:

/local/mvc_demo/notes

Если таблица была пустая, должны появиться 3 тестовые заметки.


---

Что мы сделали

Теперь у нас есть Laravel-like seeders:

Database/Migrations — структура базы
Database/Seeders    — тестовые данные

Пример:

class DatabaseSeeder extends Seeder
{
    public function run(): void
    {
        $this->call([
            DemoNotesSeeder::class,
        ]);
    }
}

Главная мысль:

Migration отвечает за таблицы.
Seeder отвечает за данные.

Это ещё один важный кирпич Laravel.