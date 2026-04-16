USER_ID=1
AUTHORIZED=Y
ROLE=
[PDOException] 
SQLSTATE[42P01]: Undefined table: 7 ОШИБКА:  отношение "sitebuilder_site_user_access" не существует
LINE 3:             FROM sitebuilder_site_user_access
                         ^ (42P01)
/srv/bx/docroot/local/sitebuilder/components/disk/lib/DiskDb.php:47
#0: PDOStatement->execute
	/srv/bx/docroot/local/sitebuilder/components/disk/lib/DiskDb.php:47
#1: DiskDb::fetchOne
	/srv/bx/docroot/local/sitebuilder/components/disk/lib/SiteAccessRepository.php:21
#2: SiteAccessRepository::getUserRole
	/srv/bx/docroot/local/sitebuilder/components/disk/test.php:15
----------
