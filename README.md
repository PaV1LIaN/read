<?php if (!empty($arResult['ERROR'])): ?>
    <pre style="padding:12px;background:#fff3f3;border:1px solid #f1b5b5;color:#8a1f1f;margin-bottom:16px;">
<?= disk_h($arResult['ERROR']) ?>
    </pre>
<?php endif; ?>
