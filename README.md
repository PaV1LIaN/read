} elseif ($type === 'columns2') {
    $left  = (string)($_POST['left'] ?? '');
    $right = (string)($_POST['right'] ?? '');
    $ratio = (string)($_POST['ratio'] ?? '50-50');

    $ratio = trim($ratio);
    if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

    $content = [
        'left' => $left,
        'right' => $right,
        'ratio' => $ratio,
    ];
}
