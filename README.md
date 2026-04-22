<pre>[PDOException] 
SQLSTATE[23505]: Unique violation: 7 ОШИБКА:  повторяющееся значение ключа нарушает ограничение уникальности &quot;uq_access_site_code&quot;
DETAIL:  Ключ &quot;(site_id, access_code)=(6, U1)&quot; уже существует. (23505)
/srv/bx/docroot/local/sitebuilder/lib/db.php:54
#0: PDOStatement-&gt;execute
	/srv/bx/docroot/local/sitebuilder/lib/db.php:54
#1: sb_db_execute
	/srv/bx/docroot/local/sitebuilder/lib/storage_db_extra.php:86
#2: sb_write_access
	/srv/bx/docroot/local/sitebuilder/api/handlers/site.php:125
#3: require(string)
	/srv/bx/docroot/local/sitebuilder/api/index.php:20
#4: require_once(string)
	/srv/bx/docroot/local/sitebuilder/api.php:5
----------
</pre>
