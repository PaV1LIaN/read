Отлично. Дальше делаем очень важную Laravel-like часть — migrations.

В Laravel таблицы создаются не руками в pgAdmin, а через миграции:

php artisan migrate

У нас artisan пока нет, поэтому сделаем проще:

/local/mvc_demo/migrations

И страницу:

/local/mvc_demo/migrations

Там можно будет нажать кнопку Запустить миграции.


---

1. Создай /local/mvc/Core/Migration.php

<?php

namespace Local\Mvc\Core;

/**
 * Migration
 *
 * Laravel-like миграция.
 *
 * up()   — применить миграцию
 * down() — откатить миграцию
 */
abstract class Migration
{
    protected string $connection = 'projects';

    abstract public function up(): void;

    public function down(): void
    {
        //
    }

    protected function statement(string $sql): void
    {
        Db::execute($sql, [], $this->connection);
    }
}


---

2. Создай /local/mvc/Core/Migrator.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

class Migrator
{
    private string $connection;
    private string $table;

    public function __construct()
    {
        $this->connection = (string)Config::get('database.migrations.connection', 'projects');
        $this->table = (string)Config::get('database.migrations.table', 'mvc.migrations');
    }

    public function run(string $path): array
    {
        $this->ensureMigrationTable();

        $ran = $this->ranMigrations();
        $files = $this->migrationFiles($path);

        $batch = $this->nextBatch();
        $results = [];

        foreach ($files as $file) {
            $name = basename($file, '.php');

            if (in_array($name, $ran, true)) {
                $results[] = [
                    'migration' => $name,
                    'status' => 'skipped',
                    'message' => 'Уже применена',
                ];

                continue;
            }

            $migration = require $file;

            if (!$migration instanceof Migration) {
                throw new RuntimeException('MIGRATION_MUST_RETURN_MIGRATION_OBJECT: ' . $file);
            }

            $migration->up();

            $this->recordMigration($name, $batch);

            $results[] = [
                'migration' => $name,
                'status' => 'done',
                'message' => 'Применена',
            ];
        }

        return $results;
    }

    public function status(string $path): array
    {
        $this->ensureMigrationTable();

        $ran = $this->ranMigrations();
        $files = $this->migrationFiles($path);

        $rows = [];

        foreach ($files as $file) {
            $name = basename($file, '.php');

            $rows[] = [
                'migration' => $name,
                'ran' => in_array($name, $ran, true),
            ];
        }

        return $rows;
    }

    private function ensureMigrationTable(): void
    {
        Db::execute("
            CREATE SCHEMA IF NOT EXISTS mvc
        ", [], $this->connection);

        Db::execute("
            CREATE TABLE IF NOT EXISTS {$this->table} (
                id BIGSERIAL PRIMARY KEY,
                migration VARCHAR(255) NOT NULL UNIQUE,
                batch INTEGER NOT NULL,
                ran_at TIMESTAMP NOT NULL DEFAULT NOW()
            )
        ", [], $this->connection);
    }

    private function ranMigrations(): array
    {
        $rows = Db::fetchAll("
            SELECT migration
            FROM {$this->table}
            ORDER BY id ASC
        ", [], $this->connection);

        return array_map(static fn ($row) => (string)($row['migration'] ?? $row['MIGRATION'] ?? ''), $rows);
    }

    private function nextBatch(): int
    {
        $max = Db::value("
            SELECT COALESCE(MAX(batch), 0)
            FROM {$this->table}
        ", [], $this->connection);

        return ((int)$max) + 1;
    }

    private function recordMigration(string $name, int $batch): void
    {
        Db::execute("
            INSERT INTO {$this->table} (migration, batch, ran_at)
            VALUES (:migration, :batch, NOW())
        ", [
            'migration' => $name,
            'batch' => $batch,
        ], $this->connection);
    }

    private function migrationFiles(string $path): array
    {
        if (!is_dir($path)) {
            return [];
        }

        $files = glob(rtrim($path, '/') . '/*.php');

        if (!is_array($files)) {
            return [];
        }

        sort($files);

        return $files;
    }
}


---

3. Обнови /local/mvc_demo/config.php

В блок database добавь migrations.

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
],


---

4. Создай папку миграций

/local/mvc_demo/Database/Migrations/


---

5. Создай миграцию заметок

Файл:

/local/mvc_demo/Database/Migrations/2026_05_29_000001_create_mvc_demo_notes_table.php

Код:

<?php

use Local\Mvc\Core\Migration;

return new class extends Migration {
    protected string $connection = 'projects';

    public function up(): void
    {
        $this->statement("
            CREATE SCHEMA IF NOT EXISTS mvc
        ");

        $this->statement("
            CREATE TABLE IF NOT EXISTS mvc.mvc_demo_notes (
                id BIGSERIAL PRIMARY KEY,
                title VARCHAR(255) NOT NULL,
                body TEXT NULL,
                created_at TIMESTAMP NULL,
                updated_at TIMESTAMP NULL
            )
        ");
    }

    public function down(): void
    {
        $this->statement("
            DROP TABLE IF EXISTS mvc.mvc_demo_notes
        ");
    }
};


---

6. Создай /local/mvc_demo/Controllers/MigrationController.php

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Migrator;
use Local\Mvc\Core\Response;

class MigrationController extends Controller
{
    public function index(Migrator $migrator): Response
    {
        $path = $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Database/Migrations';

        return $this->render('migrations/index', [
            'title' => 'Миграции',
            'migrations' => $migrator->status($path),
        ]);
    }

    public function run(Migrator $migrator): Response
    {
        $path = $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Database/Migrations';

        $results = $migrator->run($path);

        $done = 0;

        foreach ($results as $result) {
            if (($result['status'] ?? '') === 'done') {
                $done++;
            }
        }

        if ($done > 0) {
            Flash::success('Миграции применены: ' . $done);
        } else {
            Flash::success('Новых миграций нет.');
        }

        return redirect()->route('migrations.index');
    }
}


---

7. Создай view /local/mvc_demo/Views/migrations/index.php

Создай папку:

/local/mvc_demo/Views/migrations/

Файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= e($title ?? 'Миграции') ?>
    </h1>

    <p class="mvc-page-text">
        Это Laravel-like миграции. Они создают таблицы в базе без ручного создания через pgAdmin.
    </p>

    <?php if (!empty($flash)): ?>
        <?php foreach ($flash as $item): ?>
            <div class="mvc-info" style="border-color:#bbf7d0;background:#f0fdf4;color:#166534;">
                <?= e($item['message'] ?? '') ?>
            </div>
        <?php endforeach; ?>
    <?php endif; ?>

    <div class="mvc-info">
        <form method="post" action="<?= e(route('migrations.run')) ?>">
            <?= csrf_field() ?>

            <button
                type="submit"
                style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#2563eb;color:#fff;font-weight:600;cursor:pointer;"
            >
                Запустить миграции
            </button>
        </form>
    </div>

    <div class="mvc-info">
        <?php if (empty($migrations)): ?>
            <p style="margin:0;">Файлы миграций не найдены.</p>
        <?php else: ?>
            <table style="width:100%;border-collapse:collapse;">
                <thead>
                    <tr>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Миграция</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Статус</th>
                    </tr>
                </thead>

                <tbody>
                    <?php foreach ($migrations as $migration): ?>
                        <tr>
                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <span class="mvc-code">
                                    <?= e($migration['migration'] ?? '') ?>
                                </span>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?php if (!empty($migration['ran'])): ?>
                                    <span style="color:#166534;font-weight:600;">Применена</span>
                                <?php else: ?>
                                    <span style="color:#92400e;font-weight:600;">Не применена</span>
                                <?php endif; ?>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>
</div>


---

8. Добавь маршруты в /local/mvc_demo/routes.php

Вверху добавь:

use Local\MvcDemo\Controllers\MigrationController;

Ниже добавь:

Route::get('/migrations', [MigrationController::class, 'index'])
    ->middleware(['auth', 'admin'])
    ->name('migrations.index');

Route::post('/migrations/run', [MigrationController::class, 'run'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('migrations.run');


---

9. Добавь ссылку в меню

В /local/mvc_demo/Views/layouts/app.php добавь:

<a href="<?= e(route('migrations.index')) ?>">Migrations</a>


---

10. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/migrations

Нажми:

Запустить миграции

После этого в ProjectsDB должны быть:

mvc.mvc_demo_notes
mvc.migrations


---

Что мы сделали

Раньше таблицу создавали руками:

CREATE TABLE ...

Теперь Laravel-like:

Database/Migrations/2026_05_29_000001_create_mvc_demo_notes_table.php

И фреймворк сам запоминает, какие миграции уже применены, в таблице:

mvc.migrations

Главная мысль:

Migration — файл изменения базы.
Migrator — механизм, который применяет эти файлы.
migrations table — журнал уже применённых миграций.

Это очень важная часть Laravel-мышления.