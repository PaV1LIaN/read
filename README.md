Да, можно делать именно так:

/local/project1
/local/project2
/local/project3

И это даже нормальный вариант.

Тогда схема будет такая:

/local/mvc/          — общий фреймворк, инструмент
/local/project1/     — первый проект
/local/project2/     — второй проект
/local/project3/     — третий проект

То есть фреймворк отдельно, проекты отдельно.

Как это выглядело бы

Например:

/local/
  mvc/
    bootstrap.php
    Core/
      Router.php
      Request.php
      Response.php
      Controller.php
      Db.php
      Model.php
      Auth.php
      View.php

  project1/
    index.php
    routes.php
    Controllers/
      HomeController.php
      PageController.php
    Models/
      Page.php
    Views/
      layouts/
        app.php
      home/
        index.php
      pages/
        list.php
    assets/
      app.css
      app.js

  project2/
    index.php
    routes.php
    Controllers/
      DashboardController.php
      ReportController.php
    Models/
      Report.php
    Views/
      layouts/
        app.php
      dashboard/
        index.php
    assets/
      app.css
      app.js

Простыми словами

/local/mvc — это набор инструментов.

Он умеет:

принимать запрос
разбирать адрес
вызывать контроллер
рендерить view
возвращать JSON
работать с базой
проверять авторизацию

А /local/project1 — это уже конкретный сайт/раздел/приложение.

Например:

/local/sitebuilder
/local/glab
/local/qr_opros
/local/admin_dashboard

Каждый проект может использовать общий MVC.


---

Как будет заходить пользователь

Например, для первого проекта:

https://bitrix24-stage.gaz.ru/local/project1/
https://bitrix24-stage.gaz.ru/local/project1/pages
https://bitrix24-stage.gaz.ru/local/project1/api/save

Для второго проекта:

https://bitrix24-stage.gaz.ru/local/project2/
https://bitrix24-stage.gaz.ru/local/project2/reports
https://bitrix24-stage.gaz.ru/local/project2/export

Каждый проект имеет свой index.php.


---

Пример /local/project1/index.php

Он был бы маленький:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

use Local\Mvc\Core\Request;
use Local\Mvc\Core\Router;

$request = Request::createFromGlobals();

$router = new Router();

require_once __DIR__ . '/routes.php';

$router->dispatch($request);

То есть проект говорит:

Я подключаю общий MVC.
Создаю Request.
Создаю Router.
Подключаю свои маршруты.
Запускаю.


---

Пример /local/project1/routes.php

<?php

use Local\Mvc\Core\Router;
use Local\Project1\Controllers\HomeController;
use Local\Project1\Controllers\PageController;

/** @var Router $router */

$router->get('/', [HomeController::class, 'index']);

$router->get('/pages', [PageController::class, 'index']);

$router->get('/pages/create', [PageController::class, 'create']);


---

Пример контроллера проекта

Файл:

/local/project1/Controllers/HomeController.php

<?php

namespace Local\Project1\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class HomeController extends Controller
{
    public function index(): Response
    {
        return $this->render('home/index', [
            'title' => 'Проект 1',
            'message' => 'Это первый проект на общем MVC.',
        ]);
    }
}

Смотри, что важно:

use Local\Mvc\Core\Controller;

То есть контроллер проекта наследуется от общего контроллера фреймворка.


---

Как автозагрузка должна понимать проекты

В bootstrap.php фреймворка надо будет сказать:

Local\Mvc\       → /local/mvc/
Local\Project1\  → /local/project1/
Local\Project2\  → /local/project2/

Примерно так:

spl_autoload_register(function ($class) {
    $map = [
        'Local\\Mvc\\' => $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/',
        'Local\\Project1\\' => $_SERVER['DOCUMENT_ROOT'] . '/local/project1/',
        'Local\\Project2\\' => $_SERVER['DOCUMENT_ROOT'] . '/local/project2/',
    ];

    foreach ($map as $prefix => $baseDir) {
        if (strncmp($prefix, $class, strlen($prefix)) !== 0) {
            continue;
        }

        $relativeClass = substr($class, strlen($prefix));
        $file = rtrim($baseDir, '/') . '/' . str_replace('\\', '/', $relativeClass) . '.php';

        if (is_file($file)) {
            require_once $file;
        }

        return;
    }
});


---

Тогда Sitebuilder выглядел бы так

Например:

/local/
  mvc/
    Core/
      Controller.php
      Router.php
      Request.php
      Response.php
      Db.php
      Model.php

  sitebuilder/
    index.php
    routes.php
    Controllers/
      SiteController.php
      PageController.php
      BlockController.php
      DiskController.php
    Models/
      Site.php
      Page.php
      Block.php
      Access.php
    Views/
      layouts/
        admin.php
        editor.php
      sites/
        index.php
        edit.php
      pages/
        edit.php
      blocks/
        list.php
    assets/
      css/
      js/

Адреса были бы такие:

/local/sitebuilder/
/local/sitebuilder/sites
/local/sitebuilder/pages/edit?id=5
/local/sitebuilder/api/site/create


---

А GLAB так

/local/
  glab/
    index.php
    routes.php
    Controllers/
      ApplicationController.php
      AdminController.php
      ReportController.php
    Models/
      Application.php
      Enterprise.php
      Laboratory.php
    Views/
      applications/
        index.php
        form.php
      admin/
        index.php
      reports/
        dashboard.php
    assets/
      app.css
      app.js

Адреса:

/local/glab/applications
/local/glab/admin
/local/glab/reports


---

Что лучше: /local/apps/project1 или /local/project1

Оба варианта нормальные.

Вариант 1

/local/apps/sitebuilder
/local/apps/glab
/local/apps/qr

Плюс: всё аккуратно лежит в одной папке apps.

Минус: адреса могут быть длиннее, если не делать отдельный вход.

Вариант 2

/local/sitebuilder
/local/glab
/local/qr_opros

Плюс: ближе к тому, как у тебя уже сейчас устроены проекты.

Минус: в /local будет больше папок, но это не страшно.

Для тебя я бы выбрал второй вариант, потому что у тебя уже есть:

/local/sitebuilder
/local/glab
/local/qr_opros

Значит лучше не ломать привычную структуру.


---

Как я бы делал у тебя

Я бы оставил так:

/local/mvc/             — общий фреймворк

/local/sitebuilder/     — проект конструктора
/local/glab/            — лаборатории
/local/qr_opros/        — QR-опросы

И каждый проект постепенно переводил бы на общий MVC.

То есть мы не переписываем всё сразу.

Сначала делаем:

/local/mvc

Потом берём один маленький новый раздел, например:

/local/sitebuilder/admin-dashboard

или отдельный тестовый:

/local/demo_mvc

и подключаем к нему общий фреймворк.


---

Главная мысль

Да, если делать так:

/local/проект1
/local/проект2

то фреймворк всё равно остаётся общим инструментом:

/local/mvc

А проекты просто используют его.

Правильная схема:

Фреймворк:
  /local/mvc/Core

Проект:
  /local/project1/Controllers
  /local/project1/Models
  /local/project1/Views
  /local/project1/routes.php
  /local/project1/index.php

Это хороший вариант для твоего Bitrix24.