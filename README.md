[PDOException] 
SQLSTATE[08006] [7] could not connect to server: Connection refused
	Is the server running on host "127.0.0.1" and accepting
	TCP/IP connections on port 5432? (7)
/srv/bx/docroot/local/sitebuilder/lib/db.php:20
#0: PDO->__construct
	/srv/bx/docroot/local/sitebuilder/lib/db.php:20
#1: sb_db
	/srv/bx/docroot/local/sitebuilder/tools/migrate_json_to_pg.php:365
----------
