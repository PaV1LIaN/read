https://bitrix24-stage.gaz.ru/local/mvc_demo/notes

500 — Ошибка приложения
Сообщение:

DB_UNAVAILABLE
Файл:

/srv/bx/docroot/local/php_interface/lib/pg_master.php:83
Trace:

#0 /srv/bx/docroot/local/php_interface/lib/pg_master.php(91): findMasterDsn()
#1 /srv/bx/docroot/local/mvc/Core/Db.php(128): getPdo()
#2 /srv/bx/docroot/local/mvc/Core/Db.php(25): Local\Mvc\Core\Db::pdo()
#3 /srv/bx/docroot/local/mvc/Core/QueryBuilder.php(210): Local\Mvc\Core\Db::fetchAll()
#4 /srv/bx/docroot/local/mvc_demo/Models/Note.php(27): Local\Mvc\Core\QueryBuilder->get()
#5 /srv/bx/docroot/local/mvc_demo/Controllers/NoteController.php(17): Local\MvcDemo\Models\Note::latest()
#6 [internal function]: Local\MvcDemo\Controllers\NoteController->index()
#7 /srv/bx/docroot/local/mvc/Core/Container.php(109): ReflectionMethod->invokeArgs()
#8 /srv/bx/docroot/local/mvc/Core/Router.php(259): Local\Mvc\Core\Container->call()
#9 /srv/bx/docroot/local/mvc/Core/App.php(66): Local\Mvc\Core\Router->dispatch()
#10 /srv/bx/docroot/local/mvc_demo/index.php(17): Local\Mvc\Core\App::run()
#11 /srv/bx/docroot/bitrix/modules/main/include/urlrewrite.php(128): include_once('...')
#12 /srv/bx/docroot/bitrix/urlrewrite.php(2): include_once('...')
#13 {main}
