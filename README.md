Начнём с дублирования блока — это быстрее и сразу полезно.

Потом следующим шагом сделаем:

дублирование страницы,

статус draft/published прямо в списке страниц.


Для дублирования блока нужно добавить:

1. В api.php новый action block.duplicate

Вставь этот блок рядом с другими block-action, лучше после block.delete или после block.move:

if ($action === 'block.duplicate') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $src = sb_find_block($id);
    if (!$src) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page((int)($src['pageId'] ?? 0));
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();

    $maxId = 0;
    foreach ($blocks as $b) {
        $maxId = max($maxId, (int)($b['id'] ?? 0));
    }
    $newId = $maxId + 1;

    $srcSort = (int)($src['sort'] ?? 500);

    // Сдвигаем все блоки ниже, чтобы вставить копию сразу после исходного
    foreach ($blocks as &$b) {
        if ((int)($b['pageId'] ?? 0) === (int)($src['pageId'] ?? 0) && (int)($b['sort'] ?? 0) > $srcSort) {
            $b['sort'] = (int)($b['sort'] ?? 0) + 10;
            $b['updatedAt'] = date('c');
        }
    }
    unset($b);

    $copy = $src;
    $copy['id'] = $newId;
    $copy['sort'] = $srcSort + 10;
    $copy['createdBy'] = (int)$USER->GetID();
    $copy['createdAt'] = date('c');
    $copy['updatedAt'] = date('c');

    $blocks[] = $copy;
    sb_write_blocks($blocks);

    echo json_encode(['ok'=>true,'block'=>$copy], JSON_UNESCAPED_UNICODE);
    exit;
}


---

2. В editor.php добавить кнопку “Дублировать”

Во всех карточках блока, где сейчас есть кнопки:

↑

↓

Редактировать

Удалить


добавь между Редактировать и Удалить:

<button class="ui-btn ui-btn-light ui-btn-xs" data-dup-block-id="${id}">Дублировать</button>

То есть, например, для text станет так:

<div class="btns">
  <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
  <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
  <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
  <button class="ui-btn ui-btn-light ui-btn-xs" data-dup-block-id="${id}">Дублировать</button>
  <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
</div>

И так же для:

image

button

heading

columns2

gallery

spacer

card

cards



---

3. В editor.php добавить обработчик клика

В document.addEventListener('click', function (e) { ... }) вставь до удаления блока вот этот кусок:

const dupBtn = e.target.closest('[data-dup-block-id]');
if (dupBtn) {
  const id = parseInt(dupBtn.getAttribute('data-dup-block-id'), 10);

  api('block.duplicate', { id })
    .then(r => {
      if (!r || r.ok !== true) {
        notify('Не удалось дублировать блок');
        return;
      }
      notify('Блок продублирован');
      loadBlocks();
    })
    .catch(() => notify('Ошибка block.duplicate'));
  return;
}


---

Как проверить

1. Открой страницу в редакторе.


2. У любого блока нажми Дублировать.


3. Должна появиться копия сразу под исходным блоком.


4. Проверь несколько типов:

text

image

button

cards




Если это сработает, следующим сообщением сделаем дублирование страницы.