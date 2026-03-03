Ок, принимаю. Перестаю гадать и **делаем так, чтобы ты видел РЕАЛЬНУЮ причину**, а не “нет доступа” на всё подряд.

Сейчас у тебя в `settings.php` любой фейл (`SITE_ID_REQUIRED`, `FORBIDDEN`, 500, и т.д.) отображается одной фразой “нет доступа”, поэтому ты и я “ходим кругами”.

## 1) settings.php — исправь api() так, чтобы siteId всегда уходил + чтобы ты видел ошибку

**Замени целиком функцию `api(action, data)`** на эту:

```js
function api(action, data) {
  return new Promise((resolve) => {
    BX.ajax({
      url: '/local/sitebuilder/api.php',
      method: 'POST',
      dataType: 'json',
      data: Object.assign(
        { action, siteId, sessid: BX.bitrix_sessid() }, // <-- КРИТИЧНО: siteId всегда уходит
        data || {}
      ),
      onsuccess: (res) => resolve(res),
      onfailure: (xhr) => {
        const status = xhr?.status || 0;
        const raw = xhr?.responseText ? String(xhr.responseText) : '';
        let parsed = null;
        try { parsed = JSON.parse(raw); } catch (e) {}

        resolve(Object.assign(
          { ok: false, error: 'HTTP_ERROR', status, raw },
          (parsed && typeof parsed === 'object') ? parsed : {}
        ));
      }
    });
  });
}
```

## 2) settings.php — в load() покажи конкретную ошибку site.get (а не “нет доступа”)

Вот этот кусок:

```js
if (!sres || sres.ok !== true){ notify('site.get: нет доступа'); return; }
```

**замени на:**

```js
if (!sres || sres.ok !== true){
  console.log('site.get debug:', sres);
  notify(`site.get: ${sres.error || 'UNKNOWN'}${sres.status ? ' (HTTP '+sres.status+')' : ''}${sres.role ? ' role='+sres.role : ''}`);
  return;
}
```

После этого ты сразу увидишь, что именно происходит:

* `SITE_ID_REQUIRED` → значит siteId всё ещё не уходит (и это 100% фронт)
* `FORBIDDEN` + `role` → значит в `access.json` у твоего `U<id>` нет прав на этот сайт
* `HTTP_ERROR (500)` → значит на сервере фаталка/ошибка PHP

## 3) api.php — site.update сейчас может падать фатально

У тебя в `site.update` стоит:

```php
sb_require_admin($siteId);
```

Но **в твоём api.php нет функции `sb_require_admin()`** (в показанном куске её нет). Это даст **500**.

### Быстрый фикс (правильный)

Добавь рядом с остальными:

```php
function sb_require_admin(int $siteId): void { sb_require_site_role($siteId, 3); }
```

(OWNER=4, ADMIN=3, EDITOR=2, VIEWER=1)

---

# Почему у тебя “site.get: нет доступа” даже сейчас

Потому что `settings.php` **не показывает реальную ошибку**.
После правок из пунктов (1) и (2) ты увидишь точный код (`SITE_ID_REQUIRED` / `FORBIDDEN` / что-то ещё) — и это сразу снимет всю неопределённость.

---

## Мини-чек после правок (2 клика)

1. Открой `settings.php?siteId=...`
2. Если снова ошибка — **посмотри notify + консоль** (там будет `site.get debug:` с объектом)

Скинь сюда **ровно строку из notify** после этих правок (типа `site.get: SITE_ID_REQUIRED (HTTP 422)` или `site.get: FORBIDDEN role=null`) — и я дам точечный фикс без кругов.
