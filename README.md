Сейчас уже не надо трогать роутер наугад. Нужно посмотреть, что реально лежит в файлах на сервере.

Раз api.php в прямом режиме работает, используем его как “зонд”.

Замени /local/sitebuilder/api.php на этот диагностический файл

<?php
header('Content-Type: application/json; charset=UTF-8');

$files = [
    'api_index'      => __DIR__ . '/api/index.php',
    'handler_common' => __DIR__ . '/api/handlers/common.php',
    'handler_site'   => __DIR__ . '/api/handlers/site.php',
    'handler_access' => __DIR__ . '/api/handlers/access.php',
];

$result = [
    'ok' => true,
    'apiFile' => __FILE__,
    'time' => date('c'),
    'files' => [],
];

foreach ($files as $key => $path) {
    $exists = file_exists($path);
    $content = $exists ? file_get_contents($path) : '';
    $result['files'][$key] = [
        'path' => $path,
        'exists' => $exists,
        'md5' => $exists ? md5_file($path) : null,
        'size' => $exists ? filesize($path) : null,
        'head' => $exists ? substr($content, 0, 400) : null,
    ];
}

echo json_encode($result, JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT);
exit;

Потом открой:

https://portal24.itsnn.ru/local/sitebuilder/api.php

И пришли мне весь JSON.

Нас интересует:

что реально лежит в api/index.php

что реально лежит в api/handlers/common.php

что реально лежит в api/handlers/access.php


Сейчас почти наверняка окажется одно из двух:

в common.php реально лежит не тот код

в index.php не тот код

или common.php и access.php перепутаны содержимым


После этого я скажу, какой именно файл у тебя подменён.