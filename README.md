<?php

declare(strict_types=1);

header('Content-Type: application/json; charset=utf-8');

function respond(array $data, int $statusCode = 200): never {
    http_response_code($statusCode);

    echo json_encode(
        [
            $data,
            JSON_UNESCAPED_UNICODE
        ],
        $statusCode
    );

    exit();
}

if($_SERVER['REQUEST_METHOD'] !== 'POST') {
    respond(
        [
            'error' => 'METHOD_NOT_ALLOWED',
            'message' => 'Метод не POST'
        ],
        405
    );
}

$rawBody = file_get_contents('php://input');

$data = json_decode($rawBody, true);

$nameRaw = $data['name'];

$ageRaw = $data['age'];

if($nameRaw === '') {
    respond(
        [
            'error' => 'NAME_REQUIRED',
            'message' => 'Необходимо передать имя',
        ],
    400
    );
}

if($ageRaw <= 0 && !is_numeric($ageRaw)) {
    respond(
        [
            'error' => 'INVALID_AGE',
            'message' => 'Возраст должен быть числом больше 0'
        ],
        400
    );
}

$name = (string)$nameRaw;
$age = (float)$ageRaw;

respond([
    'data' => [
        'id' => $id,
        'name' => $name,
        'age' => $age
    ]
]);'
