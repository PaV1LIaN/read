Обработка заняла 2m 13s

Отлично — значит сама SQL-миграция этапа 5 рабочая. Ошибка возникает именно при запуске через PHP: либо при соединении PHP с базой, либо при регистрации миграции в schema_migration. Обычная страница скрывает настоящую причину за MIGRATION_FAILED.

Я подготовил временную диагностическую страницу:

Скачать migration_debug.php

Что сделать

1. Загрузи файл сюда:



/local/sitebuilder/tools/migration_debug.php

2. Открой под администратором Битрикс:



https://portal24.itsnn.ru/local/sitebuilder/tools/migration_debug.php

3. В верхнем блоке проверь подключение PHP:



is_replica = false
transaction_read_only = off

4. Нажми:



Запустить миграцию и показать точную ошибку

Страница покажет полную цепочку:

Класс
Message
Code
SQLSTATE
Driver message
Details

Пришли скриншот этого блока или скопируй значения Message, SQLSTATE и Driver message.

После диагностики файл обязательно удали:

/local/sitebuilder/tools/migration_debug.php

Сейчас обычную страницу apply_stage13_migration.php повторно не запускай. Базу восстанавливать не требуется.