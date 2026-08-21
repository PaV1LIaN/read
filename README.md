<?php

declare(strict_types=1);

header('Content-Type: application/json; charser=utf-8');

function respond(array $data, int $statusCode = 200): never {
    http_response_code($statusCode);

    echo json_encode($data, JSON_UNESCAPED_UNICODE);

	exit();
}

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
	respond([
		'error' => "METHOD_NOT_ALLOWED",
		'message' => "Метод должен быть POST"
	],
	405
	);
}

$bodyRaw = file_get_contents('php://input');
$data = json_decode($bodyRaw, true);

if (!is_array($data)) {
	respond([
		'error' => '$INVALID_JSON',
		'message' => '$data должен быть массивом'
	],
	400
	);
}

$titleRaw = ($data['title'] ?? '');
$priorityRaw = ($data['priority'] ?? '');
$deviceCountRaw = ($data['deviceCount'] ?? '');

