Да, это последняя часть для editor.dialogs.js.

Часть 8 из 8:

setTimeout(() => {
        const txt = document.getElementById('edit_btn_text');
        const url = document.getElementById('edit_btn_url');
        const variant = document.getElementById('edit_btn_variant');
        const prev = document.getElementById('edit_btn_preview');

        if (!txt || !url || !variant || !prev) return;

        const sync = () => {
          prev.textContent = txt.value || 'Кнопка';
          prev.setAttribute('href', url.value || '#');
          prev.className = btnClass(variant.value || 'primary');
        };

        txt.addEventListener('input', sync);
        url.addEventListener('input', sync);
        variant.addEventListener('change', sync);
        sync();
      }, 0);
    });
  }

  // ---- expose legacy names so current editor.php can still call them ----
  window.saveTemplateFromPage = window.SBEditor.saveTemplateFromPage;
  window.applyTemplateToPage = window.SBEditor.applyTemplateToPage;
  window.openSectionsLibrary = window.SBEditor.openSectionsLibrary;
  window.openCardsBuilderDialog = window.SBEditor.openCardsBuilderDialog;

  window.addTextBlock = window.SBEditor.addTextBlock;
  window.addImageBlock = window.SBEditor.addImageBlock;
  window.addButtonBlock = window.SBEditor.addButtonBlock;
  window.addHeadingBlock = window.SBEditor.addHeadingBlock;
  window.addCols2Block = window.SBEditor.addCols2Block;
  window.addGalleryBlock = window.SBEditor.addGalleryBlock;
  window.addSpacerBlock = window.SBEditor.addSpacerBlock;
  window.addCardBlock = window.SBEditor.addCardBlock;
  window.addCardsBlock = window.SBEditor.addCardsBlock;

  window.editTextBlock = window.SBEditor.editTextBlock;
  window.editImageBlock = window.SBEditor.editImageBlock;
  window.editButtonBlock = window.SBEditor.editButtonBlock;
  window.editHeadingBlock = window.SBEditor.editHeadingBlock;
  window.editCols2Block = window.SBEditor.editCols2Block;
  window.editGalleryBlock = window.SBEditor.editGalleryBlock;
  window.editSpacerBlock = window.SBEditor.editSpacerBlock;
  window.editCardBlock = window.SBEditor.editCardBlock;
  window.editCardsBlock = window.SBEditor.editCardsBlock;
})();

Если хочешь, следующим сообщением я скажу, как правильно подключить editor.dialogs.js и что уже можно удалить из editor.php.