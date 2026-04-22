`NOT_AUTHORIZED` сверху означает, что **внутрь `SitebuilderDiskComponent` в параметр `CURRENT_USER_ID` пришёл `0`**.

В этом месте ошибка возникает вот здесь, в `class.php`:

```php
'currentUserId' => (int)($this->params['CURRENT_USER_ID'] ?? 0),
```

А дальше `DiskValidator::assertContext()` режет это как `NOT_AUTHORIZED`.

---

# Что проверить в `test.php`

Сделай так, чтобы `CURRENT_USER_ID` брался не из старой переменной, а напрямую через helper.

## Замени `test.php` на такой вариант

```php
<?php
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/class.php';

$currentUserId = DiskCurrentUser::requireId();

$siteId = 1;
$pageId = 1;
$blockId = 1;

echo '<pre>';
echo 'USER_ID=' . $currentUserId . PHP_EOL;
echo 'ROLE=';
var_dump(SiteAccessRepository::getUserRole($siteId, $currentUserId));
echo 'BLOCK=';
var_dump(BlockRepository::getDiskBlockByContext($siteId, $pageId, $blockId));
echo '</pre>';
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Тест компонента Disk</title>
    <link rel="stylesheet" href="/local/sitebuilder/components/disk/styles.css">
</head>
<body style="margin:0; padding:24px; background:#f5f7fb;">
    <div style="max-width:1200px; margin:0 auto;">
        <?php
        $component = new SitebuilderDiskComponent([
            'SITE_ID' => $siteId,
            'PAGE_ID' => $pageId,
            'BLOCK_ID' => $blockId,
            'CURRENT_USER_ID' => $currentUserId,
        ]);
        $component->execute();
        ?>
    </div>

    <script src="/bitrix/js/main/core/core.js"></script>
    <script src="/local/sitebuilder/components/disk/script.js"></script>
</body>
</html>
```

---

# Если ошибка останется

Тогда нужно проверить, что реально уходит в компонент.

В `class.php` временно вставь прямо перед `DiskValidator::assertContext($context);`:

```php
echo '<pre>';
var_dump($this->params);
var_dump($context);
echo '</pre>';
```

Если там будет:

```php
CURRENT_USER_ID => 0
```

значит проблема точно в `test.php`.

---

# Почему так могло случиться

Самые вероятные причины:

* в `test.php` использовалась старая переменная `$currentUserId`, которая в момент вызова была `0`
* или `global $USER` был использован криво, а helper `DiskCurrentUser::requireId()` даст уже правильный ID
* или ты менял `test.php`, но запускал старую версию файла

---

# Что должно быть после правки

Сверху должно быть:

* `USER_ID=1`
* `ROLE=site_admin`

и ошибка `NOT_AUTHORIZED` должна исчезнуть.

Если после этого появится новая ошибка — это уже следующий реальный шаг диагностики.
