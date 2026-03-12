Да, теперь понятно, в чём логика бага.

Сейчас ветка открывается, потому что она active path, но это состояние не сохраняется в localStorage, если ты её не открывал кнопкой-стрелкой вручную.

Из-за этого происходит так:

зашёл в Вложенная 5 → ветка Вложенная 3 открылась автоматически

потом кликнул в Вложенная 3 → ветка Вложенная 5 уже не active path и, так как её открытие не было сохранено, она схлопнулась

то же самое выше по дереву


Как исправить

Нужно, чтобы:

если ветка открылась как defaultOpen,

это сразу записывалось в localStorage.


Тогда открытые ветки не будут закрываться просто из-за перехода на соседнюю страницу.


---

Что поменять

Найди внизу public.php блок <script>

И в нём найди этот кусок:

let isOpen;

// Активная ветка всегда открыта
if (defaultOpen) {
  isOpen = true;
} else if (Object.prototype.hasOwnProperty.call(state, id)) {
  isOpen = !!state[id];
} else {
  isOpen = false;
}

Замени его на это:

let isOpen;

if (Object.prototype.hasOwnProperty.call(state, id)) {
  isOpen = !!state[id];
} else if (defaultOpen) {
  isOpen = true;
  state[id] = true;
  writeState(state);
} else {
  isOpen = false;
}


---

Полный правильный JS блок

Чтобы не искать по кускам, вот весь скрипт целиком — можешь просто заменить весь <script> внизу на него:

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

    if (Object.prototype.hasOwnProperty.call(state, id)) {
      isOpen = !!state[id];
    } else if (defaultOpen) {
      isOpen = true;
      state[id] = true;
      writeState(state);
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

Почему это решает проблему

Теперь:

как только ты открыл страницу глубоко во вложенности,

все родительские ветки, которые открылись автоматически,

сразу сохраняются как open=true.


Значит при переходе:

Вложенная 5 → Вложенная 3

Вложенная 3 → Вложенная первая


ветки уже не будут схлопываться сами по себе.


---

Если после этого всё ещё будет странно

Тогда следующий шаг — сделать режим:

active path всегда open

вручную открытые ветки тоже open

вручную закрытые ветки можно закрывать даже если они были открыты раньше


Но обычно правки выше уже хватает.