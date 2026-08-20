<?php       

declare(strict_types=1);

header('Content-Type: application/json;
charset=utf-8');

$a = trim(($_GET['a'] ?? ''));
$b = trim(($_GET['b'] ?? ''));
$operation = trim(($_GET['operation'] ?? ''));

if($a === '' || $b === '' || $operation === '') {
    http_response_code(400);

    echo json_encode(
        [
            'error' => 'PARAMETRS_REQUIRED',
            'message' => 'Необходимо передать a, b и operation',
        ],
		JSON_UNESCAPED_UNICODE);

		exit;
}

if($operation === 'divide' && $b == 0) {
	http_response_code(400);

	echo json_encode(
		[
			'error' => 'DIVISION_BY_ZERO',
			'message' => 'На ноль делить нельзя',
		],
		JSON_UNESCAPED_UNICODE
	);

	exit;

}

$result = match($operation) {
    'add' => $a + $b,
    'substract' => $a - $b,
    'multiply' => $a * $b,
    'divide' => $a / $b,
	default => (http_response_code(400)),
};


http_response_code(200);

echo json_encode(
	[	
		'result' => $result,
	]);
