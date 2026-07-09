Да, ты правильно понял. Я в PageAccessRepository.php использовал:

use Bitrix\Main\Application;

$connection = Application::getConnection();

Из-за этого запрос пошёл через битриксовое подключение, поэтому ошибка и пришла из:

/bitrix/modules/main/lib/db/pgsqlconnection.php

А у тебя таблицы sitebuilder.* должны идти через:

/local/php_interface/lib/pg_master.php

То есть PageAccessRepository.php нужно переделать под pg_master.php.


---

Что делаем сейчас

Нужно заменить в:

/local/sitebuilder/lib/PageAccessRepository.php

все подключения через:

Application::getConnection()

на подключение через твой pg_master.php.

Но мне нужно видеть, как именно у тебя называется функция подключения внутри pg_master.php.

Обычно там может быть что-то вроде:

pg_master()

или:

pg_master_connect()

или:

getPgConnection()

или там создаётся $pdo.


---

Быстрая проверка

Открой файл:

/local/php_interface/lib/pg_master.php

и пришли сюда его содержимое.

Особенно важны строки, где есть:

function ...

или:

new PDO

или:

pg_connect


---

Почему так произошло

Я сделал репозиторий под стандартное подключение Битрикса:

Application::getConnection()

А это подключение смотрит не туда, где у тебя лежат таблицы sitebuilder.

Поэтому PostgreSQL и пишет:

отношение "sitebuilder.page_access" не существует

То есть таблица может существовать в твоей master-БД, но Битрикс ищет её в другой базе.


---

Дальше правильно будет так:

PageAccessRepository.php
↓
require_once /local/php_interface/lib/pg_master.php
↓
получаем PDO/PG connection
↓
работаем с sitebuilder.page_access

Пришли pg_master.php, и я сразу переделаю полный PageAccessRepository.php под твоё подключение.