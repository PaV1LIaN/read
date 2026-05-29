Тестовая форма
Это простая форма, чтобы проверить POST-запросы в нашем MVC.

500 — Ошибка приложения
Сообщение:

Call to undefined function mvc_e()
Файл:

/srv/bx/docroot/local/mvc_demo/Views/form/index.php:54
Trace:

#0 /srv/bx/docroot/local/mvc/Core/Controller.php(79): require()
#1 /srv/bx/docroot/local/mvc_demo/Controllers/FormController.php(15): Local\Mvc\Core\Controller->render()
#2 /srv/bx/docroot/local/mvc/Core/Router.php(215): Local\MvcDemo\Controllers\FormController->index()
#3 /srv/bx/docroot/local/mvc/Core/App.php(63): Local\Mvc\Core\Router->dispatch()
#4 /srv/bx/docroot/local/mvc_demo/index.php(17): Local\Mvc\Core\App::run()
#5 /srv/bx/docroot/bitrix/modules/main/include/urlrewrite.php(128): include_once('...')
#6 /srv/bx/docroot/bitrix/urlrewrite.php(2): include_once('...')
#7 {main}
