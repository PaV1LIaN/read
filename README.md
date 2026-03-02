Ок — даю **точечный патч для твоего `view.php`**, чтобы поддержать `columns2` (и сделать нормально на мобилке).

---

# 1) CSS в `view.php`

В `<style>` добавь (лучше рядом с другими `.block-*` стилями):

```css
/* columns2 */
.cols2 {
  margin-top: 14px;
  display: grid;
  gap: 14px;
}
.cols2 .col {
  background: #fff;
  border: 1px solid #eee;
  border-radius: 12px;
  padding: 12px;
}
@media (max-width: 768px) {
  .cols2 { grid-template-columns: 1fr !important; }
}
```

> `grid-template-columns` мы зададим inline из PHP по ratio, а на мобилке принудительно делаем 1 колонку.

---

# 2) Рендер блока `columns2` в цикле блоков

В `view.php` внутри `foreach ($blocks as $b)` (там где у тебя уже `text`, `image`, `button`, `heading`) вставь **ещё один обработчик**:

```php
<?php if ($type === 'columns2'): ?>
  <?php
    $left  = (string)($b['content']['left'] ?? '');
    $right = (string)($b['content']['right'] ?? '');
    $ratio = (string)($b['content']['ratio'] ?? '50-50');

    if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

    $tpl = '1fr 1fr';
    if ($ratio === '33-67') $tpl = '1fr 2fr';
    if ($ratio === '67-33') $tpl = '2fr 1fr';
  ?>
  <div class="cols2" style="grid-template-columns: <?=h($tpl)?>;">
    <div class="col"><?= nl2br(h($left)) ?></div>
    <div class="col"><?= nl2br(h($right)) ?></div>
  </div>
<?php endif; ?>
```

---

# 3) Что проверить

1. В редакторе создай **Columns2** и заполни тексты
2. Открой `view.php` — должны быть 2 колонки
3. Поменяй ratio (50/50, 33/67, 67/33) — ширина должна меняться
4. Сожми окно браузера до мобилки — колонки должны стать **одной под другой**

---

Если всё ок, следующий шаг логичный: **Gallery block** (несколько картинок из “Файлы” сайта, 2/3/4 в ряд + адаптив).
