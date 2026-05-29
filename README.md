500 — Ошибка приложения
Сообщение:

Pgsql query error: ERROR:  42P01: relation "mvc.mvc_demo_notes" does not exist
LINE 1: SELECT * FROM mvc.mvc_demo_notes ORDER BY id DESC LIMIT 20
                      ^
LOCATION:  parserOpenTable, parse_relation.c:1428
Файл:

/srv/bx/docroot/bitrix/modules/main/lib/db/pgsqlconnection.php:662
Trace:

#0 /srv/bx/docroot/bitrix/modules/main/lib/db/pgsqlconnection.php(131): Bitrix\Main\DB\PgsqlConnection->createQueryException()
#1 /srv/bx/docroot/bitrix/modules/main/lib/db/connection.php(324): Bitrix\Main\DB\PgsqlConnection->queryInternal()
#2 /srv/bx/docroot/local/mvc/Core/Db.php(44): Bitrix\Main\DB\Connection->query()
#3 /srv/bx/docroot/local/mvc/Core/QueryBuilder.php(207): Local\Mvc\Core\Db::fetchAll()
#4 /srv/bx/docroot/local/mvc_demo/Models/Note.php(25): Local\Mvc\Core\QueryBuilder->get()
#5 /srv/bx/docroot/local/mvc_demo/Controllers/NoteController.php(16): Local\MvcDemo\Models\Note::latest()
#6 [internal function]: Local\MvcDemo\Controllers\NoteController->index()
#7 /srv/bx/docroot/local/mvc/Core/Container.php(109): ReflectionMethod->invokeArgs()
#8 /srv/bx/docroot/local/mvc/Core/Router.php(259): Local\Mvc\Core\Container->call()
#9 /srv/bx/docroot/local/mvc/Core/App.php(66): Local\Mvc\Core\Router->dispatch()
#10 /srv/bx/docroot/local/mvc_demo/index.php(17): Local\Mvc\Core\App::run()
#11 /srv/bx/docroot/bitrix/modules/main/include/urlrewrite.php(128): include_once('...')
#12 /srv/bx/docroot/bitrix/urlrewrite.php(2): include_once('...')
#13 {main}
