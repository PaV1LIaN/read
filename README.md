Идём дальше. Раз у нас появились миграции, следующий Laravel-like шаг — rollback миграций.

В Laravel есть:

php artisan migrate:rollback

Это значит:

Откатить последнюю применённую пачку миграций.

У нас будет кнопка:

Откатить последнюю пачку

Важно: наша миграция down() удаляет таблицу mvc.mvc_demo_notes, поэтому rollback удалит таблицу заметок. Это учебный механизм, на боевых таблицах нажимать аккуратно.


---

1. Замени /local/mvc/Core/Migrator.php

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

    public function rollback(string $path): array
    {
        $this->ensureMigrationTable();

        $lastBatch = $this->lastBatch();

        if ($lastBatch <= 0) {
            return [
                [
                    'migration' => '',
                    'status' => 'empty',
                    'message' => 'Нет миграций для отката',
                ],
            ];
        }

        $rows = Db::fetchAll("
            SELECT migration
            FROM {$this->table}
            WHERE batch = :batch
            ORDER BY id DESC
        ", [
            'batch' => $lastBatch,
        ], $this->connection);

        $files = $this->migrationFilesByName($path);
        $results = [];

        foreach ($rows as $row) {
            $name = (string)($row['migration'] ?? $row['MIGRATION'] ?? '');

            if ($name === '') {
                continue;
            }

            if (empty($files[$name])) {
                $results[] = [
                    'migration' => $name,
                    'status' => 'file_missing',
                    'message' => 'Файл миграции не найден',
                ];

                continue;
            }

            $migration = require $files[$name];

            if (!$migration instanceof Migration) {
                throw new RuntimeException('MIGRATION_MUST_RETURN_MIGRATION_OBJECT: ' . $files[$name]);
            }

            $migration->down();

            $this->deleteMigration($name);

            $results[] = [
                'migration' => $name,
                'status' => 'rolled_back',
                'message' => 'Откат выполнен',
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
        return $this->lastBatch() + 1;
    }

    private function lastBatch(): int
    {
        $max = Db::value("
            SELECT COALESCE(MAX(batch), 0)
            FROM {$this->table}
        ", [], $this->connection);

        return (int)$max;
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

    private function deleteMigration(string $name): void
    {
        Db::execute("
            DELETE FROM {$this->table}
            WHERE migration = :migration
        ", [
            'migration' => $name,
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

    private function migrationFilesByName(string $path): array
    {
        $result = [];

        foreach ($this->migrationFiles($path) as $file) {
            $result[basename($file, '.php')] = $file;
        }

        return $result;
    }
}


---

2. Обнови /local/mvc_demo/Controllers/MigrationController.php

Полностью замени файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Migrator;
use Local\Mvc\Core\Response;

class MigrationController extends Controller
{
    private function migrationsPath(): string
    {
        return $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Database/Migrations';
    }

    public function index(Migrator $migrator): Response
    {
        return $this->render('migrations/index', [
            'title' => 'Миграции',
            'migrations' => $migrator->status($this->migrationsPath()),
        ]);
    }

    public function run(Migrator $migrator): Response
    {
        $results = $migrator->run($this->migrationsPath());

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

    public function rollback(Migrator $migrator): Response
    {
        $results = $migrator->rollback($this->migrationsPath());

        $rolledBack = 0;

        foreach ($results as $result) {
            if (($result['status'] ?? '') === 'rolled_back') {
                $rolledBack++;
            }
        }

        if ($rolledBack > 0) {
            Flash::success('Откат выполнен. Миграций откатили: ' . $rolledBack);
        } else {
            Flash::success('Откатывать нечего.');
        }

        return redirect()->route('migrations.index');
    }
}


---

3. Обнови /local/mvc_demo/Views/migrations/index.php

Полностью замени файл:

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
        Это Laravel-like миграции. Они создают и откатывают таблицы без ручной работы в pgAdmin.
    </p>

    <?php if (!empty($flash)): ?>
        <?php foreach ($flash as $item): ?>
            <div class="mvc-info" style="border-color:#bbf7d0;background:#f0fdf4;color:#166534;">
                <?= e($item['message'] ?? '') ?>
            </div>
        <?php endforeach; ?>
    <?php endif; ?>

    <div class="mvc-info">
        <div style="display:flex;gap:10px;align-items:center;flex-wrap:wrap;">
            <form method="post" action="<?= e(route('migrations.run')) ?>" style="margin:0;">
                <?= csrf_field() ?>

                <button
                    type="submit"
                    style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#2563eb;color:#fff;font-weight:600;cursor:pointer;"
                >
                    Запустить миграции
                </button>
            </form>

            <form method="post" action="<?= e(route('migrations.rollback')) ?>" style="margin:0;">
                <?= csrf_field() ?>

                <button
                    type="submit"
                    onclick="return confirm('Откатить последнюю пачку миграций? Это может удалить таблицы и данные.')"
                    style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#dc2626;color:#fff;font-weight:600;cursor:pointer;"
                >
                    Откатить последнюю пачку
                </button>
            </form>
        </div>
    </div>

    <div class="mvc-info" style="border-color:#fde68a;background:#fffbeb;color:#92400e;">
        <b>Важно:</b>
        rollback запускает метод <span class="mvc-code">down()</span> у миграции.
        В нашей demo-миграции он удаляет таблицу <span class="mvc-code">mvc.mvc_demo_notes</span>.
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

4. Добавь маршрут rollback в /local/mvc_demo/routes.php

Рядом с миграциями у тебя есть:

Route::get('/migrations', [MigrationController::class, 'index'])
    ->middleware(['auth', 'admin'])
    ->name('migrations.index');

Route::post('/migrations/run', [MigrationController::class, 'run'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('migrations.run');

Добавь ниже:

Route::post('/migrations/rollback', [MigrationController::class, 'rollback'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('migrations.rollback');


---

5. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/migrations

Проверь:

1. Если миграция применена — будет статус "Применена".
2. Нажми "Откатить последнюю пачку".
3. Статус должен стать "Не применена".
4. Нажми "Запустить миграции".
5. Статус снова должен стать "Применена".

После rollback страница /notes может сломаться, потому что таблица будет удалена. После повторного запуска миграций /notes снова заработает, но старые заметки будут удалены.


---

Что мы сделали

Теперь у нас есть Laravel-like команды, только через веб-страницу:

migrate          → Запустить миграции
migrate:rollback → Откатить последнюю пачку

И миграция теперь работает полноценно:

public function up(): void
{
    // создать таблицу
}

public function down(): void
{
    // удалить таблицу
}

Это ещё один большой шаг к пониманию Laravel.