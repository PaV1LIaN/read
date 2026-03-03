Понял. В твоём `settings.php` **“Ошибка site.update” появляется именно из `catch`**, то есть запрос ушёл, но вернулся **не-200 статус** (403/404/422/500 и т.п.).
А `BX.ajax` в таком случае вызывает `onfailure`, и промис **reject**, поэтому ты видишь только общую надпись без причины.

Для серьёзного проекта делаем правильно: **показываем точную ошибку сервера** и сразу понимаем, что чинить.

---

## 1) Быстрый фикс: чтобы settings показывал реальную причину (а не “Ошибка site.update”)

В `settings.php` замени функцию `api()` на эту (она **пытается распарсить JSON даже при onfailure** и возвращает `{ok:false, error, ...}`):

```js
function api(action, data) {
  return new Promise((resolve, reject) => {
    BX.ajax({
      url: '/local/sitebuilder/api.php',
      method: 'POST',
      dataType: 'json',
      data: Object.assign({ action, siteId, sessid: BX.bitrix_sessid() }, data || {}),
      onsuccess: resolve,
      onfailure: function (xhr) {
        // Важно: на ошибочных HTTP кодах Bitrix кидает сюда
        try {
          const txt =
            (xhr && (xhr.responseText || xhr.response || xhr.data)) ? (xhr.responseText || xhr.response || xhr.data) : '';
          const parsed = txt ? JSON.parse(txt) : null;
          if (parsed && typeof parsed === 'object') {
            resolve(parsed); // <-- возвращаем ошибку как обычный ответ
            return;
          }
        } catch (e) {}

        // fallback если не удалось распарсить
        resolve({ ok: false, error: 'HTTP_ERROR', status: xhr?.status || 0 });
      }
    });
  });
}
```

И в обработчике сохранения (`btnSave.onclick`) измени на:

```js
const r = await api('site.update', payload);
if (!r || r.ok !== true) {
  notify('site.update: ' + (r?.error || 'UNKNOWN') + (r?.message ? (' — ' + r.message) : ''));
  return;
}
notify('Сохранено');
```

✅ После этого при сохранении ты увидишь **точный `error`**, например: `UNKNOWN_ACTION`, `FORBIDDEN`, `ACCENT_BAD_FORMAT`, `LOGO_NOT_IN_SITE_FOLDER` и т.д.

---

## 2) Самые частые причины и что делать (по коду ошибки)

### `UNKNOWN_ACTION`

➡️ Ты не вставил `if ($action === 'site.update') ...` в `api.php` **или вставил после финального `UNKNOWN_ACTION`**.

**Проверь:** блок `site.update` должен быть **выше** самого конца:

```php
http_response_code(400);
echo json_encode(['ok'=>false,'error'=>'UNKNOWN_ACTION', ...]);
```

---

### `FORBIDDEN`

➡️ `site.update` у тебя стоит на `sb_require_admin($siteId)` (ADMIN/OWNER), а у пользователя роль ниже.

**Решение:**

* либо выдай себе ADMIN/OWNER в доступах,
* либо временно (на время отладки) поменяй в `site.update` на `sb_require_editor($siteId)`.

---

### `ACCENT_BAD_FORMAT`

➡️ В `accent` приходит не `#RRGGBB`.

**Решение:** в UI вводи строго `#2563eb` и т.п. (6 hex). Цветовой input обычно ок, а руками можно сломать.

---

### `LOGO_NOT_IN_SITE_FOLDER`

➡️ Выбранный файл не лежит в папке сайта в Disk.

**Решение:**

* выбери логотип из списка, который загрузил в “Файлы” именно этого сайта,
* либо поставь “— Без логотипа —”.

---

### `TOP_MENU_NOT_FOUND`

➡️ `topMenuId` пытаешься сохранить, но такого меню нет у сайта.

**Решение:**

* сначала создай меню в `menu.php`,
* потом выбери его в settings.

---

## 3) Ещё одна частая причина: забыли `sb_require_admin()`

Если у тебя в `api.php` нет:

```php
function sb_require_admin(int $siteId): void { sb_require_site_role($siteId, 3); }
```

то будет **500** (фатал), и ты ловишь “Ошибка site.update”.

---

## Что делаем прямо сейчас

1. Вставь обновлённый `api()` в `settings.php` (пункт 1).
2. Нажми “Сохранить” ещё раз.
3. Скажи мне, **какой именно `site.update: ERROR_CODE` показало уведомление** — и я дам точечную правку в `api.php` (1–2 строки), без гаданий.
