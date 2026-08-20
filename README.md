<?php

declare(strict_types=1);

header('Content-Type: application/json; charset=utf-8');

function respond(array $data, int $statusCode = 200): never {
    http_response_code($statusCode);

    echo json_encode(
        $data,
        JSON_UNESCAPED_UNICODE
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

if(!is_array($data)) {
    respond(
        [
            'error' => 'INVALID_JSON',
            'message' => 'Некорректный JSON'
        ],
        400
    );
}

$nameRaw = $data['name'] ?? '';

$ageRaw = $data['age'] ?? '';

if($nameRaw === '') {
    respond(
        [
            'error' => 'NAME_REQUIRED',
            'message' => 'Необходимо передать имя',
        ],
    400
    );
}

if(!is_numeric($ageRaw)) {
    respond(
        [
            'error' => 'INVALID_AGE',
            'message' => 'Возраст должен быть числом больше 0'
        ],
        400
    );
}

$name = trim((string)$nameRaw);
$age = (int)$ageRaw;

if($age <= 0) {
    respond(
        [
            'error' => 'INVALID_AGE',
            'message' => 'Возраст должен быть числом больше 0'
        ]
    );
}

$id = 1;

respond([
    'data' => [
        'id' => $id,
        'name' => $name,
        'age' => $age
    ],
    201
]);
