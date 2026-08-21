<?php

declare(strict_types=1);

header('Content-Type: application/json');

function respond(array $data, int $statusCode = 200) {
    http_response_code($statusCode);

    echo json_encode($data, JSON_UNESCAPED_UNICODE);

	exit();
}

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
	respond([
		'error' => "METHOD_NOT_POST",
		'message' => "Метод должен быть POST"
	],
	405
	);
}

$bodyRaw = file_get_contents('php://input');
$data = json_decode($bodyRaw, true);

if (!is_array($data)) {
	respond([
		'error' => '$data_IS_NOT_ARRAY',
		'message' => '$data должен быть массивом'
	],
	400
	);
}
