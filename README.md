Давай.

Сделаем так:

по умолчанию открыта активная ветка

у пунктов с детьми будет кнопка-стрелка

по клику ветка сворачивается/раскрывается

состояние запоминается в localStorage


1. Замени рендер левого дерева

Найди функцию:

function sb_render_left_menu_tree(array $nodes, array $site, int $currentPageId, array $activePathIds, int $level = 0): string

И замени её целиком на это:

function sb_render_left_menu_tree(array $nodes, array $site, int $currentPageId, array $activePathIds, int $level = 0): string {
    if (!$nodes) return '';

    $html = '<div class="sideTree level-' . $level . '">';
    foreach ($nodes as $node) {
        $id = (int)($node['id'] ?? 0);
        $title = (string)($node['title'] ?? 'Page');
        $isActive = ($id === $currentPageId);
        $isInPath = in_array($id, $activePathIds, true);
        $children = is_array($node['children'] ?? null) ? $node['children'] : [];
        $hasChildren = count($children) > 0;

        $classes = ['sideMenuLink'];
        if ($isActive) $classes[] = 'active';
        if ($hasChildren) $classes[] = 'hasChildren';
        if ($isInPath) $classes[] = 'open';

        $url = public_page_url($site, $node);

        $html .= '<div class="sideTreeNode level-' . $level . '" data-node-id="' . $id . '" data-open-default="' . ($isInPath ? '1' : '0') . '">';

        if ($hasChildren) {
            $html .= '<div class="sideMenuRow">';
            $html .= '<button type="button" class="sideToggle" aria-expanded="' . ($isInPath ? 'true' : 'false') . '" aria-label="Переключить ветку">';
            $html .= '<span class="sideToggleIcon">' . ($isInPath ? '▾' : '▸') . '</span>';
            $html .= '</button>';
            $html .= '<a class="' . h(implode(' ', $classes)) . '" href="' . h($url) . '">';
            $html .= '<span class="sideMenuText">' . h($title) . '</span>';
            $html .= '</a>';
            $html .= '</div>';

            $html .= '<div class="sideTreeChildren" style="display:' . ($isInPath ? 'block' : 'none') . ';">';
            $html .= sb_render_left_menu_tree($children, $site, $currentPageId, $activePathIds, $level + 1);
            $html .= '</div>';
        } else {
            $html .= '<div class="sideMenuRow">';
            $html .= '<span class="sideToggleStub"></span>';
            $html .= '<a class="' . h(implode(' ', $classes)) . '" href="' . h($url) . '">';
            $html .= '<span class="sideMenuText">' . h($title) . '</span>';
            $html .= '</a>';
            $html .= '</div>';
        }

        $html .= '</div>';
    }
    $html .= '</div>';

    return $html;
}


---

2. Добавь стили для toggle

В <style> добавь или обнови эти стили:

.sideMenuRow{
  display:flex;
  align-items:flex-start;
  gap:6px;
  min-width:0;
}

.sideToggle{
  width:18px;
  height:18px;
  flex:0 0 18px;
  margin-top:6px;
  border:0;
  background:transparent;
  color:#94a3b8;
  cursor:pointer;
  padding:0;
  display:inline-flex;
  align-items:center;
  justify-content:center;
  border-radius:6px;
}

.sideToggle:hover{
  background:#f1f5f9;
  color:#64748b;
}

.sideToggleIcon{
  font-size:10px;
  line-height:1;
}

.sideToggleStub{
  width:18px;
  height:18px;
  flex:0 0 18px;
  margin-top:6px;
}

.sideMenuLink{
  display:flex;
  align-items:flex-start;
  gap:8px;
  width:100%;
  padding:6px 8px;
  border-radius:8px;
  border:1px solid transparent;
  background:transparent;
  color:var(--text);
  text-decoration:none;
  line-height:1.35;
  transition:background .15s ease, color .15s ease, border-color .15s ease;
}

Если у тебя уже есть .sideMenuLink, оставь только одну версию — новую.


---

3. Добавь JS перед </body>

В самый низ public.php, прямо перед:

</body>

вставь:

<script>
(function(){
  const storageKey = 'sb-left-tree:' + location.pathname + location.search;

  function readState() {
    try {
      return JSON.parse(localStorage.getItem(storageKey) || '{}') || {};
    } catch (e) {
      return {};
    }
  }

  function writeState(state) {
    try {
      localStorage.setItem(storageKey, JSON.stringify(state));
    } catch (e) {}
  }

  const state = readState();

  document.querySelectorAll('.sideTreeNode[data-node-id]').forEach(node => {
    const id = node.getAttribute('data-node-id');
    const defaultOpen = node.getAttribute('data-open-default') === '1';
    const btn = node.querySelector(':scope > .sideMenuRow .sideToggle');
    const icon = node.querySelector(':scope > .sideMenuRow .sideToggle .sideToggleIcon');
    const children = node.querySelector(':scope > .sideTreeChildren');

    if (!btn || !children || !icon) return;

    let isOpen = Object.prototype.hasOwnProperty.call(state, id) ? !!state[id] : defaultOpen;

    function render() {
      children.style.display = isOpen ? 'block' : 'none';
      btn.setAttribute('aria-expanded', isOpen ? 'true' : 'false');
      icon.textContent = isOpen ? '▾' : '▸';
    }

    btn.addEventListener('click', function(e){
      e.preventDefault();
      e.stopPropagation();
      isOpen = !isOpen;
      state[id] = isOpen;
      writeState(state);
      render();
    });

    render();
  });
})();
</script>


---

4. Что получится

Теперь:

активная ветка открывается автоматически

любую ветку можно свернуть

состояние сохраняется после перехода



---

5. Что проверить

Открой страницу с вложенностью и нажимай на стрелки слева от пунктов, у которых есть дети.

Если хочешь, следующим сообщением я дам ещё мини-доработку: чтобы клик по названию родителя только переходил на страницу, а клик по стрелке только раскрывал ветку — сейчас это уже так, и это лучший вариант.