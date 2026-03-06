Отлично. Тогда делаем статус страницы DRAFT / PUBLISHED аккуратно и без путаницы.

Сейчас дам:

1. что добавить в api.php


2. что вставить в editor.php


3. как проверить




---

1. Что добавить в api.php

1.1. При создании страницы добавить status

Найди в page.create место, где собирается массив $page:

$page = [
    'id' => $id,
    'siteId' => $siteId,
    'title' => $title,
    'slug' => $slug,
    'parentId' => 0,
    'sort' => 500,
    'createdBy' => (int)$USER->GetID(),
    'createdAt' => date('c'),
];

Замени на:

$page = [
    'id' => $id,
    'siteId' => $siteId,
    'title' => $title,
    'slug' => $slug,
    'parentId' => 0,
    'sort' => 500,
    'status' => 'DRAFT',
    'createdBy' => (int)$USER->GetID(),
    'createdAt' => date('c'),
];


---

1.2. Добавить action page.setStatus

Вставь ниже блока page.updateMeta и выше page.setParent вот это:

if ($action === 'page.setStatus') {
    $id = (int)($_POST['id'] ?? 0);
    $status = strtoupper(trim((string)($_POST['status'] ?? '')));

    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if (!in_array($status, ['DRAFT', 'PUBLISHED'], true)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'STATUS_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page($id);
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $found = false;

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $p['status'] = $status;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($p);

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_pages($pages);
    echo json_encode(['ok'=>true, 'status'=>$status], JSON_UNESCAPED_UNICODE);
    exit;
}


---

2. Что вставить в editor.php

Теперь добавим в редакторе переключатель статуса страницы.

2.1. В верхнюю панель добавить статус

Найди вверху этот участок:

<div><b>siteId:</b> <code><?= (int)$siteId ?></code></div>
<div><b>pageId:</b> <code><?= (int)$pageId ?></code></div>

И сразу после него вставь:

<div><b>Статус:</b> <span id="pageStatusBadge" class="muted">...</span></div>


---

2.2. Добавить кнопку смены статуса

Найди блок верхних кнопок, где есть:

<a class="ui-btn ui-btn-light" target="_blank"
   href="/local/sitebuilder/view.php?siteId=<?= (int)$siteId ?>&pageId=<?= (int)$pageId ?>">Открыть просмотр</a>

И сразу после него вставь:

<button class="ui-btn ui-btn-light" id="btnToggleStatus">Сменить статус</button>


---

2.3. В JS добавить переменные

Найди:

const btnSaveTemplate = document.getElementById('btnSaveTemplate');
const btnApplyTemplate = document.getElementById('btnApplyTemplate');
const btnSections = document.getElementById('btnSections');

И сразу после добавь:

const btnToggleStatus = document.getElementById('btnToggleStatus');
const pageStatusBadge = document.getElementById('pageStatusBadge');

let pageMeta = null;


---

2.4. Добавить функцию загрузки информации о странице

Найди место после function fileDownloadUrl(fileId) { ... } и вставь ниже:

async function loadPageMeta() {
  const res = await api('page.list', { siteId });
  if (!res || res.ok !== true) throw new Error('page.list failed');

  const pages = res.pages || [];
  pageMeta = pages.find(p => parseInt(p.id, 10) === pageId) || null;

  const status = (pageMeta && pageMeta.status) ? String(pageMeta.status).toUpperCase() : 'DRAFT';

  if (pageStatusBadge) {
    pageStatusBadge.textContent = status;
    pageStatusBadge.style.color = (status === 'PUBLISHED') ? '#15803d' : '#b45309';
    pageStatusBadge.style.fontWeight = '700';
  }

  if (btnToggleStatus) {
    btnToggleStatus.textContent = (status === 'PUBLISHED')
      ? 'Сделать DRAFT'
      : 'Опубликовать';
  }
}


---

2.5. Добавить функцию смены статуса

Ниже вставь ещё:

function togglePageStatus() {
  if (!pageMeta) {
    notify('Страница ещё не загружена');
    return;
  }

  const currentStatus = String(pageMeta.status || 'DRAFT').toUpperCase();
  const nextStatus = (currentStatus === 'PUBLISHED') ? 'DRAFT' : 'PUBLISHED';

  api('page.setStatus', { id: pageId, status: nextStatus })
    .then(res => {
      if (!res || res.ok !== true) {
        notify('Не удалось сменить статус');
        return;
      }
      notify('Статус изменён: ' + nextStatus);
      loadPageMeta();
    })
    .catch(() => notify('Ошибка page.setStatus'));
}


---

2.6. Повесить обработчик на кнопку

Найди внизу блок кнопок:

btnSections.addEventListener('click', openSectionsLibrary);
btnSaveTemplate.addEventListener('click', saveTemplateFromPage);
btnApplyTemplate.addEventListener('click', applyTemplateToPage);

loadBlocks();

Замени на:

btnSections.addEventListener('click', openSectionsLibrary);
btnSaveTemplate.addEventListener('click', saveTemplateFromPage);
btnApplyTemplate.addEventListener('click', applyTemplateToPage);
if (btnToggleStatus) btnToggleStatus.addEventListener('click', togglePageStatus);

loadBlocks();
loadPageMeta();


---

3. Как проверить

Сейчас проверка должна быть такой:

Проверка 1

Создай новую страницу.
Она должна создаться со статусом DRAFT.

Проверка 2

Зайди в editor.php этой страницы.
Вверху должно быть:

Статус: DRAFT

кнопка Опубликовать


Проверка 3

Нажми Опубликовать.
Должно стать:

Статус: PUBLISHED

кнопка Сделать DRAFT


Проверка 4

Нажми обратно.
Должно снова стать DRAFT.


---

После этого следующим сообщением я дам уже точные вставки в index.php, чтобы статус страницы был виден прямо в дереве страниц.