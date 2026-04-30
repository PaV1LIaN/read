Да, подключаем в публичную часть.

Сейчас нужно поправить 2 файла:

/local/sitebuilder/public_page.php
/local/sitebuilder/assets/public/public.css


---

1. public_page.php

1.1. Добавь функции оформления

В public_page.php после начальных переменных, где уже есть $site, $siteId, $basePath, добавь:

<?php

if (!function_exists('sb_public_get_file_url')) {
    function sb_public_get_file_url(int $fileId): string
    {
        if ($fileId <= 0) {
            return '';
        }

        if (!class_exists('CFile')) {
            return '';
        }

        return (string)CFile::GetPath($fileId);
    }
}

if (!function_exists('sb_public_normalize_background_size')) {
    function sb_public_normalize_background_size(string $mode): string
    {
        switch ($mode) {
            case 'contain':
                return 'contain';

            case 'auto':
                return 'auto';

            case 'stretch':
                return '100% 100%';

            case 'cover':
            default:
                return 'cover';
        }
    }
}

if (!function_exists('sb_public_get_appearance')) {
    function sb_public_get_appearance(array $site): array
    {
        $settings = is_array($site['settings'] ?? null) ? $site['settings'] : [];

        $logoFileId = (int)($settings['logoFileId'] ?? 0);
        $backgroundFileId = (int)($settings['backgroundFileId'] ?? 0);

        return [
            'accent' => (string)($settings['accent'] ?? '#2563eb'),

            'logoFileId' => $logoFileId,
            'logoUrl' => sb_public_get_file_url($logoFileId),

            'backgroundFileId' => $backgroundFileId,
            'backgroundUrl' => sb_public_get_file_url($backgroundFileId),

            'backgroundColor' => (string)($settings['backgroundColor'] ?? '#f8fafc'),
            'backgroundMode' => (string)($settings['backgroundMode'] ?? 'cover'),
            'backgroundPosition' => (string)($settings['backgroundPosition'] ?? 'center center'),
            'backgroundRepeat' => (string)($settings['backgroundRepeat'] ?? 'no-repeat'),

            'headerLogoMode' => (string)($settings['headerLogoMode'] ?? 'image'),
        ];
    }
}

if (!function_exists('sb_public_appearance_style')) {
    function sb_public_appearance_style(array $appearance): string
    {
        $styles = [];

        $accent = (string)($appearance['accent'] ?? '#2563eb');
        $backgroundColor = (string)($appearance['backgroundColor'] ?? '#f8fafc');
        $backgroundUrl = (string)($appearance['backgroundUrl'] ?? '');

        $styles[] = '--sb-accent: ' . $accent;
        $styles[] = 'background-color: ' . $backgroundColor;

        if ($backgroundUrl !== '') {
            $styles[] = 'background-image: url("' . htmlspecialchars($backgroundUrl, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '")';
            $styles[] = 'background-size: ' . sb_public_normalize_background_size((string)($appearance['backgroundMode'] ?? 'cover'));
            $styles[] = 'background-position: ' . (string)($appearance['backgroundPosition'] ?? 'center center');
            $styles[] = 'background-repeat: ' . (string)($appearance['backgroundRepeat'] ?? 'no-repeat');
        }

        return implode('; ', $styles);
    }
}

if (!function_exists('sb_public_render_brand')) {
    function sb_public_render_brand(array $site, array $appearance): string
    {
        $siteName = (string)($site['name'] ?? 'Сайт');
        $logoUrl = (string)($appearance['logoUrl'] ?? '');
        $mode = (string)($appearance['headerLogoMode'] ?? 'image');

        if (!in_array($mode, ['image', 'text', 'both'], true)) {
            $mode = 'image';
        }

        $html = '';

        if (($mode === 'image' || $mode === 'both') && $logoUrl !== '') {
            $html .= '<span class="sb-public-brand__logo">';
            $html .= '<img src="' . htmlspecialchars($logoUrl, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '" alt="' . sb_public_h($siteName) . '">';
            $html .= '</span>';
        }

        if ($mode === 'text' || $mode === 'both' || $logoUrl === '') {
            $html .= '<span class="sb-public-brand__text">' . sb_public_h($siteName) . '</span>';
        }

        return $html;
    }
}


---

1.2. Подготовь $appearance

Найди место, где уже доступны данные сайта, например:

$site = $vm['site'];
$siteId = (int)$vm['siteId'];

Сразу после этого добавь:

$appearance = sb_public_get_appearance($site);
$publicRootStyle = sb_public_appearance_style($appearance);


---

1.3. Подключи фон к главному wrapper

Найди главный wrapper публичной страницы. Скорее всего там что-то типа:

<div class="sb-public-root">

Замени на:

<div class="sb-public-root" style="<?= sb_public_h($publicRootStyle) ?>">

Если у тебя нет sb-public-root, а фон висит на body, лучше сделать так:

<body>
<div class="sb-public-root" style="<?= sb_public_h($publicRootStyle) ?>">

И закрывающий </div> должен быть перед </body>.


---

1.4. Подключи логотип в шапке

Найди в public_page.php старый вывод названия сайта в шапке. Он может выглядеть примерно так:

<a class="sb-public-brand" href="<?= sb_public_h($basePath . '/public.php?siteId=' . $siteId) ?>">
    <?= sb_public_h((string)($site['name'] ?? 'Сайт')) ?>
</a>

Замени на:

<a class="sb-public-brand" href="<?= sb_public_h($basePath . '/public.php?siteId=' . $siteId) ?>">
    <?= sb_public_render_brand($site, $appearance) ?>
</a>

Если в шапке сейчас не ссылка, а просто div, можно так:

<div class="sb-public-brand">
    <?= sb_public_render_brand($site, $appearance) ?>
</div>


---

2. public.css

В конец файла:

/local/sitebuilder/assets/public/public.css

добавь:

/* =========================================================
   PUBLIC APPEARANCE: logo / background
   ========================================================= */

.sb-public-root {
    min-height: 100vh;
    background-attachment: fixed;
}

.sb-public-header {
    background: rgba(255, 255, 255, .92);
    backdrop-filter: blur(10px);
}

.sb-public-brand {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    min-width: 0;
    color: #111827;
    text-decoration: none;
    font-weight: 800;
}

.sb-public-brand:hover {
    color: var(--sb-accent, #2563eb);
}

.sb-public-brand__logo {
    width: 42px;
    height: 42px;
    border-radius: 12px;
    overflow: hidden;
    background: #fff;
    border: 1px solid #e5e7eb;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    flex: 0 0 auto;
}

.sb-public-brand__logo img {
    max-width: 100%;
    max-height: 100%;
    object-fit: contain;
    display: block;
}

.sb-public-brand__text {
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-public-main,
.sb-public-page,
.sb-public-content {
    position: relative;
}

@media (max-width: 760px) {
    .sb-public-root {
        background-attachment: scroll;
    }

    .sb-public-brand__logo {
        width: 36px;
        height: 36px;
        border-radius: 10px;
    }
}


---

3. Если CSS кэшируется

В файле, где подключается public.css, временно поменяй версию:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/public/public.css?v=2">


---

После этого проверь:

1. Загрузил логотип в settings.php.
2. Открыл public.php?siteId=...
3. Логотип появился в шапке.
4. Загрузил фон.
5. Фон появился на публичной странице.
6. Поменял режим cover/contain/repeat.
7. Публичная часть изменилась после сохранения.