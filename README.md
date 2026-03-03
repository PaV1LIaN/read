Ок, клонирование пропускаем. Тогда для “серьёзного проекта” самое полезное прямо сейчас (и не мешает текущей логике) — **журнал действий (audit log)**. Он реально спасает, когда много людей и непонятно “кто сломал меню/страницу/блок”.

## Что сделаем дальше: Audit Log (история изменений)

**Цель:** фиксировать в файл/JSON каждое важное действие: создание/удаление сайта, страницы, блоков, меню, смена верхнего меню, права доступа, загрузка/удаление файлов.

### Почему это must-have

* расследование “почему пропало меню/страница”
* откат руками (видишь что меняли)
* дисциплина при командной работе

---

# План (коротко)

1. **api.php**: добавить функции `sb_log()` и запись в `audit.json`
2. Вызывать `sb_log()` в ключевых action’ах (site/page/block/menu/access/file)
3. **Новая страница** `/local/sitebuilder/audit.php?siteId=...` — просмотр с фильтром (позже, но можно сразу)

---

## 1) api.php — добавить JSON-хранилище audit.json

Рядом с другими `sb_read_*` добавь:

```php
function sb_read_audit(): array { return sb_read_json_file('audit.json'); }
function sb_write_audit(array $audit): void { sb_write_json_file('audit.json', $audit, 'Cannot open audit.json'); }

function sb_audit_add(array $row): void {
    $audit = sb_read_audit();
    $audit[] = $row;

    // ограничим размер, чтобы файл не рос бесконечно (например 5000 записей)
    $max = 5000;
    if (count($audit) > $max) {
        $audit = array_slice($audit, -$max);
    }
    sb_write_audit($audit);
}

function sb_log(string $event, array $ctx = []): void {
    try {
        sb_audit_add([
            'ts' => date('c'),
            'userId' => (int)$GLOBALS['USER']->GetID(),
            'login' => (string)$GLOBALS['USER']->GetLogin(),
            'ip' => (string)($_SERVER['REMOTE_ADDR'] ?? ''),
            'event' => $event,
            'ctx' => $ctx,
        ]);
    } catch (\Throwable $e) {
        // лог не должен ломать работу API
    }
}
```

---

## 2) Куда вставлять вызовы `sb_log()` (минимальный набор)

Прямо перед `echo json_encode(['ok'=>true ...])` в экшенах:

* `site.create` → `sb_log('site.create', ['siteId'=>$id,'name'=>$name,'slug'=>$slug]);`
* `site.delete` → `sb_log('site.delete', ['siteId'=>$id]);`
* `page.create` → `sb_log('page.create', ['siteId'=>$siteId,'pageId'=>$id,'title'=>$title,'slug'=>$slug]);`
* `page.delete` → `sb_log('page.delete', ['siteId'=>$siteId,'pageId'=>$id]);`
* `page.setParent` → `sb_log('page.setParent', ['siteId'=>$siteId,'pageId'=>$id,'parentId'=>$parentId]);`
* `block.create/update/delete/move`
* `menu.create/update/item.add/item.delete/item.move`
* `menu.setTop` (у тебя уже есть) → `sb_log('menu.setTop', ['siteId'=>$siteId,'menuId'=>$menuId]);`
* `access.set/delete`
* `file.upload/delete`

Это 10–15 строк добавок, но даёт огромную пользу.

---

## 3) Просмотр (UI) — можно сделать быстро

Добавим страницу `audit.php`, которая:

* показывает последние 200 записей
* фильтр по `siteId` (если есть в ctx)
* поиск по `event/login`

Если хочешь — я сразу пришлю готовый `audit.php` (как делали `menu.php`/`index.php`) **без изменения архитектуры**.

---

### Альтернатива, если не хочешь лог прямо сейчас

Второй по полезности следующий шаг: **корзина удаления (soft-delete)** для страниц/блоков (не удалять навсегда, а помечать `deletedAt`, и сделать “восстановить”). Тоже очень “серьёзно” и уменьшает риск потерь.

---

Скажи что выбираем дальше:

1. **Audit log** (рекомендую первым)
2. **Корзина (soft-delete)** для страниц/блоков
3. **Импорт/экспорт JSON** (перенос между сайтами)

Я начну сразу с выбранного варианта и дам конкретные вставки “куда и что”.
