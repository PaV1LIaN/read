Хорошо.
Теперь делаем первый реальный перенос логики из старого проекта: чтение списка сайтов из JSON через нормальные классы.

На этом шаге получим:

единое место для пути к данным;

единое чтение/запись JSON;

SiteRepository;

SiteService, который уже берёт реальные данные;

новый action site.list.


Пока старый api.php не ломаем.


---

Что делаем сейчас

Создай/замени следующие файлы.

1. /local/modules/its.sitebuilder/lib/config.php

<?php

namespace Its\Sitebuilder;

class Config
{
    public static function dataDir(): string
    {
        return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder';
    }

    public static function sitesFile(): string
    {
        return self::dataDir() . '/sites.json';
    }

    public static function pagesFile(): string
    {
        return self::dataDir() . '/pages.json';
    }

    public static function blocksFile(): string
    {
        return self::dataDir() . '/blocks.json';
    }

    public static function menusFile(): string
    {
        return self::dataDir() . '/menus.json';
    }

    public static function accessFile(): string
    {
        return self::dataDir() . '/access.json';
    }

    public static function templatesFile(): string
    {
        return self::dataDir() . '/templates.json';
    }
}


---

2. /local/modules/its.sitebuilder/lib/infrastructure/storage/jsonstorage.php

Сначала создай папки:

/local/modules/its.sitebuilder/lib/infrastructure/
/local/modules/its.sitebuilder/lib/infrastructure/storage/

Файл:

<?php

namespace Its\Sitebuilder\Infrastructure\Storage;

class JsonStorage
{
    public function ensureDirExists(string $dir): void
    {
        if (is_dir($dir)) {
            return;
        }

        if (!@mkdir($dir, 0775, true) && !is_dir($dir)) {
            throw new \RuntimeException('Не удалось создать директорию: ' . $dir);
        }
    }

    public function readArray(string $file, array $default = []): array
    {
        if (!file_exists($file)) {
            return $default;
        }

        $content = @file_get_contents($file);
        if ($content === false) {
            throw new \RuntimeException('Не удалось прочитать файл: ' . $file);
        }

        $content = trim($content);
        if ($content === '') {
            return $default;
        }

        $decoded = json_decode($content, true);

        if (!is_array($decoded)) {
            throw new \RuntimeException('Некорректный JSON в файле: ' . $file);
        }

        return $decoded;
    }

    public function writeArray(string $file, array $data): void
    {
        $dir = dirname($file);
        $this->ensureDirExists($dir);

        $json = json_encode(
            $data,
            JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES | JSON_PRETTY_PRINT
        );

        if ($json === false) {
            throw new \RuntimeException('Не удалось сериализовать JSON для файла: ' . $file);
        }

        $tmpFile = $file . '.tmp';

        if (@file_put_contents($tmpFile, $json, LOCK_EX) === false) {
            throw new \RuntimeException('Не удалось записать временный файл: ' . $tmpFile);
        }

        if (!@rename($tmpFile, $file)) {
            @unlink($tmpFile);
            throw new \RuntimeException('Не удалось заменить файл: ' . $file);
        }
    }
}


---

3. /local/modules/its.sitebuilder/lib/domain/site/siteentity.php

Сначала создай папки:

/local/modules/its.sitebuilder/lib/domain/
/local/modules/its.sitebuilder/lib/domain/site/

Файл:

<?php

namespace Its\Sitebuilder\Domain\Site;

class SiteEntity
{
    private array $data;

    public function __construct(array $data)
    {
        $this->data = $data;
    }

    public function toArray(): array
    {
        return $this->data;
    }

    public function getId(): int
    {
        return (int)($this->data['id'] ?? 0);
    }

    public function getName(): string
    {
        return (string)($this->data['name'] ?? '');
    }

    public function getSlug(): string
    {
        return (string)($this->data['slug'] ?? '');
    }

    public function isActive(): bool
    {
        return (bool)($this->data['active'] ?? true);
    }
}


---

4. /local/modules/its.sitebuilder/lib/infrastructure/repository/siterepository.php

Создай папку:

/local/modules/its.sitebuilder/lib/infrastructure/repository/

Файл:

<?php

namespace Its\Sitebuilder\Infrastructure\Repository;

use Its\Sitebuilder\Config;
use Its\Sitebuilder\Domain\Site\SiteEntity;
use Its\Sitebuilder\Infrastructure\Storage\JsonStorage;

class SiteRepository
{
    private JsonStorage $storage;

    public function __construct(?JsonStorage $storage = null)
    {
        $this->storage = $storage ?: new JsonStorage();
    }

    /**
     * @return SiteEntity[]
     */
    public function findAll(): array
    {
        $rows = $this->storage->readArray(Config::sitesFile(), []);

        $result = [];

        foreach ($rows as $row) {
            if (!is_array($row)) {
                continue;
            }

            $result[] = new SiteEntity($row);
        }

        usort($result, static function (SiteEntity $a, SiteEntity $b): int {
            return $a->getId() <=> $b->getId();
        });

        return $result;
    }

    public function findById(int $id): ?SiteEntity
    {
        foreach ($this->findAll() as $site) {
            if ($site->getId() === $id) {
                return $site;
            }
        }

        return null;
    }

    public function saveAll(array $sites): void
    {
        $rows = [];

        foreach ($sites as $site) {
            if ($site instanceof SiteEntity) {
                $rows[] = $site->toArray();
            } elseif (is_array($site)) {
                $rows[] = $site;
            }
        }

        $this->storage->writeArray(Config::sitesFile(), $rows);
    }
}


---

5. Полностью замени

/local/modules/its.sitebuilder/lib/service/site/service.php

<?php

namespace Its\Sitebuilder\Service\Site;

use Its\Sitebuilder\Infrastructure\Repository\SiteRepository;

class Service
{
    private SiteRepository $siteRepository;

    public function __construct(?SiteRepository $siteRepository = null)
    {
        $this->siteRepository = $siteRepository ?: new SiteRepository();
    }

    public function ping(): array
    {
        global $USER;

        return [
            'ok' => true,
            'module' => 'its.sitebuilder',
            'service' => static::class,
            'userId' => is_object($USER) ? (int)$USER->GetID() : 0,
            'time' => date('c'),
        ];
    }

    public function list(): array
    {
        $items = [];

        foreach ($this->siteRepository->findAll() as $site) {
            $items[] = $site->toArray();
        }

        return [
            'items' => $items,
            'count' => count($items),
        ];
    }

    public function demoList(): array
    {
        return [
            'items' => [
                [
                    'id' => 1,
                    'name' => 'Демо сайт 1',
                    'slug' => 'demo-1',
                ],
                [
                    'id' => 2,
                    'name' => 'Демо сайт 2',
                    'slug' => 'demo-2',
                ],
            ],
        ];
    }
}


---

6. Полностью замени

/local/tools/its.sitebuilder/ajax.php

<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('NOT_CHECK_PERMISSIONS', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;
use Its\Sitebuilder\Service\Site\Service as SiteService;

header('Content-Type: application/json; charset=UTF-8');

global $USER;

try {
    if (!$USER || !$USER->IsAuthorized()) {
        http_response_code(401);
        echo json_encode([
            'ok' => false,
            'error' => 'NOT_AUTHORIZED',
            'message' => 'Пользователь не авторизован',
        ], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if (!Loader::includeModule('its.sitebuilder')) {
        throw new \RuntimeException('Модуль its.sitebuilder не подключается');
    }

    $action = isset($_REQUEST['action']) ? (string)$_REQUEST['action'] : '';

    $siteService = new SiteService();

    switch ($action) {
        case 'site.ping':
            $result = $siteService->ping();
            break;

        case 'site.list':
            $result = $siteService->list();
            break;

        case 'site.demoList':
            $result = $siteService->demoList();
            break;

        default:
            http_response_code(400);
            echo json_encode([
                'ok' => false,
                'error' => 'UNKNOWN_ACTION',
                'message' => 'Неизвестное действие: ' . $action,
            ], JSON_UNESCAPED_UNICODE);
            exit;
    }

    echo json_encode([
        'ok' => true,
        'data' => $result,
    ], JSON_UNESCAPED_UNICODE);
} catch (\Throwable $e) {
    http_response_code(500);

    echo json_encode([
        'ok' => false,
        'error' => 'EXCEPTION',
        'message' => $e->getMessage(),
        'file' => $e->getFile(),
        'line' => $e->getLine(),
    ], JSON_UNESCAPED_UNICODE);
}


---

7. Полностью замени

/local/sitebuilder/index.php

<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('NOT_CHECK_PERMISSIONS', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;

global $USER;

if (!$USER || !$USER->IsAuthorized()) {
    http_response_code(401);
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>Site Builder</title>
    </head>
    <body>
        <div style="padding:20px;font-family:Arial,sans-serif;color:#b00020;">
            Доступ запрещён. Пользователь не авторизован.
        </div>
    </body>
    </html>
    <?php
    exit;
}

if (!Loader::includeModule('its.sitebuilder')) {
    http_response_code(500);
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>Site Builder</title>
    </head>
    <body>
        <div style="padding:20px;font-family:Arial,sans-serif;color:#b00020;font-weight:bold;">
            Модуль its.sitebuilder не установлен или не подключается.
        </div>
    </body>
    </html>
    <?php
    exit;
}
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ITS Site Builder</title>
</head>
<body style="margin:0;background:#f5f7fb;font-family:Arial,sans-serif;">
    <div style="padding:20px;">
        <h1 style="margin-top:0;">ITS Site Builder</h1>

        <p>Проверка нового слоя хранения и репозитория сайтов.</p>

        <div style="margin:20px 0;">
            <button id="sb-ping-btn" style="padding:10px 16px;cursor:pointer;">
                Проверить ping
            </button>

            <button id="sb-site-list-btn" style="padding:10px 16px;cursor:pointer;margin-left:10px;">
                Показать реальные site.list
            </button>

            <button id="sb-demo-list-btn" style="padding:10px 16px;cursor:pointer;margin-left:10px;">
                Проверить demoList
            </button>
        </div>

        <pre id="sb-result" style="background:#fff;border:1px solid #d0d7de;padding:16px;white-space:pre-wrap;min-height:260px;">Нажми кнопку для проверки.</pre>
    </div>

    <script>
    (function () {
        var resultNode = document.getElementById('sb-result');
        var pingBtn = document.getElementById('sb-ping-btn');
        var siteListBtn = document.getElementById('sb-site-list-btn');
        var listBtn = document.getElementById('sb-demo-list-btn');

        function printResult(title, data) {
            resultNode.textContent = title + "\n\n" + JSON.stringify(data, null, 2);
        }

        function call(action) {
            printResult('Запрос...', { action: action });

            fetch('/local/tools/its.sitebuilder/ajax.php', {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
                },
                body: 'action=' + encodeURIComponent(action)
            })
            .then(function (response) {
                return response.text().then(function (text) {
                    var parsed = null;

                    try {
                        parsed = JSON.parse(text);
                    } catch (e) {
                        parsed = {
                            ok: false,
                            error: 'INVALID_JSON',
                            raw: text
                        };
                    }

                    printResult('Ответ сервера', {
                        httpStatus: response.status,
                        body: parsed
                    });
                });
            })
            .catch(function (error) {
                printResult('Ошибка fetch', {
                    message: error && error.message ? error.message : String(error)
                });
            });
        }

        pingBtn.addEventListener('click', function () {
            call('site.ping');
        });

        siteListBtn.addEventListener('click', function () {
            call('site.list');
        });

        listBtn.addEventListener('click', function () {
            call('site.demoList');
        });

        printResult('Страница загружена', {
            js: 'ok',
            endpoint: '/local/tools/its.sitebuilder/ajax.php',
            sitesFile: '/upload/sitebuilder/sites.json'
        });
    })();
    </script>
</body>
</html>


---

Что проверять

Открой:

/local/sitebuilder/index.php

Потом нажми:

Проверить ping

Показать реальные site.list



---

Какой результат ожидаем

Если старый JSON уже существует в:

/upload/sitebuilder/sites.json

то site.list должен вернуть реальные записи.

Примерно так:

{
  "ok": true,
  "data": {
    "items": [
      {
        "id": 1,
        "name": "Тестовый сайт",
        "slug": "test-site"
      }
    ],
    "count": 1
  }
}


---

Что мы только что сделали правильно

Мы уже убрали из будущего монолита три вещи:

путь к данным;

прямое чтение JSON;

работу со списком сайтов.


То есть теперь логика идёт по цепочке:

ajax.php → SiteService → SiteRepository → JsonStorage

Это уже нормальный фундамент.


---

Что дальше

Следующим шагом я дам уже готовый код для реального CRUD по сайтам:

site.get

site.create

site.delete


с генерацией ID, slug и записью в sites.json.

После этого начнём переносить:

access

pages

blocks


по тому же шаблону.