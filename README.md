Да, сделаем так:

1. в настройках появится поле “Размер логотипа”;


2. размер будет сохраняться в settings_json;


3. на публичной части логотип будет менять размер;


4. длинное название сайта будет обрезаться ..., а не переноситься под логотип.




---

1. SiteAppearanceService.php

Файл:

/local/sitebuilder/lib/SiteAppearanceService.php

1.1. В методе update() добавь обработку logoSize

Найди внутри update():

if (array_key_exists('headerLogoMode', $data)) {
    $settings['headerLogoMode'] = self::normalizeHeaderLogoMode((string)$data['headerLogoMode']);
}

Сразу после него добавь:

if (array_key_exists('logoSize', $data)) {
    $settings['logoSize'] = self::normalizeLogoSize((int)$data['logoSize']);
}


---

1.2. В normalizeAppearanceSettings() добавь logoSize

Найди:

'headerLogoMode' => self::normalizeHeaderLogoMode(
    (string)($settings['headerLogoMode'] ?? 'image')
),

Замени на:

'headerLogoMode' => self::normalizeHeaderLogoMode(
    (string)($settings['headerLogoMode'] ?? 'image')
),

'logoSize' => self::normalizeLogoSize(
    (int)($settings['logoSize'] ?? 42)
),


---

1.3. В конец класса перед последней } добавь метод

protected static function normalizeLogoSize(int $size): int
{
    if ($size < 24) {
        return 24;
    }

    if ($size > 160) {
        return 160;
    }

    return $size;
}


---

2. settings.php

Файл:

/local/sitebuilder/settings.php

2.1. Добавь поле размера логотипа

Найди блок:

<div class="sb-field">
    <label for="headerLogoModeInput">Отображение в шапке</label>
    <select class="sb-select" id="headerLogoModeInput">
        <option value="image">Только логотип</option>
        <option value="text">Только название сайта</option>
        <option value="both">Логотип и название</option>
    </select>
</div>

Сразу после него добавь:

<div class="sb-field" style="margin-top:12px;">
    <label for="logoSizeInput">Размер логотипа, px</label>
    <input class="sb-input" type="number" id="logoSizeInput" min="24" max="160" step="2" value="42">
</div>


---

2.2. В renderAppearance() добавь установку значения

Найди:

setValue('headerLogoModeInput', appearance.headerLogoMode || 'image');

Сразу после добавь:

setValue('logoSizeInput', appearance.logoSize || 42);


---

2.3. В saveAppearance() добавь отправку logoSize

Найди:

headerLogoMode: getValue('headerLogoModeInput') || 'image'

Замени на:

headerLogoMode: getValue('headerLogoModeInput') || 'image',
logoSize: getValue('logoSizeInput') || '42'


---

2.4. В renderMainPreview() добавь размер логотипа

Найди внутри renderMainPreview():

preview.style.setProperty('--preview-accent', accent);

Сразу после добавь:

preview.style.setProperty('--preview-logo-size', (getValue('logoSizeInput') || appearance.logoSize || 42) + 'px');


---

2.5. Добавь logoSizeInput в live-preview

Найди массив:

[
    'siteNameInput',
    'accentInput',
    'backgroundColorInput',
    'backgroundModeInput',
    'backgroundPositionInput',
    'backgroundRepeatInput'
]

Замени на:

[
    'siteNameInput',
    'accentInput',
    'backgroundColorInput',
    'backgroundModeInput',
    'backgroundPositionInput',
    'backgroundRepeatInput',
    'logoSizeInput',
    'headerLogoModeInput'
]


---

3. settings.css

Файл:

/local/sitebuilder/assets/admin/settings.css

В конец добавь:

/* Размер логотипа в предпросмотре */
.sb-appearance-preview__logo {
    width: var(--preview-logo-size, 38px);
    height: var(--preview-logo-size, 38px);
    flex: 0 0 var(--preview-logo-size, 38px);
}

.sb-appearance-preview__header {
    min-width: 0;
}

.sb-appearance-preview__title {
    min-width: 0;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}


---

4. public_page.php

Файл:

/local/sitebuilder/views/layout/public_page.php

4.1. В sb_public_appearance_get() добавь logoSize

Найди в return [:

'headerLogoMode' => $headerLogoMode,

Замени на:

'headerLogoMode' => $headerLogoMode,

'logoSize' => max(24, min(160, (int)($settings['logoSize'] ?? 42))),


---

4.2. В sb_public_appearance_style() добавь CSS-переменную

Найди:

$styles[] = '--sb-accent: ' . sb_public_h((string)($appearance['accent'] ?? '#2563eb'));

Сразу после добавь:

$styles[] = '--sb-logo-size: ' . max(24, min(160, (int)($appearance['logoSize'] ?? 42))) . 'px';


---

5. public.css

Файл:

/local/sitebuilder/assets/public/public.css

В конец добавь:

/* =========================================================
   BRAND / LOGO SIZE / LONG SITE NAME FIX
   ========================================================= */

.sb-public-header .sb-container {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 18px;
    min-width: 0;
}

.sb-brand {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    min-width: 0;
    max-width: 55%;
    flex: 1 1 auto;
    flex-wrap: nowrap;
}

.sb-brand__logo {
    width: var(--sb-logo-size, 42px);
    height: var(--sb-logo-size, 42px);
    min-width: var(--sb-logo-size, 42px);
    max-width: var(--sb-logo-size, 42px);
    min-height: var(--sb-logo-size, 42px);
    max-height: var(--sb-logo-size, 42px);
    flex: 0 0 var(--sb-logo-size, 42px);
}

.sb-brand__text {
    display: block;
    min-width: 0;
    max-width: 100%;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.sb-public-menu {
    flex: 0 1 auto;
    min-width: 0;
}

@media (max-width: 760px) {
    .sb-public-header .sb-container {
        align-items: flex-start;
        flex-direction: column;
    }

    .sb-brand {
        max-width: 100%;
        width: 100%;
    }

    .sb-public-menu {
        width: 100%;
    }
}


---

6. Обнови версию CSS

В public_page.php найди:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=2">

Замени на:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=3">

И в settings.php можно тоже обновить:

/assets/admin/settings.css?v=2

После этого:

1. открой settings.php;


2. выстави размер логотипа, например 56;


3. нажми “Сохранить настройки фона”;


4. обнови публичную страницу через Ctrl + F5.


