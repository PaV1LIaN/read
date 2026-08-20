<?php

declare(strict_types=1);

header('Content-Type: application/json; charset=utf-8');

$aRaw = trim((string)($_GET['a'] ?? ''));
$bRaw = trim((string)($_GET['b'] ?? ''));
$operation = trim((string)($_GET['operation'] ?? ''));

function respond(array $data, int $statusCode = 200): never {
    http_response_code($statusCode);

    echo json_encode($data, JSON_UNESCAPED_UNICODE);
    exit();
}

if($aRaw === null || $bRaw === null || $operation === null) {
    respond(
        [
            'error' => 'PARAMETERS_REQUIRED',
            'message' => 'Необходимо передать a, b и operation'
        ]
    );
}

if(!is_numeric($aRaw) || !is_numeric($bRaw)) {
    respond(
        [
            'error' => 'NOT_NUMERIC',
            'message' => 'а и б должны быть числом'
        ],
        400
    );
}

$a = (float)$aRaw;
$b = (float)$bRaw;

if((float)$b === 0.0 && $operation === 'divide') {
    respond(
        [
            'error' => 'DIVISION_BY_ZERO',
            'message' => 'На ноль делить нельзя'
        ],
        400
    );
}

$result = match($operation) {
    'add' => $a + $b,
    'subtract' => $a - $b,
    'multiply' => $a * $b,
    'divide' => $a / $b,
    default => null
};

if($result == null) {
    respond(
        [
            'error' => 'UNKNOWN_OPERATION',
            'message' => 'Неизвестная операция',
        ],
        400
    );
}

respond(
    [
        'data' => [
            'a' => $a,
            'b' => $b,
            'operation' => $operation,
            'result' => $result
        ]
    ]
);
