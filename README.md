Дальше логично сделать **нормальный редирект в `public.php`** и **красивый 404**, чтобы публичка выглядела уже как законченный модуль.

## Что делаем сейчас

1. Если открыли
   `/local/sitebuilder/public.php?siteId=5`
   без `p=slug`, то:

   * находим home page,
   * если home нет — первую опубликованную,
   * и делаем **redirect** на URL со slug.

2. Если страница не найдена или она draft — показываем **красивую HTML-страницу 404**, а не просто текст `Page not found`.

---

# Что поменять в `public.php`

## 1. Добавь функцию красивого 404

Вставь после `file_url(...)`:

```php id="x0s7f1"
function public_not_found(string $title = 'Страница не найдена', string $text = 'Запрошенная страница недоступна или не опубликована.'): void {
    http_response_code(404);
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
      <meta charset="utf-8" />
      <meta name="viewport" content="width=device-width, initial-scale=1" />
      <title><?= htmlspecialcharsbx($title) ?></title>
      <style>
        body{
          margin:0;
          font-family:Arial,sans-serif;
          background:#f8fafc;
          color:#111827;
          display:flex;
          align-items:center;
          justify-content:center;
          min-height:100vh;
          padding:24px;
          box-sizing:border-box;
        }
        .box{
          max-width:560px;
          width:100%;
          background:#fff;
          border:1px solid #e5e7eb;
          border-radius:20px;
          padding:28px;
          box-shadow:0 10px 30px rgba(0,0,0,.05);
        }
        h1{
          margin:0 0 10px;
          font-size:28px;
          line-height:1.2;
        }
        p{
          margin:0;
          color:#6b7280;
          line-height:1.7;
        }
        .actions{
          margin-top:18px;
          display:flex;
          gap:10px;
          flex-wrap:wrap;
        }
        a{
          display:inline-flex;
          align-items:center;
          justify-content:center;
          padding:10px 14px;
          border-radius:12px;
          border:1px solid #e5e7eb;
          text-decoration:none;
          color:#111827;
          background:#fff;
        }
        a.primary{
          background:#2563eb;
          border-color:#2563eb;
          color:#fff;
        }
      </style>
    </head>
    <body>
      <div class="box">
        <h1><?= htmlspecialcharsbx($title) ?></h1>
        <p><?= htmlspecialcharsbx($text) ?></p>
        <div class="actions">
          <a href="javascript:history.back()">Назад</a>
          <a class="primary" href="/">На главную</a>
        </div>
      </div>
    </body>
    </html>
    <?php
    exit;
}
```

---

## 2. Замени простые `echo 'Site not found'` и `echo 'Page not found'`

### Было:

```php id="34k3uh"
if ($siteId <= 0) {
    http_response_code(404);
    echo 'Site not found';
    exit;
}
```

### Стало:

```php id="h0mep2"
if ($siteId <= 0) {
    public_not_found('Сайт не найден', 'Не передан корректный siteId.');
}
```

---

### Было:

```php id="w3lt4j"
$site = find_site($siteId);
if (!$site) {
    http_response_code(404);
    echo 'Site not found';
    exit;
}
```

### Стало:

```php id="9gk6m0"
$site = find_site($siteId);
if (!$site) {
    public_not_found('Сайт не найден', 'Сайт с таким идентификатором не существует.');
}
```

---

### Было:

```php id="sqv4h6"
if (!$page) {
    http_response_code(404);
    echo 'Page not found';
    exit;
}
```

### Стало:

```php id="qq0a8w"
if (!$page) {
    public_not_found('Страница не найдена', 'Запрошенная страница отсутствует или не опубликована.');
}
```

И там, где slug задан, но страница не найдена, тоже замени аналогично:

```php id="shjdgb"
if ($slug !== '') {
    $page = find_page_by_slug($siteId, $slug);
    if (!$page) {
        public_not_found('Страница не найдена', 'Страница с таким адресом отсутствует или ещё не опубликована.');
    }
}
```

---

## 3. Добавь редирект на canonical URL со slug

После блока, где страница уже выбрана, то есть **сразу после**:

```php
$pageId = (int)($page['id'] ?? 0);
```

вставь:

```php id="v2wsk2"
if ($slug === '') {
    $target = public_page_url($siteId, $page);
    header('Location: ' . $target, true, 302);
    exit;
}
```

---

# Что получится

Теперь поведение будет таким:

* `/local/sitebuilder/public.php?siteId=5`
  → редиректит на
  `/local/sitebuilder/public.php?siteId=5&p=home-slug`

* если slug ведёт на draft или несуществующую страницу
  → красивая 404

* если `siteId` битый
  → красивая 404

---

# Как проверить

1. Открой:

   ```text
   /local/sitebuilder/public.php?siteId=ТВОЙ_ID
   ```

   Должен быть редирект на `&p=slug`

2. Открой:

   ```text
   /local/sitebuilder/public.php?siteId=ТВОЙ_ID&p=slug-draft-page
   ```

   Должна быть красивая 404

3. Открой несуществующий `siteId`
   Должна быть красивая 404

Следующим шагом после этого я бы сделал **человекочитаемые маршруты без `?siteId=&p=`**, например через `/sb/{siteSlug}/{pageSlug}/`.
