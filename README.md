Начинаем с index.php, в блоке списка страниц.

Что улучшим

В дереве страниц сделаем:

яркий бейдж DRAFT / PUBLISHED

бейдж HOME

более читаемую мета-строку: sort, parent, slug

визуально выделим draft-страницы


Ниже готовые правки.


---

1. Добавь стили в <style>

Вставь рядом со стилями .node, .nodeMeta, .nodeSlug вот это:

.pageBadge{
  display:inline-flex;
  align-items:center;
  gap:6px;
  padding:3px 8px;
  border-radius:999px;
  font-size:11px;
  font-weight:700;
  line-height:1;
  border:1px solid transparent;
}

.pageBadgePublished{
  background:#ecfdf3;
  border-color:#b7ebc6;
  color:#027a48;
}

.pageBadgeDraft{
  background:#fff4ed;
  border-color:#ffd6ae;
  color:#b54708;
}

.pageBadgeHome{
  background:#eef2ff;
  border-color:#c7d2fe;
  color:#3730a3;
}

.node.isDraft{
  background:#fffaf5;
  border-color:#f5d7b2;
}

.nodeBadges{
  display:flex;
  gap:6px;
  flex-wrap:wrap;
  align-items:center;
}

.nodeMeta code{
  background:#f3f4f6;
  padding:2px 6px;
  border-radius:999px;
  font-size:11px;
}


---

2. Обнови renderPagesTree(container, siteId, pages, q)

Полностью замени функцию renderPagesTree на эту:

function renderPagesTree(container, siteId, pages, q) {
  const query = (q || '').trim().toLowerCase();

  const homePageId = pages.reduce((acc, p) => {
    if (parseInt(p.isHome || 0, 10) === 1) return parseInt(p.id, 10);
    return acc;
  }, 0);

  const matches = (p) => {
    if (!query) return true;
    const t = (p.title || '').toLowerCase();
    const s = (p.slug || '').toLowerCase();
    const status = (p.status || '').toLowerCase();
    return t.includes(query) || s.includes(query) || status.includes(query) || String(p.id).includes(query);
  };

  const { roots } = buildTree(pages);

  const renderNode = (node) => {
    const kidsHtml = (node.children || [])
      .map(renderNode)
      .filter(Boolean)
      .join('');

    const selfMatch = matches(node);
    const hasVisibleKids = kidsHtml !== '';
    if (!selfMatch && !hasVisibleKids) return '';

    const pid = parseInt(node.parentId || 0, 10) || 0;
    const parentLabel = pid ? `parent: #${pid}` : 'root';

    const status = String(node.status || 'published').toLowerCase() === 'draft' ? 'draft' : 'published';
    const isDraft = status === 'draft';
    const isHome = parseInt(node.id, 10) === homePageId || parseInt(node.homePageId || 0, 10) === parseInt(node.id, 10);

    const statusBadge = isDraft
      ? '<span class="pageBadge pageBadgeDraft">DRAFT</span>'
      : '<span class="pageBadge pageBadgePublished">PUBLISHED</span>';

    const homeBadge = isHome
      ? '<span class="pageBadge pageBadgeHome">HOME</span>'
      : '';

    const title = BX.util.htmlspecialchars(node.title || '');
    const slug = BX.util.htmlspecialchars(node.slug || '');
    const sort = parseInt(node.sort || 500, 10);

    return `
      <div class="node ${isDraft ? 'isDraft' : ''}">
        <div class="nodeHead">
          <div class="nodeLeft">
            <div class="nodeIcon">≡</div>
            <div class="nodeMain">
              <div class="nodeTitleLine">
                <b>#${node.id} ${title}</b>
                <span class="nodeSlug">${slug}</span>
              </div>

              <div class="nodeBadges" style="margin-top:6px;">
                ${statusBadge}
                ${homeBadge}
              </div>

              <div class="nodeMeta">
                <span>sort: <code>${sort}</code></span>
                <span>${parentLabel}</span>
                <span>slug: <code>${slug}</code></span>
              </div>
            </div>
          </div>

          <div class="nodeBtns">
            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="up">↑</button>
            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="down">↓</button>

            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-parent="${node.id}">Вложить…</button>
            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-root="${node.id}">В корень</button>

            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>

            <a class="ui-btn ui-btn-primary ui-btn-xs btnTiny"
               href="/local/sitebuilder/editor.php?siteId=${siteId}&pageId=${node.id}"
               target="_blank">Редактор</a>

            <a class="ui-btn ui-btn-light ui-btn-xs btnTiny"
               href="/local/sitebuilder/view.php?siteId=${siteId}&pageId=${node.id}"
               target="_blank">Открыть</a>

            <button class="ui-btn ui-btn-danger ui-btn-xs btnTiny" data-page-delete="${node.id}">Удалить</button>
          </div>
        </div>

        ${hasVisibleKids ? `<div class="children">${kidsHtml}</div>` : ''}
      </div>
    `;
  };

  const html = roots.map(renderNode).filter(Boolean).join('');
  container.innerHTML = html || '<div class="muted">Страниц пока нет.</div>';
}


---

3. Небольшая доработка loadAndRender

Сейчас page.list должен отдавать статус страницы. Если уже отдаёт status, всё ок.

Но для бейджа HOME удобнее явно пробросить homePageId в массив страниц после загрузки.
Внутри openPagesDialog, в функции loadAndRender, замени этот кусок:

pagesCache = res.pages || [];
renderPagesTree(container, siteId, pagesCache, q);

на:

pagesCache = (res.pages || []).map(p => Object.assign({}, p, {
  isHome: parseInt(res.homePageId || 0, 10) === parseInt(p.id || 0, 10) ? 1 : 0
}));
renderPagesTree(container, siteId, pagesCache, q);


---

4. Если page.list ещё не возвращает homePageId

Тогда в api.php в экшене page.list надо вернуть его.

Найди:

echo json_encode(['ok' => true, 'pages' => $pages], JSON_UNESCAPED_UNICODE);

и замени на:

$homePageId = 0;
$site = sb_find_site($siteId);
if ($site) {
    $homePageId = (int)($site['homePageId'] ?? 0);
}

echo json_encode([
    'ok' => true,
    'pages' => $pages,
    'homePageId' => $homePageId,
], JSON_UNESCAPED_UNICODE);


---

Что увидишь после правки

у опубликованных страниц зелёный PUBLISHED

у черновиков оранжевый DRAFT

у домашней страницы бейдж HOME

draft-страницы будут слегка подсвечены фоном

дерево станет заметно читаемее


Следующим шагом я бы добавил туда же быстрые кнопки Draft / Published прямо в списке страниц.