<?php

/**
 * Диагностический index.php
 * Нужен только чтобы увидеть настоящую ошибку 500.
 */

ini_set('display_errors', '1');
ini_set('display_startup_errors', '1');
error_reporting(E_ALL);

register_shutdown_function(function () {
    $error = error_get_last();

    if ($error !== null) {
        echo '<pre style="background:#300;color:#fff;padding:20px;border-radius:8px;">';
        echo "FATAL ERROR:\n\n";
        print_r($error);
        echo '</pre>';
    }
});

try {
    require_once __DIR__ . '/bootstrap.php';

    echo '<pre style="background:#eef;padding:15px;border-radius:8px;">';
    echo "bootstrap.php подключился успешно\n";
    echo '</pre>';

    useController();

} catch (Throwable $e) {
    echo '<pre style="background:#300;color:#fff;padding:20px;border-radius:8px;">';
    echo "EXCEPTION / ERROR:\n\n";
    echo $e->getMessage() . "\n\n";
    echo "File: " . $e->getFile() . "\n";
    echo "Line: " . $e->getLine() . "\n\n";
    echo $e->getTraceAsString();
    echo '</pre>';
}

function useController(): void
{
    $class = '\\Local\\Mvc\\Controllers\\HomeController';

    echo '<pre style="background:#efe;padding:15px;border-radius:8px;">';
    echo "Пробуем загрузить класс: {$class}\n";
    echo '</pre>';

    if (!class_exists($class)) {
        echo '<pre style="background:#800;color:#fff;padding:20px;border-radius:8px;">';
        echo "Класс НЕ найден: {$class}\n\n";
        echo "Проверь файл:\n";
        echo $_SERVER['DOCUMENT_ROOT'] . "/local/mvc/Controllers/HomeController.php\n";
        echo '</pre>';
        return;
    }

    $controller = new $class();

    $action = $_GET['action'] ?? 'index';

    if ($action === 'ping') {
        $controller->ping();
        return;
    }

    $controller->index();
}