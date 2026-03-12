Отлично. Переходим к контентным областям страницы.

Идём правильно, по этапам, чтобы ничего не сломать:

Что сделаем сначала

Добавим новый тип блока: section

Он будет задавать:

фон секции

отступ сверху

отступ снизу

boxed / full width

border

radius


А обычные блоки будут идти внутри секции по правилу:

встретили section → начали новую секцию

все следующие блоки рендерятся в неё

встретили следующую section → закрыли предыдущую


Это самый мягкий способ встроить секции в твою текущую архитектуру.


---

Шаг 1. Дорабатываем api.php

1. Разреши section в block.create

Найди в block.create этот кусок:

if (!in_array($type, ['text','image','button', 'heading', 'columns2', 'gallery', 'spacer', 'card', 'cards'], true)) {

Замени на:

if (!in_array($type, ['text','image','button','heading','columns2','gallery','spacer','card','cards','section'], true)) {


---

2. Добавь обработку section в block.create

Внутри block.create, перед последним else { // button ... }, вставь новый блок:

} elseif ($type === 'section') {
    $boxed = (string)($_POST['boxed'] ?? '1');
    $boxed = ($boxed === '1' || $boxed === 'true');

    $background = trim((string)($_POST['background'] ?? '#ffffff'));
    if (!preg_match('~^#[0-9a-fA-F]{6}$~', $background)) $background = '#ffffff';

    $paddingTop = (int)($_POST['paddingTop'] ?? 32);
    $paddingBottom = (int)($_POST['paddingBottom'] ?? 32);

    if ($paddingTop < 0) $paddingTop = 0;
    if ($paddingTop > 200) $paddingTop = 200;
    if ($paddingBottom < 0) $paddingBottom = 0;
    if ($paddingBottom > 200) $paddingBottom = 200;

    $border = (string)($_POST['border'] ?? '0');
    $border = ($border === '1' || $border === 'true');

    $radius = (int)($_POST['radius'] ?? 0);
    if ($radius < 0) $radius = 0;
    if ($radius > 40) $radius = 40;

    $content = [
        'boxed' => $boxed,
        'background' => strtoupper($background),
        'paddingTop' => $paddingTop,
        'paddingBottom' => $paddingBottom,
        'border' => $border,
        'radius' => $radius,
    ];


---

3. Разреши section в block.update

Найди в block.update обработчики типов и перед последним else { TYPE_NOT_SUPPORTED } вставь:

} elseif ($type === 'section') {
    $boxed = (string)($_POST['boxed'] ?? '1');
    $boxed = ($boxed === '1' || $boxed === 'true');

    $background = trim((string)($_POST['background'] ?? '#ffffff'));
    if (!preg_match('~^#[0-9a-fA-F]{6}$~', $background)) $background = '#ffffff';

    $paddingTop = (int)($_POST['paddingTop'] ?? 32);
    $paddingBottom = (int)($_POST['paddingBottom'] ?? 32);

    if ($paddingTop < 0) $paddingTop = 0;
    if ($paddingTop > 200) $paddingTop = 200;
    if ($paddingBottom < 0) $paddingBottom = 0;
    if ($paddingBottom > 200) $paddingBottom = 200;

    $border = (string)($_POST['border'] ?? '0');
    $border = ($border === '1' || $border === 'true');

    $radius = (int)($_POST['radius'] ?? 0);
    if ($radius < 0) $radius = 0;
    if ($radius > 40) $radius = 40;

    $b['content']['boxed'] = $boxed;
    $b['content']['background'] = strtoupper($background);
    $b['content']['paddingTop'] = $paddingTop;
    $b['content']['paddingBottom'] = $paddingBottom;
    $b['content']['border'] = $border;
    $b['content']['radius'] = $radius;


---

Шаг 2. Что дальше

После этого следующий шаг — в public.php сделать рендер секций.

То есть:

сейчас у тебя блоки рендерятся по одному

мы добавим группировку по section



---

Важно

Пока после правок в api.php секция ещё не будет красиво отображаться в публичке — это нормально.
Сначала мы добавляем поддержку типа, потом рендер.


---

Что делаем следующим сообщением

После того как внесёшь эти правки, я дам тебе:

готовый код для public.php,

чтобы section реально начал работать как контейнер контента.