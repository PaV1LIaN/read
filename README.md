

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/dashboard

500 — Ошибка приложения
Сообщение:

Class "Local\MvcDemo\Models\User" not found
Файл:

/srv/bx/docroot/local/mvc_demo/Controllers/AdminController.php:23
Trace:

#0 /srv/bx/docroot/local/mvc/Core/Router.php(135): Local\MvcDemo\Controllers\AdminController->dashboard()
#1 /srv/bx/docroot/local/mvc/Core/App.php(60): Local\Mvc\Core\Router->dispatch()
#2 /srv/bx/docroot/local/mvc_demo/index.php(17): Local\Mvc\Core\App::run()
#3 /srv/bx/docroot/bitrix/modules/main/include/urlrewrite.php(128): include_once('...')
#4 /srv/bx/docroot/bitrix/urlrewrite.php(2): include_once('...')
#5 {main}

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users

500 — Ошибка приложения
Сообщение:

Class "Local\MvcDemo\Models\User" not found
Файл:

/srv/bx/docroot/local/mvc_demo/Controllers/AdminController.php:29
Trace:

#0 /srv/bx/docroot/local/mvc/Core/Router.php(135): Local\MvcDemo\Controllers\AdminController->users()
#1 /srv/bx/docroot/local/mvc/Core/App.php(60): Local\Mvc\Core\Router->dispatch()
#2 /srv/bx/docroot/local/mvc_demo/index.php(17): Local\Mvc\Core\App::run()
#3 /srv/bx/docroot/bitrix/modules/main/include/urlrewrite.php(128): include_once('...')
#4 /srv/bx/docroot/bitrix/urlrewrite.php(2): include_once('...')
#5 {main}
$blocks = Block::byPage($pageId);

Следующим шагом сделаем Repository/Service слой, чтобы понять разницу: где должна быть бизнес-логика, а где просто работа с таблицей.
