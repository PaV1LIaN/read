
   function fillSettingsForm() {
    
    document.getElementById('showHeader').checked = !!layoutSettings.showHeader;
    document.getElementById('showFooter').checked = !!layoutSettings.showFooter;
    document.getElementById('showLeft').checked = !!layoutSettings.showLeft;
    document.getElementById('leftMode').value = layoutSettings.leftMode || 'blocks';
    document.getElementById('showRight').checked = !!layoutSettings.showRight;
    document.getElementById('leftWidth').value = parseInt(layoutSettings.leftWidth || 260, 10);
    document.getElementById('rightWidth').value = parseInt(layoutSettings.rightWidth || 260, 10);
  }


 async function saveLayoutSettings() {
    try {
      const res = await api('layout.updateSettings', {
        siteId,
        showHeader: document.getElementById('showHeader').checked ? '1' : '0',
        showFooter: document.getElementById('showFooter').checked ? '1' : '0',
        showLeft: document.getElementById('showLeft').checked ? '1' : '0',
        showRight: document.getElementById('showRight').checked ? '1' : '0',
        leftWidth: parseInt(document.getElementById('leftWidth').value || '260', 10),
        rightWidth: parseInt(document.getElementById('rightWidth').value || '260', 10),
        leftMode: document.getElementById('leftMode').value || 'blocks',
      });
      if (!res || res.ok !== true) {
        notify('Не удалось сохранить настройки');
        return;
      }
      notify('Настройки layout сохранены');
    } catch (e) {
      notify('Ошибка layout.updateSettings');
    }
  }


if ($action === 'layout.updateSettings') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_admin($siteId);

    $sites = sb_read_sites();
    $found = false;

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) !== $siteId) continue;

        $layout = is_array($s['layout'] ?? null) ? $s['layout'] : [];

        if (array_key_exists('showHeader', $_POST)) {
            $layout['showHeader'] = in_array((string)$_POST['showHeader'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('showFooter', $_POST)) {
            $layout['showFooter'] = in_array((string)$_POST['showFooter'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('showLeft', $_POST)) {
            $layout['showLeft'] = in_array((string)$_POST['showLeft'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('showRight', $_POST)) {
            $layout['showRight'] = in_array((string)$_POST['showRight'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('leftWidth', $_POST)) {
            $w = (int)$_POST['leftWidth'];
            if ($w < 160) $w = 160;
            if ($w > 500) $w = 500;
            $layout['leftWidth'] = $w;
        }
        if (array_key_exists('rightWidth', $_POST)) {
            $w = (int)$_POST['rightWidth'];
            if ($w < 160) $w = 160;
            if ($w > 500) $w = 500;
            $layout['rightWidth'] = $w;
        }

        if (array_key_exists('leftMode', $_POST)) {
            $leftMode = trim((string)$_POST['leftMode']);
            if (!in_array($leftMode, ['blocks', 'menu'], true)) {
                $leftMode = 'blocks';
            }
            $layout['leftMode'] = $leftMode;
        }

        $layout += [
            'showHeader' => true,
            'showFooter' => true,
            'showLeft' => false,
            'showRight' => false,
            'leftWidth' => 260,
            'rightWidth' => 260,
            'leftMode' => 'blocks',
        ];

        $s['layout'] = $layout;
        $s['updatedAt'] = date('c');
        $s['updatedBy'] = (int)$USER->GetID();
        $found = true;
        break;
    }
    unset($s);

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_sites($sites);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
