Да, вместо кнопки «Меню» в editor.php добавь кнопку «Настройки».

В editor.php найди блок верхних кнопок, где сейчас примерно так:

<div class="sb-editor-topline-actions">
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/public.php?siteId=<?= (int)$siteId ?>" target="_blank">Открыть публичную</a>
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/layout.php?siteId=<?= (int)$siteId ?>">Layout</a>
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
</div>

Замени на:

<div class="sb-editor-topline-actions">
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/public.php?siteId=<?= (int)$siteId ?>" target="_blank">Открыть публичную</a>

    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/layout.php?siteId=<?= (int)$siteId ?>">Layout</a>

    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/settings.php?siteId=<?= (int)$siteId ?>">Настройки</a>
</div>

Если кнопку menu.php ты уже закомментировал, то просто вместо неё вставь:

<a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/settings.php?siteId=<?= (int)$siteId ?>">Настройки</a>

Так будет:

Открыть публичную | Layout | Настройки

И menu.php больше не будет светиться в интерфейсе.