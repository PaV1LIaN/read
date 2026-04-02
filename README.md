Проверил новый архив и вижу важную вещь:

assets/js/editor.dialogs.js у тебя сейчас пустышка-шаблон Bitrix

assets/js/editor.init.js тоже пустышка-шаблон Bitrix


То есть их нельзя сейчас использовать как рабочие файлы — там просто:

<?
require($_SERVER["DOCUMENT_ROOT"]."/bitrix/header.php");
$APPLICATION->SetTitle("Title");
?>Text here....<?require($_SERVER["DOCUMENT_ROOT"]."/bitrix/footer.php");?>

Поэтому дальше безопаснее идти не “дописывать их по месту”, а заменить их нормальными файлами.

Что предлагаю сейчас

Не выносить пока весь огромный editor.dialogs.js, а сделать следующий безопасный шаг:

1. Оставить все add...Block() и edit...Block() пока в editor.php

2. Вынести только:

editor.core.js

editor.api.js

editor.sections.js

editor.dnd.js

editor.blocks.js


3. Следующим шагом сделать реальный editor.init.js

Чтобы:

он забрал DOM-элементы

положил их в SBEditor.state

вызывал loadBlocks()

подключал уже вынесенные куски


Это сейчас намного безопаснее, чем пытаться сразу выносить весь блок диалогов на сотни строк.


---

Что делать прямо сейчас

Удали/не используй пока эти 2 файла:

assets/js/editor.dialogs.js

assets/js/editor.init.js


или просто оставь, но не подключай их, пока они пустые.


---

Следующий правильный шаг

Я сейчас могу дать тебе готовый рабочий editor.init.js, который станет мостом между:

твоим editor.php

и уже вынесенными editor.core.js / editor.api.js / editor.sections.js / editor.dnd.js / editor.blocks.js


После этого можно будет аккуратно переносить оставшиеся куски.

Если идём так, следующим сообщением я дам тебе готовый editor.init.js.