Тогда это уже почти наверняка не роутер, а перепутанные файлы в api/handlers.

Потому что:

opcache_reset() сработал

ping всё равно отвечает из access

значит PHP читает текущие файлы, но в них лежит не то содержимое


Сейчас делаем самый жёсткий и короткий тест.

1. Полностью замени /local/sitebuilder/api/handlers/common.php на это

<?php
header('Content-Type: application/json; charset=UTF-8');
echo json_encode([
    'ok' => true,
    'hardcoded' => 'COMMON_HANDLER_WORKS',
    'file' => __FILE__,
    'action' => $_POST['action'] ?? null,
], JSON_UNESCAPED_UNICODE);
exit;

2. Полностью замени /local/sitebuilder/api/index.php на это

<?php

require_once __DIR__ . '/bootstrap.php';

$action = (string)($_POST['action'] ?? '');

if ($action === 'ping') {
    require __DIR__ . '/handlers/common.php';
    exit;
}

header('Content-Type: application/json; charset=UTF-8');
echo json_encode([
    'ok' => false,
    'hardcoded' => 'INDEX_REACHED',
    'action' => $action,
    'file' => __FILE__,
], JSON_UNESCAPED_UNICODE);
exit;

3. Оставь /local/sitebuilder/api.php таким

<?php
ini_set('display_errors', '1');
error_reporting(E_ALL);

require_once __DIR__ . '/api/index.php';

4. Снова нажми Ping

Нормальный результат сейчас должен быть строго такой:

{
  "ok": true,
  "hardcoded": "COMMON_HANDLER_WORKS",
  "file": "/srv/bx/docroot/local/sitebuilder/api/handlers/common.php",
  "action": "ping"
}


---

Если снова будет handler":"access"

Тогда возможны только две вещи:

Вариант А. Ты редактируешь не тот файл

Проверь прямо в консоли сервера:

cat /srv/bx/docroot/local/sitebuilder/api/handlers/common.php
cat /srv/bx/docroot/local/sitebuilder/api/index.php
cat /srv/bx/docroot/local/sitebuilder/api.php

И убедись, что там именно тот код, который ты вставил.

Вариант Б. common.php — это ссылка или копия не туда

Проверь:

ls -l /srv/bx/docroot/local/sitebuilder/api/handlers/

Нужно посмотреть, не является ли common.php ссылкой на другой файл.


---

Самая полезная команда сейчас

Запусти на сервере:

grep -R "NOT_MOVED_YET" -n /srv/bx/docroot/local/sitebuilder

И пришли вывод.

Особенно важно увидеть:

где именно лежит строка с "handler" => "access"

нет ли её в common.php

нет ли нескольких копий проекта



---

Почему я уверен, что это уже файловая путаница

После такого common.php:

echo json_encode(['hardcoded' => 'COMMON_HANDLER_WORKS']);
exit;

никакой нормальный роутер уже не сможет вернуть handler":"access", если только реально не подключается другой файл.


---

Что пришли следующим сообщением

Либо:

1. ответ кнопки Ping после этих двух жёстких замен



Либо, если есть доступ к серверу: 2. вывод команд

cat /srv/bx/docroot/local/sitebuilder/api/handlers/common.php
cat /srv/bx/docroot/local/sitebuilder/api/index.php
ls -l /srv/bx/docroot/local/sitebuilder/api/handlers/
grep -R "NOT_MOVED_YET" -n /srv/bx/docroot/local/sitebuilder

После этого я уже скажу точно, где у тебя подмена.