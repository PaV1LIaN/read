[PDOException] 
SQLSTATE[42P08]: Ambiguous parameter: 7 ОШИБКА:  для параметра $1 выведены несогласованные типы
LINE 1: UPDATE sitebuilder.form_submission SET status=$1,handled_by=...
                                                      ^
DETAIL:  text и character varying (42P08)
/srv/bx/docroot/local/sitebuilder/lib/db.php:61
#0: PDOStatement->execute
	/srv/bx/docroot/local/sitebuilder/lib/db.php:61
#1: sb_db_fetch_one
	/srv/bx/docroot/local/sitebuilder/lib/FormSubmissionService.php:986
#2: FormSubmissionService::updateStatus
	/srv/bx/docroot/local/sitebuilder/forms.php:199
----------
