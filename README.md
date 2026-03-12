Идём дальше. Теперь делаем, чтобы section реально работал в public.php.

Смысл будет такой:

блок section открывает новую секцию

все следующие обычные блоки рендерятся внутри неё

следующая section закрывает предыдущую

если секций нет вообще, всё работает как раньше


Шаг 1. Добавь функции рендера секций

В public.php после функции:

function sb_render_blocks(array $blocks, int $siteId): string {

или сразу после неё вставь это:

function sb_section_style(array $content): string {
    $boxed = !empty($content['boxed']);
    $background = (string)($content['background'] ?? '#ffffff');
    if (!preg_match('~^#[0-9a-fA-F]{6}$~', $background)) $background = '#ffffff';

    $paddingTop = (int)($content['paddingTop'] ?? 32);
    $paddingBottom = (int)($content['paddingBottom'] ?? 32);
    if ($paddingTop < 0) $paddingTop = 0;
    if ($paddingTop > 200) $paddingTop = 200;
    if ($paddingBottom < 0) $paddingBottom = 0;
    if ($paddingBottom > 200) $paddingBottom = 200;

    $border = !empty($content['border']);
    $radius = (int)($content['radius'] ?? 0);
    if ($radius < 0) $radius = 0;
    if ($radius > 40) $radius = 40;

    $styles = [];
    $styles[] = 'background:' . $background;
    $styles[] = 'padding-top:' . $paddingTop . 'px';
    $styles[] = 'padding-bottom:' . $paddingBottom . 'px';

    if ($border) {
        $styles[] = 'border:1px solid #e5e7eb';
    }
    if ($radius > 0) {
        $styles[] = 'border-radius:' . $radius . 'px';
    }

    return implode(';', $styles);
}

function sb_render_page_with_sections(array $blocks, int $siteId): string {
    if (!$blocks) return '';

    $html = '';
    $currentSection = null;
    $buffer = [];

    $flush = function() use (&$html, &$currentSection, &$buffer, $siteId) {
        if ($currentSection === null) {
            if ($buffer) {
                $html .= sb_render_blocks($buffer, $siteId);
            }
            $buffer = [];
            return;
        }

        $content = is_array($currentSection['content'] ?? null) ? $currentSection['content'] : [];
        $boxed = !empty($content['boxed']);
        $style = sb_section_style($content);

        $html .= '<section class="sbSection" style="' . h($style) . '">';
        if ($boxed) {
            $html .= '<div class="sbSectionInner sbSectionInnerBoxed">';
        } else {
            $html .= '<div class="sbSectionInner sbSectionInnerFull">';
        }

        if ($buffer) {
            $html .= sb_render_blocks($buffer, $siteId);
        }

        $html .= '</div>';
        $html .= '</section>';

        $buffer = [];
    };

    foreach ($blocks as $b) {
        $type = (string)($b['type'] ?? '');

        if ($type === 'section') {
            $flush();
            $currentSection = $b;
            continue;
        }

        $buffer[] = $b;
    }

    $flush();

    return $html;
}


---

Шаг 2. Добавь стили секций

В <style> public.php добавь:

.sbSection{
  margin-top:18px;
}

.sbSectionInner{
  width:100%;
}

.sbSectionInnerBoxed{
  width:min(var(--sb-container), calc(100% - 32px));
  margin:0 auto;
}

.sbSectionInnerFull{
  width:100%;
  margin:0;
}

.sbSection .block:first-child{
  margin-top:0;
}


---

Шаг 3. Не показывай section как обычный блок

Сейчас в sb_render_block(...) секция не обрабатывается, и это хорошо.
Главное, чтобы она не попадала как обычный блок в общий рендер.


---

Шаг 4. Замени рендер страницы

Найди в public.php:

$pageHtml = sb_render_blocks($blocks, $siteId);

И замени на:

$pageHtml = sb_render_page_with_sections($blocks, $siteId);


---

Что получится

Теперь структура будет такой:

если на странице нет ни одного section
всё отображается как раньше

если есть section
она начинает новую секцию, и все следующие блоки идут внутрь неё



---

Как проверить

Создай блоки в таком порядке:

1. section


2. heading


3. text


4. button


5. section


6. cards



И у первой секции поставь, например:

boxed = 1

background = #F9FAFB

paddingTop = 40

paddingBottom = 40

border = 1

radius = 16


У второй:

boxed = 0

background = #EEF2FF


Тогда в публичке увидишь две разные смысловые области.


---

Следующий шаг после этого — добавить section в UI редактора блоков, чтобы его можно было удобно создавать и редактировать через интерфейс.