<?php
require_once($_SERVER['DOCUMENT_ROOT'].'/bitrix/header.php');
global $USER;

\Bitrix\Main\Page\Asset::getInstance()->addCss('/local/glab/assets/lab_files.css');

require_once($_SERVER['DOCUMENT_ROOT'].'/local/php_interface/lib/pg_master.php');
require_once($_SERVER['DOCUMENT_ROOT'].'/local/php_interface/lib/lab_disk.php');

use Bitrix\Main\Loader;
use Bitrix\Disk\File;

function h($s){ return htmlspecialcharsbx((string)$s); }

CJSCore::Init(['ui.viewer']); // важно для data-viewer

function statusBadge(string $status): string {
    $st = strtoupper(trim($status));
    $map = [
        'NEW'         => ['badge--new',      'Новая'],
        'IN_PROGRESS' => ['badge--progress', 'В работе'],
        'DONE'        => ['badge--done',     'Готово'],
        'REJECTED'    => ['badge--reject',   'Отклонено'],
    ];
    if (isset($map[$st])) {
        [$cls, $label] = $map[$st];
        return '<span class="badge '.$cls.'" title="'.h($label).'">'.h($label).'</span>';
    }
    return '<span class="badge badge--other" title="'.h($status ?: '—').'">'.h($status ?: '—').'</span>';
}

$appId = isset($_GET['id']) ? (int)$_GET['id'] : 0;
if ($appId <= 0) { ShowError('Нет id заявки'); require_once($_SERVER['DOCUMENT_ROOT'].'/bitrix/footer.php'); exit; }

try {
    $pdo = getPdo();

    $st = $pdo->prepare("SELECT * FROM lab.user_applications WHERE id=:id LIMIT 1");
    $st->execute([':id'=>$appId]);
    $app = $st->fetch(PDO::FETCH_ASSOC);
    if (!$app) throw new RuntimeException('Заявка не найдена');

    if (strtoupper((string)$app['status']) === 'NEW') {
        throw new RuntimeException('Папка доступна после перевода заявки в IN_PROGRESS');
    }

    if (!Loader::includeModule('disk')) {
        throw new RuntimeException('Модуль disk не установлен');
    }

    $appFolder = labEnsureDiskFolderForApp($pdo, $app, (int)$USER->GetID());

    if ($_SERVER['REQUEST_METHOD']==='POST' && check_bitrix_sessid()) {
        $action = (string)($_POST['action'] ?? '');

        if ($action === 'upload' && isset($_FILES['files'])) {
            foreach ($_FILES['files']['name'] as $i => $origName) {
                if ($_FILES['files']['error'][$i] !== UPLOAD_ERR_OK) continue;

                $fileArray = [
                    'name' => $_FILES['files']['name'][$i],
                    'type' => $_FILES['files']['type'][$i] ?? '',
                    'tmp_name' => $_FILES['files']['tmp_name'][$i],
                    'error' => $_FILES['files']['error'][$i],
                    'size' => $_FILES['files']['size'][$i] ?? 0,
                ];

                $diskFile = $appFolder->uploadFile(
                    $fileArray,
                    ['CREATED_BY' => (int)$USER->GetID()],
                    [],
                    true
                );

                if ($diskFile) {
                    $ins = $pdo->prepare("
                        INSERT INTO lab.application_files (application_id, disk_file_id, original_name, uploaded_by)
                        VALUES (:app, :fid, :n, :by)
                    ");
                    $ins->execute([
                        ':app' => $appId,
                        ':fid' => (int)$diskFile->getId(),
                        ':n'   => (string)$origName,
                        ':by'  => (int)$USER->GetID(),
                    ]);
                }
            }
            LocalRedirect('files.php?id='.$appId);
        }

        if ($action === 'rename') {
            $fileId = (int)($_POST['file_id'] ?? 0);
            $newName = trim((string)($_POST['new_name'] ?? ''));

            if ($fileId > 0 && $newName !== '') {
                $file = File::loadById($fileId);
                if ($file) {
                    $file->rename($newName, (int)$USER->GetID());
                    $pdo->prepare("
                        UPDATE lab.application_files
                        SET original_name=:n
                        WHERE application_id=:app AND disk_file_id=:fid
                    ")->execute([':n'=>$newName, ':app'=>$appId, ':fid'=>$fileId]);
                }
            }
            LocalRedirect('files.php?id='.$appId);
        }

        if ($action === 'replace' && isset($_FILES['replace_file'])) {
            $fileId = (int)($_POST['file_id'] ?? 0);

            if ($fileId > 0 && $_FILES['replace_file']['error'] === UPLOAD_ERR_OK) {
                $file = File::loadById($fileId);
                if ($file) {
                    $fileArray = [
                        'name' => $_FILES['replace_file']['name'],
                        'type' => $_FILES['replace_file']['type'] ?? '',
                        'tmp_name' => $_FILES['replace_file']['tmp_name'],
                        'error' => $_FILES['replace_file']['error'],
                        'size' => $_FILES['replace_file']['size'] ?? 0,
                    ];

                    $file->uploadVersion($fileArray, (int)$USER->GetID());

                    $pdo->prepare("
                        UPDATE lab.application_files
                        SET original_name=:n, uploaded_by=:by, uploaded_at=now()
                        WHERE application_id=:app AND disk_file_id=:fid
                    ")->execute([
                        ':n' => (string)($_FILES['replace_file']['name'] ?? ''),
                        ':by' => (int)$USER->GetID(),
                        ':app' => $appId,
                        ':fid' => $fileId
                    ]);
                }
            }
            LocalRedirect('files.php?id='.$appId);
        }

        if ($action === 'delete') {
            $fileId = (int)($_POST['file_id'] ?? 0);
            if ($fileId > 0) {
                $file = File::loadById($fileId);
                if ($file) $file->delete((int)$USER->GetID());

                $pdo->prepare("
                    DELETE FROM lab.application_files
                    WHERE application_id=:app AND disk_file_id=:fid
                ")->execute([':app'=>$appId, ':fid'=>$fileId]);
            }
            LocalRedirect('files.php?id='.$appId);
        }
    }

    $lst = $pdo->prepare("
        SELECT disk_file_id, original_name, uploaded_at
        FROM lab.application_files
        WHERE application_id=:id
        ORDER BY uploaded_at DESC
    ");
    $lst->execute([':id'=>$appId]);
    $rows = $lst->fetchAll(PDO::FETCH_ASSOC);

} catch (Throwable $e) {
    ShowError(h($e->getMessage()));
    require_once($_SERVER['DOCUMENT_ROOT'].'/bitrix/footer.php');
    exit;
}

$cnt = is_array($rows) ? count($rows) : 0;

$svgEdit = '<svg viewBox="0 0 24 24" fill="none" aria-hidden="true"><path d="M12 20h9" stroke="currentColor" stroke-width="2" stroke-linecap="round"/><path d="M16.5 3.5a2.12 2.12 0 0 1 3 3L7 19l-4 1 1-4 12.5-12.5Z" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/></svg>';
$svgSwap = '<svg viewBox="0 0 24 24" fill="none" aria-hidden="true"><path d="M12 3v10" stroke="currentColor" stroke-width="2" stroke-linecap="round"/><path d="M8 9l4 4 4-4" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/><path d="M4 21h16" stroke="currentColor" stroke-width="2" stroke-linecap="round"/></svg>';
$svgTrash = '<svg viewBox="0 0 24 24" fill="none" aria-hidden="true"><path d="M3 6h18" stroke="currentColor" stroke-width="2" stroke-linecap="round"/><path d="M8 6V4h8v2" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"/><path d="M19 6l-1 14H6L5 6" stroke="currentColor" stroke-width="2" stroke-linejoin="round"/></svg>';
?>

<div class="lab-wrap">
  <div class="lab-card">
    <div class="lab-head">
      <div class="lab-head-top">
        <div>
          <p class="lab-title">Файлы заявки №<?= (int)$appId ?></p>
          <div class="lab-meta">
            <?= statusBadge((string)($app['status'] ?? '')) ?>
            <span class="chip">project_<?= (int)$app['block_id'] ?></span>
            <span class="chip"><?= h((string)($app['work_department'] ?? '—')) ?></span>
          </div>
        </div>
        <div class="chip">Файлов: <b><?= (int)$cnt ?></b></div>
      </div>
    </div>

    <div class="lab-body">

      <form method="POST" enctype="multipart/form-data" class="upload">
        <?= bitrix_sessid_post() ?>
        <input type="hidden" name="action" value="upload">
        <input class="file" type="file" name="files[]" multiple>
        <button class="btn btn--primary" type="submit">Загрузить</button>
      </form>

      <?php if ($cnt === 0): ?>
        <div class="empty">Файлы пока не загружены.</div>
      <?php else: ?>

        <div class="table-wrap">
          <table class="lab-table">
            <colgroup>
              <col class="c-name">
              <col class="c-view">
              <col class="c-date">
              <col class="c-actions">
            </colgroup>

            <thead>
              <tr>
                <th>Файл</th>
                <th>Просмотр</th>
                <th>Загружен</th>
                <th>Действия</th>
              </tr>
            </thead>

            <tbody>
            <?php foreach ($rows as $r):
              $fid  = (int)$r['disk_file_id'];
              $name = (string)$r['original_name'];
              $f    = File::loadById($fid);

              $src = '/disk/downloadFile/'.$fid.'/?&ncc=1&filename='.rawurlencode($name);

              $uploadedAt = (string)($r['uploaded_at'] ?? '');
              $uploadedAt = $uploadedAt ? date('d.m.Y H:i', strtotime($uploadedAt)) : '—';

              $actions = htmlspecialcharsbx('[{"type":"download"},{"type":"edit","action":"BX.Disk.Viewer.Actions.runActionDefaultEdit","params":{"objectId":"'.$fid.'","name":"'.htmlspecialcharsbx($name).'"}}]');
            ?>
              <tr>
                <td title="<?= h($name) ?>"><?= h($name) ?></td>

                <td>
                  <div class="links">
                    <?php if ($f): ?>
                      <span
                        class="viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show"
                        data-viewer=""
                        data-viewer-type="cloud-document"
                        data-src="<?= h($src) ?>"
                        data-viewer-type-class="BX.Disk.Viewer.DocumentItem"
                        data-viewer-extension="disk.viewer.document-item"
                        data-object-id="<?= (int)$fid ?>"
                        data-title="<?= h($name) ?>"
                        data-actions="<?= $actions ?>"
                      >Просмотреть</span>
                    <?php else: ?>
                      <span class="muted">(не найден)</span>
                    <?php endif; ?>

                    <a href="<?= h($src) ?>">Скачать</a>
                  </div>
                </td>

                <td><?= h($uploadedAt) ?></td>

                <td class="actions-cell">
  <div class="act-row">

    <!-- Rename: mini popup -->
    <details class="mini">
      <summary class="iconbtn-mini iconbtn-mini--edit" title="Переименовать">
        <?= $svgEdit ?>
      </summary>
      <div class="mini-panel">
        <p class="mini-title">Переименовать</p>
        <form method="POST" class="mini-form">
          <?= bitrix_sessid_post() ?>
          <input type="hidden" name="action" value="rename">
          <input type="hidden" name="file_id" value="<?= $fid ?>">
          <input class="input" type="text" name="new_name" value="<?= h($name) ?>">
          <button class="btn-mini" type="submit">ОК</button>
        </form>
      </div>
    </details>

    <!-- Replace: mini popup -->
    <details class="mini">
      <summary class="iconbtn-mini iconbtn-mini--replace" title="Заменить (новая версия)">
        <?= $svgSwap ?>
      </summary>
      <div class="mini-panel">
        <p class="mini-title">Заменить (новая версия)</p>
        <form method="POST" enctype="multipart/form-data" class="mini-form">
          <?= bitrix_sessid_post() ?>
          <input type="hidden" name="action" value="replace">
          <input type="hidden" name="file_id" value="<?= $fid ?>">
          <input class="file" type="file" name="replace_file" required>
          <button class="btn-mini btn-mini--primary" type="submit">Заменить</button>
        </form>
      </div>
    </details>

    <!-- Delete: one-click icon -->
    <form method="POST" onsubmit="return confirm('Удалить файл?');" style="display:inline;">
      <?= bitrix_sessid_post() ?>
      <input type="hidden" name="action" value="delete">
      <input type="hidden" name="file_id" value="<?= $fid ?>">
      <button class="iconbtn-mini iconbtn-mini--danger" type="submit" title="Удалить">
        <?= $svgTrash ?>
      </button>
    </form>

  </div>
</td>
              </tr>
            <?php endforeach; ?>
            </tbody>

          </table>
        </div>

      <?php endif; ?>

    </div>
  </div>
</div>

<script>
(function(){
  function placePanel(details){
    const summary = details.querySelector('summary');
    const panel = details.querySelector('.mini-panel');
    if (!summary || !panel) return;

    // сначала показываем, чтобы получить реальные размеры
    panel.style.left = '0px';
    panel.style.top = '0px';
    panel.style.visibility = 'hidden';
    panel.style.display = 'block';

    const btn = summary.getBoundingClientRect();
    const p = panel.getBoundingClientRect();

    const gap = 8;
    const vw = window.innerWidth;
    const vh = window.innerHeight;

    // x: стараемся выровнять по правому краю кнопки
    let left = btn.right - p.width;
    left = Math.max(12, Math.min(left, vw - p.width - 12));

    // y: вниз, если помещается; иначе вверх
    let topDown = btn.bottom + gap;
    let topUp = btn.top - p.height - gap;

    let top = topDown;
    if (topDown + p.height > vh - 12) {
      top = topUp;
    }
    top = Math.max(12, Math.min(top, vh - p.height - 12));

    panel.style.left = left + 'px';
    panel.style.top = top + 'px';
    panel.style.visibility = 'visible';
  }

  // закрывать другие меню при открытии одного
  document.addEventListener('toggle', function(e){
    const d = e.target;
    if (!(d instanceof HTMLElement)) return;
    if (!d.matches('details.mini')) return;

    if (d.open) {
      document.querySelectorAll('details.mini[open]').forEach(function(other){
        if (other !== d) other.open = false;
      });
      placePanel(d);
    }
  }, true);

  // при ресайзе/скролле пересчитываем позицию открытого
  function repositionOpen(){
    const open = document.querySelector('details.mini[open]');
    if (open) placePanel(open);
  }
  window.addEventListener('resize', repositionOpen);
  window.addEventListener('scroll', repositionOpen, true);

  // клик вне панели закрывает
  document.addEventListener('mousedown', function(e){
    const open = document.querySelector('details.mini[open]');
    if (!open) return;

    const panel = open.querySelector('.mini-panel');
    const summary = open.querySelector('summary');
    if (!panel || !summary) return;

    if (!panel.contains(e.target) && !summary.contains(e.target)) {
      open.open = false;
    }
  });
})();
</script>

<?php require_once($_SERVER['DOCUMENT_ROOT'].'/bitrix/footer.php'); ?>
