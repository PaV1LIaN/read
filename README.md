Идём дальше: теперь сделаем дублирование страницы.

Это тоже очень полезно для серьёзного проекта: можно быстро копировать готовые страницы вместе со всеми блоками.


---

1. В api.php добавь action page.duplicate

Вставь его рядом с другими page-action, лучше после page.delete или после page.move:

if ($action === 'page.duplicate') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $srcPage = sb_find_page($id);
    if (!$srcPage) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($srcPage['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $blocks = sb_read_blocks();

    $maxPageId = 0;
    foreach ($pages as $p) {
        $maxPageId = max($maxPageId, (int)($p['id'] ?? 0));
    }
    $newPageId = $maxPageId + 1;

    $srcTitle = (string)($srcPage['title'] ?? 'Страница');
    $newTitle = $srcTitle . ' (копия)';

    $baseSlug = sb_slugify((string)($srcPage['slug'] ?? ($srcPage['title'] ?? 'page')));
    if ($baseSlug === '') $baseSlug = 'page';

    $newSlug = $baseSlug . '-copy';

    $existing = array_map(
        fn($x) => (string)($x['slug'] ?? ''),
        array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId)
    );

    $base = $newSlug;
    $i = 2;
    while (in_array($newSlug, $existing, true)) {
        $newSlug = $base . '-' . $i;
        $i++;
    }

    $srcSort = (int)($srcPage['sort'] ?? 500);

    // вставляем копию сразу после исходной страницы среди соседей
    foreach ($pages as &$p) {
        if (
            (int)($p['siteId'] ?? 0) === $siteId &&
            (int)($p['parentId'] ?? 0) === (int)($srcPage['parentId'] ?? 0) &&
            (int)($p['sort'] ?? 0) > $srcSort
        ) {
            $p['sort'] = (int)($p['sort'] ?? 0) + 10;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($p);

    $newPage = $srcPage;
    $newPage['id'] = $newPageId;
    $newPage['title'] = $newTitle;
    $newPage['slug'] = $newSlug;
    $newPage['sort'] = $srcSort + 10;
    $newPage['createdBy'] = (int)$USER->GetID();
    $newPage['createdAt'] = date('c');
    $newPage['updatedAt'] = date('c');
    $newPage['updatedBy'] = (int)$USER->GetID();

    // если у тебя уже есть статус страницы — сохраняем draft для копии
    if (array_key_exists('status', $srcPage)) {
        $newPage['status'] = 'draft';
        $newPage['publishedAt'] = null;
    }

    $pages[] = $newPage;

    // копируем блоки страницы
    $maxBlockId = 0;
    foreach ($blocks as $b) {
        $maxBlockId = max($maxBlockId, (int)($b['id'] ?? 0));
    }
    $nextBlockId = $maxBlockId + 1;

    $srcBlocks = array_values(array_filter($blocks, fn($b) => (int)($b['pageId'] ?? 0) === $id));
    usort($srcBlocks, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    foreach ($srcBlocks as $b) {
        $copy = $b;
        $copy['id'] = $nextBlockId++;
        $copy['pageId'] = $newPageId;
        $copy['createdBy'] = (int)$USER->GetID();
        $copy['createdAt'] = date('c');
        $copy['updatedAt'] = date('c');
        $blocks[] = $copy;
    }

    sb_write_pages($pages);
    sb_write_blocks($blocks);

    echo json_encode([
        'ok' => true,
        'page' => $newPage
    ], JSON_UNESCAPED_UNICODE);
    exit;
}


---

2. В index.php в дереве страниц добавь кнопку “Дублировать”

В renderPagesTree() внутри .nodeBtns добавь кнопку:

<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-duplicate="${node.id}">Дублировать</button>

Вставить лучше вот сюда, рядом с Имя/slug:

<div class="nodeBtns">
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="up">↑</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="down">↓</button>

  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-parent="${node.id}">Вложить…</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-root="${node.id}">В корень</button>

  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>
  <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-duplicate="${node.id}">Дублировать</button>

  <a class="ui-btn ui-btn-primary ui-btn-xs btnTiny"
     href="/local/sitebuilder/editor.php?siteId=${siteId}&pageId=${node.id}"
     target="_blank">Редактор</a>

  <a class="ui-btn ui-btn-light ui-btn-xs btnTiny"
     href="/local/sitebuilder/view.php?siteId=${siteId}&pageId=${node.id}"
     target="_blank">Открыть</a>

  <button class="ui-btn ui-btn-danger ui-btn-xs btnTiny" data-page-delete="${node.id}">Удалить</button>
</div>


---

3. В index.php добавь обработчик клика

Внутри container.addEventListener('click', function(e){ ... }) добавь блок:

const dup = e.target.closest('[data-page-duplicate]');
if (dup) {
  const id = parseInt(dup.getAttribute('data-page-duplicate'), 10);

  BX.UI.Dialogs.MessageBox.show({
    title: 'Дублировать страницу #' + id + '?',
    message: 'Будет создана копия страницы вместе со всеми блоками.',
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      api('page.duplicate', { id }).then(r => {
        if (!r || r.ok !== true) {
          notify('Не удалось дублировать страницу');
          return;
        }
        notify('Страница продублирована');
        mb.close();
        loadAndRender();
      }).catch(() => notify('Ошибка page.duplicate'));
    }
  });
  return;
}

Лучше вставить его до удаления страницы, чтобы обработчики шли в нормальном порядке.


---

Как проверить

1. Открой index.php.


2. Зайди в Страницы.


3. У любой страницы нажми Дублировать.


4. Должна появиться:

новая страница рядом с исходной,

с названием Имя страницы (копия),

с новым slug,

со всеми теми же блоками.



5. Открой копию в редакторе и проверь, что блоки реально скопировались.



Если это ок, следующим шагом сделаем статус draft/published прямо в дереве страниц, чтобы сразу видеть состояние страницы и быстро переключать его.