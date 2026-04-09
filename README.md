Хорошо, идём дальше.

Сейчас делаем первый реальный CRUD для сайтов в новом модуле:

site.get

site.create

site.delete


Пока удаление будет только записи сайта из sites.json, без каскадного удаления pages.json, blocks.json, menus.json, access.json. Это специально, чтобы сначала аккуратно поднять новый слой CRUD, а уже следующим шагом сделать нормальное каскадное удаление.


---

Что делаем сейчас

1. Создай файл

/local/modules/its.sitebuilder/lib/infrastructure/support/idgenerator.php

Сначала создай папку:

/local/modules/its.sitebuilder/lib/infrastructure/support/

Файл:

<?php

namespace Its\Sitebuilder\Infrastructure\Support;

class IdGenerator
{
    public function nextId(array $items, string $field = 'id'): int
    {
        $max = 0;

        foreach ($items as $item) {
            $value = 0;

            if (is_array($item)) {
                $value = (int)($item[$field] ?? 0);
            } elseif (is_object($item) && method_exists($item, 'toArray')) {
                $row = $item->toArray();
                $value = (int)($row[$field] ?? 0);
            }

            if ($value > $max) {
                $max = $value;
            }
        }

        return $max + 1;
    }
}


---

2. Создай файл

/local/modules/its.sitebuilder/lib/infrastructure/support/slugger.php

<?php

namespace Its\Sitebuilder\Infrastructure\Support;

class Slugger
{
    public function slugify(string $value): string
    {
        $value = trim($value);
        $value = mb_strtolower($value, 'UTF-8');

        $map = [
            'а' => 'a', 'б' => 'b', 'в' => 'v', 'г' => 'g', 'д' => 'd',
            'е' => 'e', 'ё' => 'e', 'ж' => 'zh', 'з' => 'z', 'и' => 'i',
            'й' => 'y', 'к' => 'k', 'л' => 'l', 'м' => 'm', 'н' => 'n',
            'о' => 'o', 'п' => 'p', 'р' => 'r', 'с' => 's', 'т' => 't',
            'у' => 'u', 'ф' => 'f', 'х' => 'h', 'ц' => 'cz', 'ч' => 'ch',
            'ш' => 'sh', 'щ' => 'shh', 'ъ' => '', 'ы' => 'y', 'ь' => '',
            'э' => 'e', 'ю' => 'yu', 'я' => 'ya',
        ];

        $value = strtr($value, $map);
        $value = preg_replace('/[^a-z0-9]+/u', '-', $value);
        $value = trim((string)$value, '-');

        if ($value === '') {
            $value = 'site';
        }

        return $value;
    }

    public function makeUnique(string $baseSlug, array $existingSlugs): string
    {
        $baseSlug = $this->slugify($baseSlug);

        if (!in_array($baseSlug, $existingSlugs, true)) {
            return $baseSlug;
        }

        $i = 2;
        while (in_array($baseSlug . '-' . $i, $existingSlugs, true)) {
            $i++;
        }

        return $baseSlug . '-' . $i;
    }
}


---

3. Полностью замени

/local/modules/its.sitebuilder/lib/infrastructure/repository/siterepository.php

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

    public function add(array $siteData): SiteEntity
    {
        $sites = $this->findAll();
        $sites[] = new SiteEntity($siteData);

        $this->saveAll($sites);

        return new SiteEntity($siteData);
    }

    public function deleteById(int $id): bool
    {
        $sites = $this->findAll();
        $before = count($sites);

        $filtered = [];

        foreach ($sites as $site) {
            if ($site->getId() !== $id) {
                $filtered[] = $site;
            }
        }

        if (count($filtered) === $before) {
            return false;
        }

        $this->saveAll($filtered);

        return true;
    }

    public function existsBySlug(string $slug): bool
    {
        foreach ($this->findAll() as $site) {
            if ($site->getSlug() === $slug) {
                return true;
            }
        }

        return false;
    }

    public function getAllSlugs(): array
    {
        $result = [];

        foreach ($this->findAll() as $site) {
            $slug = $site->getSlug();
            if ($slug !== '') {
                $result[] = $slug;
            }
        }

        return $result;
    }
}


---

4. Полностью замени

/local/modules/its.sitebuilder/lib/service/site/service.php

<?php

namespace Its\Sitebuilder\Service\Site;

use Its\Sitebuilder\Domain\Site\SiteEntity;
use Its\Sitebuilder\Infrastructure\Repository\SiteRepository;
use Its\Sitebuilder\Infrastructure\Support\IdGenerator;
use Its\Sitebuilder\Infrastructure\Support\Slugger;

class Service
{
    private SiteRepository $siteRepository;
    private IdGenerator $idGenerator;
    private Slugger $slugger;

    public function __construct(
        ?SiteRepository $siteRepository = null,
        ?IdGenerator $idGenerator = null,
        ?Slugger $slugger = null
    ) {
        $this->siteRepository = $siteRepository ?: new SiteRepository();
        $this->idGenerator = $idGenerator ?: new IdGenerator();
        $this->slugger = $slugger ?: new Slugger();
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

    public function get(int $id): array
    {
        $site = $this->siteRepository->findById($id);

        if (!$site) {
            throw new \RuntimeException('Сайт не найден: #' . $id);
        }

        return [
            'item' => $site->toArray(),
        ];
    }

    public function create(array $data): array
    {
        global $USER;

        $name = trim((string)($data['name'] ?? ''));
        $slug = trim((string)($data['slug'] ?? ''));

        if ($name === '') {
            throw new \InvalidArgumentException('Не заполнено имя сайта');
        }

        $existingSites = $this->siteRepository->findAll();
        $newId = $this->idGenerator->nextId($existingSites, 'id');
        $existingSlugs = $this->siteRepository->getAllSlugs();

        if ($slug === '') {
            $slug = $this->slugger->makeUnique($name, $existingSlugs);
        } else {
            $slug = $this->slugger->makeUnique($slug, $existingSlugs);
        }

        $now = date('c');
        $userId = is_object($USER) ? (int)$USER->GetID() : 0;

        $siteData = [
            'id' => $newId,
            'name' => $name,
            'slug' => $slug,
            'active' => true,
            'createdAt' => $now,
            'updatedAt' => $now,
            'createdBy' => $userId,
            'updatedBy' => $userId,
        ];

        $site = $this->siteRepository->add($siteData);

        return [
            'item' => $site->toArray(),
        ];
    }

    public function delete(int $id): array
    {
        $site = $this->siteRepository->findById($id);

        if (!$site) {
            throw new \RuntimeException('Сайт не найден: #' . $id);
        }

        $deleted = $this->siteRepository->deleteById($id);

        if (!$deleted) {
            throw new \RuntimeException('Не удалось удалить сайт: #' . $id);
        }

        return [
            'deleted' => true,
            'id' => $id,
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

5. Полностью замени

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

        case 'site.get':
            $result = $siteService->get((int)($_REQUEST['id'] ?? 0));
            break;

        case 'site.create':
            $result = $siteService->create([
                'name' => (string)($_REQUEST['name'] ?? ''),
                'slug' => (string)($_REQUEST['slug'] ?? ''),
            ]);
            break;

        case 'site.delete':
            $result = $siteService->delete((int)($_REQUEST['id'] ?? 0));
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

6. Полностью замени

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
    <div style="padding:20px;max-width:1200px;">
        <h1 style="margin-top:0;">ITS Site Builder</h1>

        <p>Проверка CRUD сайтов в новом модуле.</p>

        <div style="display:flex;gap:24px;align-items:flex-start;flex-wrap:wrap;">
            <div style="flex:1 1 360px;background:#fff;border:1px solid #d0d7de;padding:16px;">
                <h2 style="margin-top:0;font-size:20px;">Создать сайт</h2>

                <div style="margin-bottom:12px;">
                    <label for="sb-site-name" style="display:block;margin-bottom:6px;">Название</label>
                    <input id="sb-site-name" type="text" value="" placeholder="Например: Тестовый сайт"
                           style="width:100%;padding:10px;box-sizing:border-box;">
                </div>

                <div style="margin-bottom:12px;">
                    <label for="sb-site-slug" style="display:block;margin-bottom:6px;">Slug (необязательно)</label>
                    <input id="sb-site-slug" type="text" value="" placeholder="Например: test-site"
                           style="width:100%;padding:10px;box-sizing:border-box;">
                </div>

                <div style="margin-top:16px;">
                    <button id="sb-create-site-btn" style="padding:10px 16px;cursor:pointer;">
                        Создать сайт
                    </button>
                </div>
            </div>

            <div style="flex:1 1 360px;background:#fff;border:1px solid #d0d7de;padding:16px;">
                <h2 style="margin-top:0;font-size:20px;">Операции</h2>

                <div style="display:flex;gap:10px;flex-wrap:wrap;">
                    <button id="sb-ping-btn" style="padding:10px 16px;cursor:pointer;">Ping</button>
                    <button id="sb-site-list-btn" style="padding:10px 16px;cursor:pointer;">site.list</button>
                    <button id="sb-site-get-btn" style="padding:10px 16px;cursor:pointer;">site.get</button>
                    <button id="sb-site-delete-btn" style="padding:10px 16px;cursor:pointer;">site.delete</button>
                </div>

                <div style="margin-top:12px;">
                    <label for="sb-site-id" style="display:block;margin-bottom:6px;">ID сайта</label>
                    <input id="sb-site-id" type="number" value="" placeholder="Например: 1"
                           style="width:160px;padding:10px;box-sizing:border-box;">
                </div>
            </div>
        </div>

        <div style="margin-top:20px;">
            <pre id="sb-result" style="background:#fff;border:1px solid #d0d7de;padding:16px;white-space:pre-wrap;min-height:320px;">Нажми кнопку для проверки.</pre>
        </div>
    </div>

    <script>
    (function () {
        var resultNode = document.getElementById('sb-result');

        var pingBtn = document.getElementById('sb-ping-btn');
        var siteListBtn = document.getElementById('sb-site-list-btn');
        var siteGetBtn = document.getElementById('sb-site-get-btn');
        var siteDeleteBtn = document.getElementById('sb-site-delete-btn');
        var createSiteBtn = document.getElementById('sb-create-site-btn');

        var siteIdInput = document.getElementById('sb-site-id');
        var siteNameInput = document.getElementById('sb-site-name');
        var siteSlugInput = document.getElementById('sb-site-slug');

        function printResult(title, data) {
            resultNode.textContent = title + "\n\n" + JSON.stringify(data, null, 2);
        }

        function toFormBody(params) {
            var parts = [];

            Object.keys(params).forEach(function (key) {
                parts.push(encodeURIComponent(key) + '=' + encodeURIComponent(params[key]));
            });

            return parts.join('&');
        }

        function call(action, params) {
            var payload = Object.assign({ action: action }, params || {});

            printResult('Запрос...', payload);

            fetch('/local/tools/its.sitebuilder/ajax.php', {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
                },
                body: toFormBody(payload)
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

        siteGetBtn.addEventListener('click', function () {
            call('site.get', {
                id: siteIdInput.value || ''
            });
        });

        siteDeleteBtn.addEventListener('click', function () {
            call('site.delete', {
                id: siteIdInput.value || ''
            });
        });

        createSiteBtn.addEventListener('click', function () {
            call('site.create', {
                name: siteNameInput.value || '',
                slug: siteSlugInput.value || ''
            });
        });

        printResult('Страница загружена', {
            js: 'ok',
            endpoint: '/local/tools/its.sitebuilder/ajax.php',
            sitesFile: '/upload/sitebuilder/sites.json',
            note: 'Удаление пока только из sites.json, без каскада'
        });
    })();
    </script>
</body>
</html>


---

Что проверить сейчас

Открой:

/local/sitebuilder/index.php

И проверь по порядку:

1. site.list


2. создать новый сайт


3. снова site.list


4. site.get по ID


5. site.delete


6. снова site.list




---

Что важно на этом этапе

Сейчас мы уже получили в новом модуле:

Config

JsonStorage

SiteRepository

IdGenerator

Slugger

SiteService

реальный CRUD для sites.json


То есть кусок старого монолита уже реально вынесен в нормальные слои.


---

Что дальше

Следующим шагом правильно сделать одно из двух:

вариант А — лучший сейчас:
сразу добавить каскадное удаление сайта из:

pages.json

blocks.json

menus.json

access.json


вариант Б:
перенести page.list / page.create / page.delete.

Я рекомендую сначала вариант А, потому что без него новый site.delete потенциально оставляет мусор в данных.