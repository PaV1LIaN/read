`site.update:HTTP_ERROR` означает одно из двух:

1. **`api.php` вернул НЕ 200** (например 400/403/422/500) → `BX.ajax` уходит в `onfailure`, и ты видишь `HTTP_ERROR`
2. **вернулся ответ, который не распарсился как JSON** (например PHP warning/notice до `json_encode`)

Чтобы понять причину за 30 секунд — сделай так, чтобы `api()` **вытаскивал HTTP-код и тело ответа** даже при ошибке.

## 1) Быстрый фикс: замени `api()` в `settings.php` (и можно потом везде)

Найди в `settings.php` функцию `api(action, data)` и замени на эту:

```js
function api(action, data) {
  return new Promise((resolve) => {
    BX.ajax({
      url: '/local/sitebuilder/api.php',
      method: 'POST',
      dataType: 'json',
      data: Object.assign({ action, sessid: BX.bitrix_sessid() }, data || {}),
      onsuccess: (res) => resolve(res),
      onfailure: (xhr) => {
        // BX.ajax считает любой non-200 ошибкой и не даёт JSON в onsuccess
        const status = xhr && xhr.status ? xhr.status : 0;
        const raw = (xhr && xhr.responseText) ? String(xhr.responseText) : '';

        // попробуем вытащить JSON из responseText
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

## 2) И обработчик `btnSave.onclick` сделай таким

```js
btnSave.onclick = async () => {
  const r = await api('site.update', payload);

  if (!r || r.ok !== true) {
    const msg =
      'site.update failed' +
      (r?.status ? ` (HTTP ${r.status})` : '') +
      `: ${r?.error || 'UNKNOWN'}` +
      (r?.message ? ` — ${r.message}` : '') +
      (r?.raw ? `\n\n${String(r.raw).slice(0, 500)}` : '');

    notify(msg.replace(/\n/g, '<br>')); // если notify умеет html, иначе убери replace
    console.log('site.update debug:', r);
    return;
  }

  notify('Сохранено');
};
```

После этого ты увидишь **реальную** причину: например

* `HTTP 400: UNKNOWN_ACTION` → в `api.php` нет `site.update` (или не туда вставлен)
* `HTTP 403: BAD_SESSID / FORBIDDEN` → сессия/права
* `HTTP 422: ...` → валидация данных
* `HTTP 500: ...` → PHP ошибка (и в `raw` будет текст/trace)

## 3) Самая частая причина именно у вас

Если `site.update` у тебя заканчивается `http_response_code(400)` / `422` и JSONом — **BX.ajax всё равно считает это failure** → и ты видишь `HTTP_ERROR`.

То есть проблема может быть даже не “сломано”, а просто **сервер вернул 4xx**, а фронт не умел его показать.

---

Сделай замену `api()` + `btnSave.onclick` как выше и скинь сюда **новое сообщение ошибки**, которое появится (особенно `HTTP xxx` и `error/message/raw` кусок). Тогда я скажу точечно: это не хватает `site.update`, не те поля в payload, права, валидация или PHP фатал.
