Ошибка понятная: Router нашёл контроллер:

Local\Mvc\Controllers\HomeController

но внутри него не нашёл метод:

index()

То есть файл HomeController.php есть, класс есть, но метода index() в нём нет или он называется иначе.

Полностью замени файл:

/local/mvc/Controllers/HomeController.php

на такой:

<?php

namespace Local\Mvc\Controllers;

use Local\Mvc\Core\Controller;

/**
 * HomeController
 *
 * Это первый тестовый контроллер.
 */
class HomeController extends Controller
{
    /**
     * Главная страница.
     *
     * Этот метод вызывается для маршрута:
     * /local/mvc/
     * или
     * /local/mvc/?route=/
     */
    public function index(): void
    {
        $this->render('home/index', [
            'title' => 'MVC работает',
            'message' => 'Наш первый MVC-каркас успешно подключился к Битриксу.',
        ]);
    }

    /**
     * Тестовый JSON-ответ.
     *
     * Этот метод вызывается для маршрута:
     * /local/mvc/?route=/ping
     */
    public function ping(): void
    {
        $this->success([
            'message' => 'pong',
            'mvc' => true,
        ]);
    }
}

Теперь проверь:

https://bitrix24-stage.gaz.ru/local/mvc/

и:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/ping

Если после замены всё равно будет та же ошибка, значит сервер может держать старую версию файла в OPcache. Тогда временно в начало /local/mvc/index.php после <?php добавь:

if (function_exists('opcache_reset')) {
    opcache_reset();
}

Потом обнови страницу. Когда заработает — эту строчку можно убрать.