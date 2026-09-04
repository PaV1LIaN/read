
			
			<div id='footer'>
				<span>Поддержка +7(831)299-09-11</span>&copy; 2023
			</div>

			
		</div>
		
		
 
    <!-- Модальное окно авторизации -->
    <div class="modal fade" id="loginModal" tabindex="-1" aria-labelledby="loginModalLabel" aria-hidden="true">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title" id="loginModalLabel">Авторизация</h5>
                    <button type="button" class="close" data-dismiss="modal" aria-label="Close">
                        <span>&times;</span>
                    </button>
                </div>
                <div class="modal-body">
                    <form>
						<table>
							<tr>
								<td>
									<label for="email">Логин</label>
								</td>
								<td>
									<input type="email" class="form-control" id="email" placeholder="Введите электронную почту" required>
								</td>
							</tr>
							<tr>
								<td>
									<label for="password">Пароль</label>
								</td>
								<td>
									<input type="password" class="form-control" id="password" placeholder="Введите пароль" required>
								</td>
							</tr>
							<tr>
								<td></td>
								<td>								
									<input type="checkbox" class="form-check-input" id="rememberMe">
									<label class="form-check-label" for="rememberMe">Запомнить меня</label>								
								</td>
							</tr>
						</table>						
                        <button type="submit" class="btn btn-primary">Войти</button>
                    </form>
                </div>
                <div class="modal-footer">
                    <button type="button" class="btn btn-secondary" data-dismiss="modal">Закрыть</button>
                </div>
            </div>
        </div>
    </div>		


	</body>
</html>

<?php
// Подключаем футер Битрикса, закрываем теги и подключаем скрипты
require_once($_SERVER['DOCUMENT_ROOT'].'/bitrix/footer.php');
?>






<?if(!defined("B_PROLOG_INCLUDED") || B_PROLOG_INCLUDED!==true)die();?>
<?IncludeTemplateLangFile(__FILE__);?>

<?php
use Bitrix\Main\Page\Asset;

Asset::getInstance()->addCss(SITE_TEMPLATE_PATH."/styles.css");
Asset::getInstance()->addCss(SITE_TEMPLATE_PATH."/css/modal.css");
Asset::getInstance()->addCss('/local/glab/assets/lab_global_search.css');
?>

<!DOCTYPE html>
<html>
  <head>
    <title><?$APPLICATION->ShowTitle();?></title>
    <meta name='viewport' content='width=device-width, initial-scale=1.0'>
    <meta http-equiv='Content-Type' content='text/html; charset=UTF-8' />
    <?$APPLICATION->ShowHead();?>

    <style>
      #search {
        position: relative;
      }

      #search .d1 form {
        position: relative;
        display: flex;
        align-items: center;
        margin: 0;
      }

      #labGlobalSearch {
        width: 100%;
        padding-right: 46px;
        box-sizing: border-box;
      }

      #labGlobalSearchButton {
        position: absolute;
        right: 4px;
        top: 50%;
        transform: translateY(-50%);

        width: 34px;
        height: 34px;

        display: flex;
        align-items: center;
        justify-content: center;

        border: none;
        border-radius: 50%;

        background: #78ccfd;
        color: #fff;

        cursor: pointer;

        transition:
          background 0.2s ease,
          box-shadow 0.2s ease,
          transform 0.2s ease;
      }

      #labGlobalSearchButton:hover {
        background: #55bdf5;
        box-shadow: 0 6px 14px rgba(0, 0, 0, 0.16);
      }

      #labGlobalSearchButton:active {
        transform: translateY(-50%) scale(0.96);
      }

      #labGlobalSearchButton svg {
        width: 17px;
        height: 17px;
        display: block;
        fill: currentColor;
      }
    </style>

    <script type='text/javascript' src='<?=SITE_TEMPLATE_PATH?>/js/jquery.js'></script>
    <script src="https://stackpath.bootstrapcdn.com/bootstrap/4.5.2/js/bootstrap.min.js"></script>
    <script src="<?=SITE_TEMPLATE_PATH?>/js/script.js"></script>

    <script>
      window.LAB_GLOBAL_SEARCH = {
        sessid: "<?=bitrix_sessid()?>",
        ajaxUrl: "/local/glab/ajax/search_applications.php"
      };
    </script>
    <script src="/local/glab/assets/lab_global_search.js" defer></script>

    <script>
      document.addEventListener('DOMContentLoaded', function () {
        var form = document.getElementById('labGlobalSearchForm');
        var input = document.getElementById('labGlobalSearch');
        var button = document.getElementById('labGlobalSearchButton');

        if (form) {
          form.addEventListener('submit', function (event) {
            event.preventDefault();

            if (!input) {
              return;
            }

            input.focus();
            input.dispatchEvent(new Event('input', { bubbles: true }));
          });
        }

        if (button) {
          button.addEventListener('click', function () {
            if (!input) {
              return;
            }

            input.focus();
            input.dispatchEvent(new Event('input', { bubbles: true }));
          });
        }
      });
    </script>
  </head>

  <body>
    <?$APPLICATION->ShowPanel();?>
    <div id='content'>

      <div id='topline' class='dis-flex'>
        <? date_default_timezone_set('Europe/Moscow'); ?>
        <div id='timer'><?=date('d.m.Y H:i')?></div>

        <?php
          global $USER;
          if($USER->IsAuthorized()):
            $uid = (int)$USER->GetID();
            $fio = htmlspecialcharsbx($USER->GetFullName());
        ?>
          <nav id="user-nav">
            <a href="/local/glab/profile.php?user_id=<?=$uid?>" class="user-link"><?=$fio?></a>
          </nav>
        <?php endif; ?>
      </div>

      <div id='header' class='dis-flex'>
        <div id="logo" class="lab-header-logo">
  			<img
    			class="lab-header-logo__img"
    			src="<?=SITE_TEMPLATE_PATH?>/img/logo.png"
    			alt="Laboratory"
  			>
  <span class="lab-header-logo__text">Laboratory</span>
</div>

        <div id='search'>
          <div class="fa d1">
            <form id="labGlobalSearchForm" action="javascript:void(0);" method="get" autocomplete="off">
              <input
                id="labGlobalSearch"
                name="s"
                placeholder="Поиск заявки"
                type="search"
                autocomplete="off"
              >

              <button id="labGlobalSearchButton" type="submit" aria-label="Поиск">
                <svg viewBox="0 0 24 24" aria-hidden="true">
                  <path d="M9.5 3C5.9 3 3 5.9 3 9.5S5.9 16 9.5 16c1.6 0 3.1-.6 4.2-1.5l4.4 4.4c.4.4 1 .4 1.4 0s.4-1 0-1.4l-4.4-4.4c.9-1.1 1.5-2.6 1.5-4.2C16 5.9 13.1 3 9.5 3zm0 2C12 5 14 7 14 9.5S12 14 9.5 14 5 12 5 9.5 7 5 9.5 5z"/>
                </svg>
              </button>
            </form>
          </div>

          <div id="labSearchPopup" class="lab-search-popup" style="display:none;">
            <div class="lab-search-popup__head">
              <div class="lab-search-popup__title">Результаты</div>
              <div class="lab-search-popup__hint">Enter — открыть, Esc — закрыть</div>
            </div>
            <div id="labSearchPopupBody" class="lab-search-popup__body"></div>
          </div>
        </div>
      </div>

      <?php
      $APPLICATION->IncludeComponent(
        "bitrix:menu",
        "top",
        [
          "ROOT_MENU_TYPE"        => "top",
          "MAX_LEVEL"             => "1",
          "CHILD_MENU_TYPE"       => "",
          "USE_EXT"               => "Y",
          "DELAY"                 => "N",
          "ALLOW_MULTI_SELECT"    => "N",
          "MENU_CACHE_TYPE"       => "A",
          "MENU_CACHE_TIME"       => "3600",
          "MENU_CACHE_USE_GROUPS" => "Y",
          "MENU_CACHE_GET_VARS"   => []
        ],
        false
      );
      ?>





