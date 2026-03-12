Проверять руками через код пока неудобно. Проще так:

Что уже должно быть сделано

У тебя сейчас должно быть:

в api.php добавлены layout.* action'ы,

создан файл
/upload/sitebuilder/layouts.json

в нём лежит:


[]


---

Самая простая проверка

Открыть браузер, зайти в проект, открыть DevTools → Network и отправить запросы через уже существующий JS или временную тестовую страницу.

Но ещё проще — сделать временный тестовый php-файл.


---

Вариант проверки через тестовый файл

Создай файл:

/local/sitebuilder/layout_test.php

И вставь туда это:

<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

global $USER, $APPLICATION;

if (!$USER->IsAuthorized()) {
    LocalRedirect('/auth/');
}

$siteId = (int)($_GET['siteId'] ?? 0);
?>
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8">
  <title>Layout test</title>
  <?php $APPLICATION->ShowHead(); ?>
  <script src="/bitrix/js/main/core/core.js"></script>
</head>
<body>
  <h2>Layout test</h2>
  <div>siteId: <?= (int)$siteId ?></div>

  <button onclick="testGet()">layout.get</button>
  <button onclick="testCreate()">layout.block.create(header text)</button>
  <button onclick="testList()">layout.block.list(header)</button>

  <pre id="out" style="margin-top:20px;background:#f5f5f5;padding:12px;border:1px solid #ddd;"></pre>

<script>
function out(x){
  document.getElementById('out').textContent =
    typeof x === 'string' ? x : JSON.stringify(x, null, 2);
}

function api(action, data) {
  return new Promise((resolve, reject) => {
    BX.ajax({
      url: '/local/sitebuilder/api.php',
      method: 'POST',
      dataType: 'json',
      data: Object.assign({ action, sessid: BX.bitrix_sessid() }, data || {}),
      onsuccess: resolve,
      onfailure: reject
    });
  });
}

async function testGet() {
  try {
    const r = await api('layout.get', { siteId: <?= (int)$siteId ?> });
    out(r);
  } catch (e) {
    out('layout.get error');
  }
}

async function testCreate() {
  try {
    const r = await api('layout.block.create', {
      siteId: <?= (int)$siteId ?>,
      zone: 'header',
      type: 'text',
      text: 'Тестовый header блок'
    });
    out(r);
  } catch (e) {
    out('layout.block.create error');
  }
}

async function testList() {
  try {
    const r = await api('layout.block.list', {
      siteId: <?= (int)$siteId ?>,
      zone: 'header'
    });
    out(r);
  } catch (e) {
    out('layout.block.list error');
  }
}
</script>
</body>
</html>


---

Как тестировать

Открой:

/local/sitebuilder/layout_test.php?siteId=1

Где 1 — это id существующего сайта.

Потом по порядку:

1. Нажми layout.get

Должно вернуть примерно:

{
  "ok": true,
  "layout": {
    "showHeader": true,
    "showFooter": true,
    "showLeft": false,
    "showRight": false,
    "leftWidth": 260,
    "rightWidth": 260
  },
  "zones": {
    "header": [],
    "footer": [],
    "left": [],
    "right": []
  }
}


---

2. Нажми layout.block.create(header text)

Должно вернуть что-то вроде:

{
  "ok": true,
  "block": {
    "id": 1,
    "type": "text",
    "sort": 10,
    "content": {
      "text": "Тестовый header блок"
    }
  }
}


---

3. Нажми layout.block.list(header)

Должен вернуться массив с этим блоком.


---

Если что-то не работает

Тогда смотри, что именно вернулось в pre.

Например:

SITE_ID_REQUIRED

FORBIDDEN

UNKNOWN_ACTION

Cannot open layouts.json


И уже по этой ошибке можно точно понять, где проблема.


---

Самые частые причины

UNKNOWN_ACTION

Значит блок layout.* вставлен не туда или не сохранился в api.php.

Cannot open layouts.json

Значит файл /upload/sitebuilder/layouts.json не создан или нет прав.

FORBIDDEN

Значит у пользователя нет прав на этот siteId.


---

Если хочешь, следующим сообщением я дам уже готовый layout.php, и тогда ты будешь тестировать не кнопками-заглушками, а сразу нормальный интерфейс.