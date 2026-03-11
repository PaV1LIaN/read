Отлично.

Следующий сильный шаг — сделать **человекочитаемые публичные ссылки** вместо:

```text
/local/sitebuilder/public.php?siteId=5&p=about
```

сделать, например:

```text
/sb/laboratoriya/about/
```

## Почему это сейчас лучший следующий шаг

У тебя уже есть:

* draft/published,
* public.php,
* home page,
* top menu,
* редиректы и 404.

То есть публичка уже почти готова, и теперь логично довести её до нормальных URL.

## Что будем делать

В несколько шагов:

### 1. Поддержка открытия сайта по `site slug`

Чтобы можно было искать сайт не только по `siteId`, но и по `site.slug`.

### 2. Поддержка открытия страницы по `page slug`

Чтобы `public.php` мог работать как роутер.

### 3. Новый формат URL

Например:

* `/local/sitebuilder/public.php?site=laboratoriya`
* `/local/sitebuilder/public.php?site=laboratoriya&page=about`

А потом уже при желании можно вынести это в rewrite.

### 4. Rewrite через `.htaccess` / Bitrix routing

Финальный красивый вариант:

* `/sb/laboratoriya/`
* `/sb/laboratoriya/about/`

---

## Я бы сейчас сделал именно промежуточный этап

Без сложного rewrite, но уже удобно:

* `public.php?site=slug`
* `public.php?site=slug&page=slug`

Это быстро, безопасно и удобно тестировать.

Если идём этим путём, следующим сообщением я пришлю тебе **готовый `public.php`**, который умеет:

* работать и по `siteId`,
* и по `site slug`,
* и по `page slug`,
* с приоритетом новых URL.
