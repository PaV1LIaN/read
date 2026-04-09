Да, идём по плану правильно: сначала создаём параллельный каркас нового модуля, а старый api.php и текущие страницы пока не трогаем. Это безопаснее, потому что Bitrix ожидает модуль в /local/modules/<vendor>.<module>/ с install/index.php, include.php, .settings.php, а контроллеры внутри модуля должны наследоваться от \Bitrix\Main\Engine\Controller; для JS Bitrix рекомендует расширения в /local/js/<module>/<extension>/ со структурой src, dist, bundle.config.js, config.php. 

Я предлагаю взять имя модуля its.sitebuilder.
Namespace тогда будет: Its\Sitebuilder.

Что делаем сейчас

Создаём вот такую минимальную структуру:

/local/modules/its.sitebuilder/
  include.php
  .settings.php
  install/
    index.php
    version.php
  lib/
    controller/
      site.php

/local/sitebuilder/
  index.php

На этом шаге у нас будет:

модуль, который видно Bitrix;

D7-контроллер its:sitebuilder.Site.ping;

тестовая страница /local/sitebuilder/index.php, которая вызывает контроллер через BX.ajax.runAction, как и рекомендует Bitrix. 



---

1. /local/modules/its.sitebuilder/install/version.php

<?php

$arModuleVersion = [
    'VERSION' => '0.1.0',
    'VERSION_DATE' => '2026-04-09 12:00:00',
];


---

2. /local/modules/its.sitebuilder/install/index.php

<?php

use Bitrix\Main\Localization\Loc;
use Bitrix\Main\ModuleManager;

Loc::loadMessages(__FILE__);

if (class_exists('its_sitebuilder')) {
    return;
}

class its_sitebuilder extends CModule
{
    public $MODULE_ID = 'its.sitebuilder';
    public $MODULE_VERSION;
    public $MODULE_VERSION_DATE;
    public $MODULE_NAME;
    public $MODULE_DESCRIPTION;
    public $PARTNER_NAME = 'ITS';
    public $PARTNER_URI = '';

    public function __construct()
    {
        $versionFile = __DIR__ . '/version.php';

        if (file_exists($versionFile)) {
            include $versionFile;
            if (is_array($arModuleVersion)) {
                $this->MODULE_VERSION = $arModuleVersion['VERSION'];
                $this->MODULE_VERSION_DATE = $arModuleVersion['VERSION_DATE'];
            }
        }

        $this->MODULE_NAME = 'ITS Site Builder';
        $this->MODULE_DESCRIPTION = 'Конструктор сайтов для Bitrix24';
    }

    public function DoInstall()
    {
        ModuleManager::registerModule($this->MODULE_ID);
    }

    public function DoUninstall()
    {
        ModuleManager::unRegisterModule($this->MODULE_ID);
    }
}


---

3. /local/modules/its.sitebuilder/include.php

<?php

use Bitrix\Main\Loader;

/**
 * Файл include.php должен существовать в модуле.
 * Bitrix подключает его при Loader::includeModule('its.sitebuilder').
 *
 * Пока дополнительная инициализация не нужна.
 */

if (!defined('B_PROLOG_INCLUDED') && !defined('B_PROLOG_INCLUDED')) {
    // Ничего не делаем специально.
}


---

4. /local/modules/its.sitebuilder/.settings.php

<?php

return [
    'controllers' => [
        'value' => [
            'defaultNamespace' => '\\Its\\Sitebuilder\\Controller',
        ],
        'readonly' => true,
    ],
];


---

5. /local/modules/its.sitebuilder/lib/controller/site.php

<?php

namespace Its\Sitebuilder\Controller;

use Bitrix\Main\Engine\Controller;
use Bitrix\Main\Error;

class Site extends Controller
{
    /**
     * Простое тестовое действие.
     * Проверяем, что модуль и D7-контроллер работают.
     *
     * Вызов:
     * BX.ajax.runAction('its:sitebuilder.Site.ping')
     */
    public function pingAction(): array
    {
        global $USER;

        return [
            'ok' => true,
            'module' => 'its.sitebuilder',
            'controller' => static::class,
            'userId' => is_object($USER) ? (int)$USER->GetID() : 0,
            'time' => date('c'),
        ];
    }

    /**
     * Пример второго действия.
     * Позже сюда переедет site.list из старого api.php.
     */
    public function demoListAction(): array
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

    /**
     * Настройка prefilters.
     * Пока специально не ужесточаем, чтобы сначала проверить работу контроллера.
     * На следующем шаге добавим auth + csrf-логику под реальные действия.
     */
    public function configureActions(): array
    {
        return [
            'ping' => [
                'prefilters' => [],
            ],
            'demoList' => [
                'prefilters' => [],
            ],
        ];
    }
}


---

6. /local/sitebuilder/index.php

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/header.php';

use Bitrix\Main\Loader;
use Bitrix\Main\UI\Extension;

global $APPLICATION, $USER;

$APPLICATION->SetTitle('Site Builder');

if (!Loader::includeModule('its.sitebuilder')) {
    echo '<div style="padding:20px;color:#b00020;font-weight:bold;">Модуль its.sitebuilder не установлен или не подключается.</div>';
    require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/footer.php';
    return;
}

Extension::load('main.core');
?>

<div style="padding:20px;font-family:Arial,sans-serif;">
    <h1>ITS Site Builder</h1>

    <p>
        Это тестовая страница нового модуля.
        Старую реализацию пока не трогаем.
    </p>

    <div style="margin:20px 0;">
        <button id="sb-ping-btn" style="padding:10px 16px;cursor:pointer;">
            Проверить контроллер ping
        </button>

        <button id="sb-demo-list-btn" style="padding:10px 16px;cursor:pointer;margin-left:10px;">
            Проверить demoList
        </button>
    </div>

    <pre id="sb-result" style="background:#f6f8fa;border:1px solid #d0d7de;padding:16px;white-space:pre-wrap;">
Нажми кнопку для проверки.
    </pre>
</div>

<script>
BX.ready(function () {
    var resultNode = document.getElementById('sb-result');
    var pingBtn = document.getElementById('sb-ping-btn');
    var listBtn = document.getElementById('sb-demo-list-btn');

    function printResult(title, data) {
        resultNode.textContent = title + "\n\n" + JSON.stringify(data, null, 2);
    }

    pingBtn.addEventListener('click', function () {
        printResult('Запрос...', { action: 'its:sitebuilder.Site.ping' });

        BX.ajax.runAction('its:sitebuilder.Site.ping')
            .then(function (response) {
                printResult('Успех', response);
            })
            .catch(function (error) {
                printResult('Ошибка', error);
            });
    });

    listBtn.addEventListener('click', function () {
        printResult('Запрос...', { action: 'its:sitebuilder.Site.demoList' });

        BX.ajax.runAction('its:sitebuilder.Site.demoList')
            .then(function (response) {
                printResult('Успех', response);
            })
            .catch(function (error) {
                printResult('Ошибка', error);
            });
    });
});
</script>

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/footer.php';


---

Что сделать после вставки файлов

1. Создай папки и файлы ровно по путям выше.


2. Зайди в Marketplace → Установленные решения и установи модуль its.sitebuilder, если он появится в списке.


3. Открой страницу:



/local/sitebuilder/index.php

4. Нажми:



Проверить контроллер ping

Проверить demoList


Если всё ок, увидишь JSON-ответ от нового D7-контроллера.

Что должно получиться

Если модуль подключился правильно, ping вернёт примерно такое:

{
  "status": "success",
  "data": {
    "ok": true,
    "module": "its.sitebuilder",
    "controller": "Its\\Sitebuilder\\Controller\\Site",
    "userId": 1,
    "time": "2026-04-09T..."
  }
}


---

Важный момент

На этом шаге мы ещё не переносим:

старый api.php

JSON-хранилище

Disk-логику

роли

editor/menu/files/settings


Сейчас цель только одна:
поднять новый модульный каркас и убедиться, что D7-контроллеры работают.

После этого следующим сообщением я дам тебе второй шаг: готовый bootstrap для общего доступа, AccessService, JsonStorage и первый реальный перенос site.list из старого api.php в новый модуль.