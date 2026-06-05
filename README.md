500 — Ошибка приложения
Сообщение:

Local\MvcDemo\Controllers\NoteController::destroy(): Argument #1 ($note) must be of type Local\MvcDemo\Models\Note, string given
Файл:

/srv/bx/docroot/local/mvc_demo/Controllers/NoteController.php:76
Trace:

#0 [internal function]: Local\MvcDemo\Controllers\NoteController->destroy()
#1 /srv/bx/docroot/local/mvc/Core/Container.php(109): ReflectionMethod->invokeArgs()
#2 /srv/bx/docroot/local/mvc/Core/Router.php(372): Local\Mvc\Core\Container->call()
#3 /srv/bx/docroot/local/mvc/Core/App.php(69): Local\Mvc\Core\Router->dispatch()
#4 /srv/bx/docroot/local/mvc_demo/index.php(17): Local\Mvc\Core\App::run()
#5 /srv/bx/docroot/bitrix/modules/main/include/urlrewrite.php(128): include_once('...')
#6 /srv/bx/docroot/bitrix/urlrewrite.php(2): include_once('...')
#7 {main}
