<?php

/**
 * index.php
 *
 * Это входная точка нашего MVC.
 *
 * Пока здесь всё очень просто:
 * 1. Подключаем bootstrap.php.
 * 2. Создаём HomeController.
 * 3. Вызываем нужный метод.
 */

require_once __DIR__ . '/bootstrap.php';

use Local\Mvc\Controllers\HomeController;

/**
 * Получаем action из адреса.
 *
 * Например:
 * /local/mvc/?action=ping
 *
 * Тогда:
 * $action = 'ping';
 *
 * Если action не передали,
 * будет:
 * $action = 'index';
 */
$action = $_GET['action'] ?? 'index';

/**
 * Создаём контроллер.
 */
$controller = new HomeController();

/**
 * Если пользователь запросил ping:
 *
 * /local/mvc/?action=ping
 *
 * вернём JSON.
 */
if ($action === 'ping') {
    $controller->ping();
    exit;
}

/**
 * По умолчанию показываем главную страницу.
 */
$controller->index();