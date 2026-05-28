https://bitrix24-stage.gaz.ru/local/mvc_demo/debug/routes

500 — Ошибка приложения
Сообщение:

syntax error, unexpected token "<", expecting "function" or "const"
Файл:

/srv/bx/docroot/local/mvc/Core/Router.php:36
Trace:

#0 /srv/bx/docroot/local/mvc/Core/App.php(52): {closure}()
#1 /srv/bx/docroot/local/mvc_demo/index.php(17): Local\Mvc\Core\App::run()
#2 /srv/bx/docroot/bitrix/modules/main/include/urlrewrite.php(128): include_once('...')
#3 /srv/bx/docroot/bitrix/urlrewrite.php(2): include_once('...')
#4 {main}
