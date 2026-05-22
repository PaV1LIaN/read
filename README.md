Отлично. Значит маршруты уже работают:

/local/mvc/about
/local/mvc/ping

Теперь делаем следующий важный кирпич — Layout.

Что такое Layout простыми словами

Сейчас у нас View — это сама страница.

Например:

Views/home/index.php

Но у каждой страницы обычно есть общие части:

верхнее меню
контейнер
фон
заголовок
подвал
подключение CSS

Чтобы не копировать это в каждую страницу, делают layout.

Представь так:

Layout — это рамка.
View — это содержимое внутри рамки.

Например:

Layout:
  ┌─────────────────────────────┐
  │ Меню                        │
  │ ┌─────────────────────────┐ │
  │ │ Здесь конкретная View   │ │
  │ └─────────────────────────┘ │
  └─────────────────────────────┘


---

Шаг 1. Создаём папку assets

Создай папку:

/local/mvc/assets/

В ней файл:

/local/mvc/assets/app.css

Код:

.mvc-app {
    max-width: 1180px;
    margin: 24px auto 60px;
    padding: 0 20px;
}

.mvc-topbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
    margin-bottom: 24px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e6e9ef;
    border-radius: 18px;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.06);
}

.mvc-brand {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.mvc-brand__title {
    font-size: 20px;
    font-weight: 700;
    color: #111827;
}

.mvc-brand__subtitle {
    font-size: 13px;
    color: #6b7280;
}

.mvc-nav {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
}

.mvc-nav a {
    display: inline-flex;
    align-items: center;
    min-height: 36px;
    padding: 0 14px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #374151;
    text-decoration: none;
    font-size: 14px;
    transition: 0.15s ease;
}

.mvc-nav a:hover {
    background: #e5e7eb;
    color: #111827;
}

.mvc-card {
    background: #ffffff;
    border: 1px solid #e6e9ef;
    border-radius: 20px;
    padding: 28px;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.06);
}

.mvc-page-title {
    margin: 0 0 12px;
    font-size: 28px;
    line-height: 1.2;
    color: #111827;
}

.mvc-page-text {
    margin: 0;
    font-size: 17px;
    line-height: 1.6;
    color: #4b5563;
}

.mvc-info {
    margin-top: 24px;
    padding: 18px;
    background: #f8fafc;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
}

.mvc-info b {
    color: #111827;
}

.mvc-info ol {
    margin: 12px 0 0;
    padding-left: 20px;
}

.mvc-info li {
    margin: 6px 0;
    color: #4b5563;
}

.mvc-code {
    display: inline-block;
    padding: 2px 6px;
    border-radius: 6px;
    background: #eef2ff;
    color: #3730a3;
    font-family: monospace;
    font-size: 13px;
}

@media (max-width: 700px) {
    .mvc-topbar {
        align-items: flex-start;
        flex-direction: column;
    }

    .mvc-card {
        padding: 20px;
    }

    .mvc-page-title {
        font-size: 24px;
    }
}


---

Шаг 2. Создаём layout

Создай папку:

/local/mvc/Views/layouts/

В ней файл:

/local/mvc/Views/layouts/app.php

Код:

<?php

/**
 * Layout
 *
 * Это общая рамка для страниц MVC.
 *
 * Внутри переменной $content лежит конкретная страница.
 * Например:
 * Views/home/index.php
 */

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-app">
    <div class="mvc-topbar">
        <div class="mvc-brand">
            <div class="mvc-brand__title">Local MVC</div>
            <div class="mvc-brand__subtitle">Учебный MVC-фреймворк внутри Битрикс24</div>
        </div>

        <nav class="mvc-nav">
            <a href="/local/mvc/">Главная</a>
            <a href="/local/mvc/about">О MVC</a>
            <a href="/local/mvc/ping" target="_blank">Ping JSON</a>
        </nav>
    </div>

    <?= $content ?? '' ?>
</div>

Что важно понять

Вот эта строка:

<?= $content ?? '' ?>

Это место, куда будет вставляться конкретная страница.

То есть layout говорит:

Я рисую общую оболочку.
А сюда вставьте содержимое страницы.


---

Шаг 3. Обновляем Controller.php

Теперь render() должен работать так:

1. Сначала собрать View в переменную $content.
2. Потом вставить $content внутрь Layout.
3. Потом отдать всё через Bitrix header/footer.

Полностью замени файл:

/local/mvc/Core/Controller.php

на этот:

<?php

namespace Local\Mvc\Core;

use Bitrix\Main\Page\Asset;

/**
 * Controller
 *
 * Базовый контроллер.
 */
class Controller
{
    /**
     * Текущий запрос.
     */
    protected Request $request;

    /**
     * Layout по умолчанию.
     */
    protected string $layout = 'layouts/app';

    public function __construct(?Request $request = null)
    {
        $this->request = $request ?? Request::createFromGlobals();
    }

    /**
     * Показать HTML-страницу.
     */
    protected function render(string $view, array $params = [], ?string $layout = null): Response
    {
        $viewFile = dirname(__DIR__) . '/Views/' . $view . '.php';

        if (!is_file($viewFile)) {
            return Response::html(
                '<h1>500</h1><p>View не найден.</p><pre>' . htmlspecialchars($viewFile) . '</pre>',
                500
            );
        }

        $layoutName = $layout ?? $this->layout;
        $layoutFile = dirname(__DIR__) . '/Views/' . $layoutName . '.php';

        if (!is_file($layoutFile)) {
            return Response::html(
                '<h1>500</h1><p>Layout не найден.</p><pre>' . htmlspecialchars($layoutFile) . '</pre>',
                500
            );
        }

        /**
         * Делаем переменные из массива.
         *
         * Например:
         * ['title' => 'Главная']
         *
         * станет:
         * $title = 'Главная';
         */
        extract($params);

        /**
         * 1. Собираем конкретную View.
         */
        ob_start();
        require $viewFile;
        $content = ob_get_clean();

        /**
         * 2. Подключаем CSS через механизм Битрикса.
         */
        if (class_exists(Asset::class)) {
            Asset::getInstance()->addCss('/local/mvc/assets/app.css');
        }

        /**
         * 3. Устанавливаем заголовок страницы Битрикса.
         */
        global $APPLICATION;

        if (isset($APPLICATION) && is_object($APPLICATION) && isset($title)) {
            $APPLICATION->SetTitle((string)$title);
        }

        /**
         * 4. Собираем итоговую страницу:
         * header Битрикса + layout + footer Битрикса.
         */
        ob_start();

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/header.php';

        require $layoutFile;

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/footer.php';

        $html = ob_get_clean();

        return Response::html($html);
    }

    /**
     * Вернуть произвольный JSON.
     */
    protected function json(array $data, int $status = 200): Response
    {
        return Response::json($data, $status);
    }

    /**
     * Успешный JSON-ответ.
     */
    protected function success(array $data = []): Response
    {
        return $this->json([
            'ok' => true,
            'data' => $data,
        ]);
    }

    /**
     * JSON-ошибка.
     */
    protected function error(string $message, array $details = [], int $status = 400): Response
    {
        return $this->json([
            'ok' => false,
            'error' => $message,
            'details' => $details,
        ], $status);
    }

    /**
     * Редирект.
     */
    protected function redirect(string $url): Response
    {
        return Response::redirect($url);
    }
}


---

Шаг 4. Обновляем View

Теперь в View больше не нужен большой контейнер с inline-стилями.

Замени файл:

/local/mvc/Views/home/index.php

на этот:

<?php

/**
 * View главной страницы.
 *
 * Здесь только содержимое страницы.
 * Меню, общий контейнер и стили находятся в layout.
 */

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Без заголовка') ?>
    </h1>

    <p class="mvc-page-text">
        <?= htmlspecialcharsbx($message ?? '') ?>
    </p>

    <div class="mvc-info">
        <b>Что сейчас происходит:</b>

        <ol>
            <li>Пользователь открывает адрес.</li>
            <li><span class="mvc-code">urlrewrite.php</span> отправляет запрос в MVC.</li>
            <li><span class="mvc-code">index.php</span> создаёт Request и Router.</li>
            <li><span class="mvc-code">Router</span> находит нужный контроллер.</li>
            <li><span class="mvc-code">Controller</span> собирает View.</li>
            <li><span class="mvc-code">Layout</span> оборачивает View в общий шаблон.</li>
            <li><span class="mvc-code">Response</span> отправляет HTML в браузер.</li>
        </ol>
    </div>
</div>


---

Шаг 5. Проверяем

Открой:

https://bitrix24-stage.gaz.ru/local/mvc/

Потом:

https://bitrix24-stage.gaz.ru/local/mvc/about

Теперь обе страницы должны быть внутри общего оформления:

верхняя панель
меню
карточка
единые стили


---

Что мы сделали

Раньше было так:

Controller
  ↓
View
  ↓
Bitrix header/footer

Теперь так:

Controller
  ↓
View собирается в $content
  ↓
Layout получает $content
  ↓
Bitrix header/footer
  ↓
Response отправляет HTML

Главная мысль:

View — это начинка.
Layout — это оболочка.

Теперь каждая новая страница может быть маленькой.

Например:

<div class="mvc-card">
    <h1>Список сайтов</h1>
</div>

А меню, контейнер, CSS и общая структура уже будут браться из layout.

Следующим шагом можно делать Base Model + подключение к PostgreSQL, чтобы наш MVC начал работать с данными.