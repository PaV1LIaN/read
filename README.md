Теперь уже точно по твоему файлу.

У тебя сейчас в `index.php` **одновременно включены два варианта управления статусом**, поэтому ты и видишь странный набор кнопок.

## Что именно у тебя лишнее

В `renderPagesTree()` → `renderNode()` сейчас есть:

### 1. Нормальная кнопка-переключатель

Она правильная, её **оставляем**:

```js
${(status === 'published')
  ? `<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">В черновик</button>`
  : `<button class="ui-btn ui-btn-success ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Опубликовать</button>`}
```

### 2. Лишние кнопки

Вот эти **нужно удалить**:

```js
<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">Draft</button>
<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Published</button>
```

---

## Ещё у тебя дублируется сам статус в UI

Сейчас статус показывается:

* как `statusBadge` рядом со slug,
* и ещё раз как `.nodeStatus` внутри `.nodeMeta`.

### Оставить лучше только один вариант

Самый красивый у тебя уже есть — это `statusBadge`:

```js
const statusBadge = (status === 'published')
  ? '<span class="nodeBadge nodeBadgePub">PUBLISHED</span>'
  : '<span class="nodeBadge nodeBadgeDraft">DRAFT</span>';
```

### А вот этот второй дубль лучше удалить:

```js
<span class="nodeStatus ${String(node.status || 'published') === 'draft' ? 'isDraft' : 'isPublished'}">
  ${BX.util.htmlspecialchars(String(node.status || 'published').toUpperCase())}
</span>
```

---

# Что сделать прямо сейчас

## Было

```js
<div class="nodeMeta">
  <span>sort: ${parseInt(node.sort||500,10)}</span>
  <span>${parentLabel}</span>
  <span class="nodeStatus ${String(node.status || 'published') === 'draft' ? 'isDraft' : 'isPublished'}">
    ${BX.util.htmlspecialchars(String(node.status || 'published').toUpperCase())}
  </span>
</div>
```

## Должно стать

```js
<div class="nodeMeta">
  <span>sort: ${parseInt(node.sort||500,10)}</span>
  <span>${parentLabel}</span>
</div>
```

---

## Было

```js
${(status === 'published')
  ? `<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">В черновик</button>`
  : `<button class="ui-btn ui-btn-success ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Опубликовать</button>`}

<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>
<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-duplicate="${node.id}">Дублировать</button>

<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">Draft</button>
<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Published</button>
```

## Должно стать

```js
${(status === 'published')
  ? `<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="draft">В черновик</button>`
  : `<button class="ui-btn ui-btn-success ui-btn-xs btnTiny" data-page-status="${node.id}" data-status="published">Опубликовать</button>`}

<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>
<button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-duplicate="${node.id}">Дублировать</button>
```

---

## Итог после правки

У страницы будет:

* один бейдж `DRAFT` / `PUBLISHED`,
* одна кнопка действия:

  * `Опубликовать`, если draft
  * `В черновик`, если published

И это уже будет правильно.

Если хочешь, следующим сообщением я могу прислать тебе **готовый цельный кусок `renderNode()`**, который ты просто вставишь вместо текущего.
