Смотрю `settings.php` — **ошибка у тебя ровно в одном месте**:

### ✅ Проблема №1 (из-за неё “site.get: нет доступа”)

Твой `api()` **НЕ передаёт `siteId`** автоматически.

А дальше ты вызываешь:

```js
api('site.get')
api('page.list')
api('menu.list')
api('file.list')
```

Но все эти экшены в `api.php` ждут `$_POST['siteId']`.
В итоге на сервер уходит `siteId=0` ⇒ сервер отвечает **422 SITE_ID_REQUIRED**, а ты показываешь это как “нет доступа”.

---

## 1) Исправь `api()` в settings.php

Найди в `settings.php` функцию `api()` и замени строку `data: ...` на вариант **с `siteId`**:

```js
data: Object.assign({ action, siteId, sessid: BX.bitrix_sessid() }, data || {}),
```

То есть целиком:

```js
function api(action, data) {
  return new Promise((resolve) => {
    BX.ajax({
      url: '/local/sitebuilder/api.php',
      method: 'POST',
      dataType: 'json',
      data: Object.assign({ action, siteId, sessid: BX.bitrix_sessid() }, data || {}),
      onsuccess: (res) => resolve(res),
      onfailure: (xhr) => {
        const status = xhr && xhr.status ? xhr.status : 0;
        const raw = (xhr && xhr.responseText) ? String(xhr.responseText) : '';
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

После этого `site.get/page.list/menu.list/file.list` начнут работать нормально.

---

## 2) Добавь защиту на пустой siteId (желательно)

Сразу после:

```php
$siteId = (int)($_GET['siteId'] ?? 0);
```

добавь:

```php
if ($siteId <= 0) {
  http_response_code(400);
  echo 'siteId required';
  exit;
}
```

---

# Теперь про `site.update:HTTP_ERROR`

После фикса `api()` у тебя `site.update` станет “честным”:

* если в `api.php` **нет обработчика `site.update`** → будет `UNKNOWN_ACTION`
* если обработчик есть, но **права недостаточны** → будет `403 FORBIDDEN` (роль)
* если не передавался `siteId` → было `422 SITE_ID_REQUIRED` (это и было у тебя)

✅ Сначала сделай правку `api()` — это ключевое.

---

## Чтобы я сказал точно, почему `site.update` падает:

скинь кусок из **api.php**, где у тебя сейчас есть/нет:

* `if ($action === 'site.update') { ... }`
* и блок с `sb_require_*` (роль/ранги)

И я дам точный патч: куда вставить и что именно должно сохраняться (settings/name/slug/home/topMenu/logo) и какие проверки сделать.
