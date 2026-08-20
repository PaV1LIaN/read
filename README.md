<?php

declare(stict_types=1);

header('Content-Type: application/json; charset=utf-8');

$aRaw = trim((string)($_GET['a'] ?? ''));
$bRaw = trim((string)($_GET['b'] ?? ''));
$operation = trim((string)($_GET['operation'] ?? ''));

function respond(array $data, int $statusCode = 200) {
    http_response_code($statusCode);

    echo json_encode($data, JSON_UNESCAPED_UNICODE);
    exit();
}

if(!is_numeric($aRaw) || !is_numeric($bRaw) || $aRaw === '' || $bRaw === '') {
    respond(
        [
            'error' => 'NOT_NUMERIC',
            'message' => 'а и б должны быть числом'
        ],
        400
    );
}

if((float)$bRaw === 0.0) {
    respond(
        [
            'error' => 'DIVISION_BY_ZERO',
            'message' => 'На ноль делить нельзя'
        ],
        400
    );
}

$result = match($operation) {
    'add' => $aRaw + $bRaw,
    'subtract' => $aRaw - $bRaw,
    'multiply' => $aRaw * $bRaw,
    'divide' => $aRaw / $bRaw,
    default => null
};

if($result === null) {
    respond(
        [
            'error' => 'UNKNOWN_OPERATION',
            'message' => 'Неизвестная операция',
        ],
        400
    );
}

echo respond(
    [
        'a' => $aRaw,
        'b' => $bRaw,
        'operation' => $operation,
        'result' => $result
    ]
);
