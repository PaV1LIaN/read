Да, понял проблему.

Это значит, что сейчас логика открытия веток работает по одному общему набору id, и при переходе/клике одна ветка перерисовывает состояние другой.
Обычно это происходит, когда:

id узлов недостаточно уникально обрабатывается в JS,

или при новой странице defaultOpen снова вмешивается,

или состояние веток хранится, но потом частично перезаписывается.


Как исправить правильно

Нужно сделать так:

состояние хранить по id конкретного узла

активную ветку открывать всегда

остальные ветки брать только из localStorage

при клике на один пункт не трогать другие



---

Замени JS внизу public.php целиком

Найди весь блок:

<script>
...
</script>

который отвечает за sideTreeNode, и замени полностью на это:

<script>
(function(){
  const siteKey = document.body.getAttribute('data-site-key') || location.pathname;
  const storageKey = 'sb-left-tree:' + siteKey;

  function readState() {
    try {
      const raw = localStorage.getItem(storageKey);
      const parsed = raw ? JSON.parse(raw) : {};
      return parsed && typeof parsed === 'object' ? parsed : {};
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
    const id = String(node.getAttribute('data-node-id') || '');
    const defaultOpen = node.getAttribute('data-open-default') === '1';

    const btn = node.querySelector(':scope > .sideMenuRow .sideToggle');
    const icon = node.querySelector(':scope > .sideMenuRow .sideToggle .sideToggleIcon');
    const children = node.querySelector(':scope > .sideTreeChildren');

    if (!id || !btn || !icon || !children) return;

    let isOpen;

    // Активная ветка всегда открыта
    if (defaultOpen) {
      isOpen = true;
    } else if (Object.prototype.hasOwnProperty.call(state, id)) {
      isOpen = !!state[id];
    } else {
      isOpen = false;
    }

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

И проверь body

Должно быть так:

<body
  data-site-key="<?=h((string)($site['slug'] ?? ('site-'.$siteId)))?>"
  style="--left-col: <?=h($leftCol)?>; --right-col: <?=h($rightCol)?>;"
>


---

Почему это исправляет баг

Теперь логика такая:

если пункт находится в активной цепочке страницы, он всегда открыт

если не в активной цепочке, его состояние берётся только из localStorage

клик по одному узлу меняет только его id

соседние ветки не должны схлопываться



---

Если после этого всё равно будет схлопываться

Тогда проблема уже не в JS, а в том, что в PHP дереве у разных узлов могут оказаться пересекающиеся data-node-id или структура рендерится не так, как ожидается.
Тогда я попрошу у тебя кусок HTML одного sidebar из браузера и сразу укажу точку.

Сейчас сначала поставь этот JS.