Объясню просто.

Что значит 3 пункт

В addSectionBlock() мало просто добавить <select id="sec_preset">....

Нужно ещё сделать так, чтобы:

ты выбрал Hero

и поля формы сами поменялись

background

paddingTop

paddingBottom

border

radius

boxed



То есть пункт 3 — это подключить поведение пресета.


---

Что именно делать

Было

У тебя, скорее всего, сейчас функция выглядит примерно так:

function addSectionBlock() {
  BX.UI.Dialogs.MessageBox.show({
    title: 'Новая Section',
    message: `...`,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function (mb) {
      const boxed = document.getElementById('sec_boxed')?.checked ? '1' : '0';
      const background = (document.getElementById('sec_bg')?.value || '#FFFFFF').trim();
      const paddingTop = parseInt(document.getElementById('sec_pt')?.value || '32', 10);
      const paddingBottom = parseInt(document.getElementById('sec_pb')?.value || '32', 10);
      const border = document.getElementById('sec_border')?.checked ? '1' : '0';
      const radius = parseInt(document.getElementById('sec_radius')?.value || '0', 10);

      api('block.create', {
        pageId,
        type: 'section',
        boxed,
        background,
        paddingTop,
        paddingBottom,
        border,
        radius
      })
      .then(...)
    }
  });
}


---

Нужно сделать

Надо заменить BX.UI.Dialogs.MessageBox.show({...}) на вариант, где результат show(...) сохраняется в переменную, а потом мы вешаем обработчик на select.

Вот полностью готовая addSectionBlock()

Просто замени всю свою функцию addSectionBlock() целиком на это:

function addSectionBlock() {
  const mb = BX.UI.Dialogs.MessageBox.show({
    title: 'Новая Section',
    message: `
      <div>
        <div class="field">
          <label>Пресет</label>
          <select id="sec_preset" class="input">
            ${sectionPresetOptions('default')}
          </select>
        </div>

        <div class="field">
          <label><input id="sec_boxed" type="checkbox" checked> Ограничить по контейнеру</label>
        </div>

        <div class="field">
          <label>Цвет фона</label>
          <input id="sec_bg" class="input" value="#FFFFFF" placeholder="#FFFFFF" />
        </div>

        <div class="field">
          <label>Отступ сверху (0..200)</label>
          <input id="sec_pt" class="input" type="number" min="0" max="200" value="32" />
        </div>

        <div class="field">
          <label>Отступ снизу (0..200)</label>
          <input id="sec_pb" class="input" type="number" min="0" max="200" value="32" />
        </div>

        <div class="field">
          <label><input id="sec_border" type="checkbox"> Показать рамку</label>
        </div>

        <div class="field">
          <label>Скругление (0..40)</label>
          <input id="sec_radius" class="input" type="number" min="0" max="40" value="0" />
        </div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function (mbox) {
      const boxed = document.getElementById('sec_boxed')?.checked ? '1' : '0';
      const background = (document.getElementById('sec_bg')?.value || '#FFFFFF').trim();
      const paddingTop = parseInt(document.getElementById('sec_pt')?.value || '32', 10);
      const paddingBottom = parseInt(document.getElementById('sec_pb')?.value || '32', 10);
      const border = document.getElementById('sec_border')?.checked ? '1' : '0';
      const radius = parseInt(document.getElementById('sec_radius')?.value || '0', 10);

      api('block.create', {
        pageId,
        type: 'section',
        boxed,
        background,
        paddingTop,
        paddingBottom,
        border,
        radius
      })
        .then(res => {
          if (!res || res.ok !== true) {
            notify('Не удалось создать section');
            return;
          }
          notify('Section создана');
          mbox.close();
          loadBlocks();
        })
        .catch(() => notify('Ошибка block.create (section)'));
    }
  });

  setTimeout(() => {
    const presetEl = document.getElementById('sec_preset');
    if (presetEl) {
      presetEl.addEventListener('change', () => {
        applySectionPresetToForm(presetEl.value);
      });

      applySectionPresetToForm('default');
    }
  }, 0);
}


---

Что делает setTimeout(...)

Вот именно это и есть “добавить применение пресета”.

Этот кусок:

setTimeout(() => {
  const presetEl = document.getElementById('sec_preset');
  if (presetEl) {
    presetEl.addEventListener('change', () => {
      applySectionPresetToForm(presetEl.value);
    });

    applySectionPresetToForm('default');
  }
}, 0);

делает 2 вещи:

1. Когда выбрал пресет

Например Hero, вызывается:

applySectionPresetToForm('hero');

и поля меняются автоматически.

2. Когда окно только открылось

Сразу вызывается:

applySectionPresetToForm('default');

чтобы форма сразу была заполнена корректным дефолтом.


---

Итого

С 3 пунктом тебе не нужно ничего “внутрь onOk” добавлять.

Нужно:

заменить всю addSectionBlock() на готовую версию выше

и всё



---

Если хочешь, следующим сообщением я так же целиком дам готовую editSectionBlock().








Да. Делаем точечно в editor.php.

Нужно добавить 3 вещи:

объект пресетов

select пресета в окно создания/редактирования

автоподстановку значений при выборе пресета



---

1. Добавь объект пресетов

Куда вставить

В editor.php, в <script> рядом с другими общими функциями, до addSectionBlock() и editSectionBlock() вставь:

const SECTION_PRESETS = {
  default: {
    boxed: true,
    background: '#FFFFFF',
    paddingTop: 32,
    paddingBottom: 32,
    border: false,
    radius: 0
  },
  hero: {
    boxed: true,
    background: '#F8FAFC',
    paddingTop: 72,
    paddingBottom: 72,
    border: false,
    radius: 0
  },
  light: {
    boxed: true,
    background: '#F9FAFB',
    paddingTop: 40,
    paddingBottom: 40,
    border: false,
    radius: 0
  },
  accent: {
    boxed: false,
    background: '#EEF2FF',
    paddingTop: 56,
    paddingBottom: 56,
    border: false,
    radius: 0
  },
  card: {
    boxed: true,
    background: '#FFFFFF',
    paddingTop: 32,
    paddingBottom: 32,
    border: true,
    radius: 16
  }
};

function sectionPresetOptions(selected = 'default') {
  return `
    <option value="default" ${selected === 'default' ? 'selected' : ''}>Default</option>
    <option value="hero" ${selected === 'hero' ? 'selected' : ''}>Hero</option>
    <option value="light" ${selected === 'light' ? 'selected' : ''}>Light</option>
    <option value="accent" ${selected === 'accent' ? 'selected' : ''}>Accent</option>
    <option value="card" ${selected === 'card' ? 'selected' : ''}>Card</option>
  `;
}

function applySectionPresetToForm(presetKey, suffix = '') {
  const preset = SECTION_PRESETS[presetKey] || SECTION_PRESETS.default;

  const boxedEl = document.getElementById('sec_boxed' + suffix);
  const bgEl = document.getElementById('sec_bg' + suffix);
  const ptEl = document.getElementById('sec_pt' + suffix);
  const pbEl = document.getElementById('sec_pb' + suffix);
  const borderEl = document.getElementById('sec_border' + suffix);
  const radiusEl = document.getElementById('sec_radius' + suffix);

  if (boxedEl) boxedEl.checked = !!preset.boxed;
  if (bgEl) bgEl.value = preset.background;
  if (ptEl) ptEl.value = preset.paddingTop;
  if (pbEl) pbEl.value = preset.paddingBottom;
  if (borderEl) borderEl.checked = !!preset.border;
  if (radiusEl) radiusEl.value = preset.radius;
}


---

2. Обнови addSectionBlock()

Куда смотреть

Найди функцию:

function addSectionBlock() {

Внутри неё, в message: \...`` добавь поле пресета самым первым.

Было примерно так:

<div>
  <div class="field">
    <label><input id="sec_boxed" type="checkbox" checked> Boxed контейнер</label>
  </div>

Должно стать так:

message: `
  <div>
    <div class="field">
      <label>Пресет</label>
      <select id="sec_preset" class="input">
        ${sectionPresetOptions('default')}
      </select>
    </div>

    <div class="field">
      <label><input id="sec_boxed" type="checkbox" checked> Ограничить по контейнеру</label>
    </div>

    <div class="field">
      <label>Цвет фона</label>
      <input id="sec_bg" class="input" value="#FFFFFF" placeholder="#FFFFFF" />
    </div>

    <div class="field">
      <label>Отступ сверху (0..200)</label>
      <input id="sec_pt" class="input" type="number" min="0" max="200" value="32" />
    </div>

    <div class="field">
      <label>Отступ снизу (0..200)</label>
      <input id="sec_pb" class="input" type="number" min="0" max="200" value="32" />
    </div>

    <div class="field">
      <label><input id="sec_border" type="checkbox"> Показать рамку</label>
    </div>

    <div class="field">
      <label>Скругление (0..40)</label>
      <input id="sec_radius" class="input" type="number" min="0" max="40" value="0" />
    </div>
  </div>
`,


---

3. В addSectionBlock() добавь применение пресета

В той же функции, перед api('block.create', ...) в onOk ничего особенного не нужно.

Но нужно повесить change на select после открытия окна.

Сразу после BX.UI.Dialogs.MessageBox.show({ ... });

если функция у тебя просто вызывает show(...), замени на такой шаблон:

const mb = BX.UI.Dialogs.MessageBox.show({
  title: 'Новая Section',
  message: `...`,
  buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
  onOk: function (mbox) {
    const boxed = document.getElementById('sec_boxed')?.checked ? '1' : '0';
    const background = (document.getElementById('sec_bg')?.value || '#FFFFFF').trim();
    const paddingTop = parseInt(document.getElementById('sec_pt')?.value || '32', 10);
    const paddingBottom = parseInt(document.getElementById('sec_pb')?.value || '32', 10);
    const border = document.getElementById('sec_border')?.checked ? '1' : '0';
    const radius = parseInt(document.getElementById('sec_radius')?.value || '0', 10);

    api('block.create', {
      pageId,
      type: 'section',
      boxed,
      background,
      paddingTop,
      paddingBottom,
      border,
      radius
    })
      .then(res => {
        if (!res || res.ok !== true) {
          notify('Не удалось создать section');
          return;
        }
        notify('Section создана');
        mbox.close();
        loadBlocks();
      })
      .catch(() => notify('Ошибка block.create (section)'));
  }
});

setTimeout(() => {
  const presetEl = document.getElementById('sec_preset');
  if (presetEl) {
    presetEl.addEventListener('change', () => {
      applySectionPresetToForm(presetEl.value);
    });
    applySectionPresetToForm('default');
  }
}, 0);


---

4. Обнови editSectionBlock(id)

Куда смотреть

Найди функцию:

function editSectionBlock(id) {

Внутри message: \...`` добавь select пресета сверху.

Вставь в начало формы:

<div class="field">
  <label>Пресет</label>
  <select id="sec_preset_e" class="input">
    ${sectionPresetOptions('default')}
  </select>
</div>


---

5. Обнови подписи полей в редактировании

Сделай так же, как в создании:

Boxed контейнер → Ограничить по контейнеру

Фон секции → Цвет фона

Граница → Показать рамку



---

6. Повесь обработчик на пресет в editSectionBlock

После BX.UI.Dialogs.MessageBox.show({...}) добавь:

setTimeout(() => {
  const presetEl = document.getElementById('sec_preset_e');
  if (presetEl) {
    presetEl.addEventListener('change', () => {
      applySectionPresetToForm(presetEl.value, '_e');
    });
  }
}, 0);


---

7. Что получится

Теперь:

в создании section будет выбор пресета

в редактировании тоже

выбор пресета будет автоматически подставлять:

фон

boxed

отступы

border

radius




---

Что проверить

1. Нажми + Section


2. Посмотри, появился ли Пресет


3. Переключи Hero, Accent, Card


4. Убедись, что поля меняются автоматически


5. Создай секцию


6. Нажми Редактировать у секции и проверь, что пресет-селект там тоже есть



Если хочешь, следующим сообщением я могу дать тебе готовую целиком функцию addSectionBlock() и целиком editSectionBlock(), чтобы ты не собирал по кускам.