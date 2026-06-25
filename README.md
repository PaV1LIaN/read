[ErrorException] E_USER_ERROR
Unable to get session lock within 60 seconds. (0)
/srv/bx/docroot/bitrix/modules/main/lib/session/handlers/abstractsessionhandler.php:71
#0: trigger_error
	/srv/bx/docroot/bitrix/modules/main/lib/session/handlers/abstractsessionhandler.php:71
#1: Bitrix\Main\Session\Handlers\AbstractSessionHandler->triggerLockFatalError
	/srv/bx/docroot/bitrix/modules/main/lib/session/handlers/abstractsessionhandler.php:53
#2: Bitrix\Main\Session\Handlers\AbstractSessionHandler->read
	/srv/bx/docroot/bitrix/modules/main/lib/session/handlers/abstractsessionhandler.php:153
#3: Bitrix\Main\Session\Handlers\AbstractSessionHandler->validateId
	
#4: session_start
	/srv/bx/docroot/bitrix/modules/main/lib/session/session.php:154
#5: Bitrix\Main\Session\Session->start
	/srv/bx/docroot/bitrix/modules/main/lib/session/kernelsessionproxy.php:47
#6: Bitrix\Main\Session\KernelSessionProxy->start
	/srv/bx/docroot/bitrix/modules/main/include.php:182
#7: require_once(string)
	/srv/bx/docroot/bitrix/modules/main/include/prolog_before.php:19
#8: require(string)
	/srv/bx/docroot/local/sitebuilder/public.php:2
----------
