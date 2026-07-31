{
    "generatedAt": "2026-07-31T13:22:57+03:00",
    "phpVersion": "8.1.12-1ubuntu4.3+ci6",
    "userId": 1,
    "siteId": 13,
    "initialError": null,
    "checks": [
        {
            "name": "database.connection",
            "ok": true,
            "durationMs": 2,
            "result": {
                "database": "ProjectsDB",
                "db_user": "bx_user",
                "version": "PostgreSQL 17.4 on x86_64-pc-linux-gnu, compiled by gcc (AstraLinuxSE 8.3.0-6) 8.3.0, 64-bit"
            }
        },
        {
            "name": "schema.objects",
            "ok": true,
            "durationMs": 16,
            "result": {
                "sitebuilder.site": true,
                "sitebuilder.page": true,
                "sitebuilder.block": true,
                "sitebuilder.entity_revision": true,
                "sitebuilder.audit_log": true,
                "sitebuilder.schema_migration": true,
                "sitebuilder.page_id_seq": true
            }
        },
        {
            "name": "schema.columns",
            "ok": true,
            "durationMs": 30,
            "result": {
                "sitebuilder.page.version": true,
                "sitebuilder.page.seo_json": true,
                "sitebuilder.page.published_at": true,
                "sitebuilder.entity_revision.restored_from_revision_id": true,
                "sitebuilder.entity_revision.snapshot_json": true
            }
        },
        {
            "name": "schema.page_constraints",
            "ok": true,
            "durationMs": 4,
            "result": [
                {
                    "conname": "page_pkey",
                    "definition": "PRIMARY KEY (id)"
                },
                {
                    "conname": "page_seo_json_chk",
                    "definition": "CHECK ((jsonb_typeof(seo_json) = 'object'::text))"
                }
            ]
        },
        {
            "name": "schema.revision_constraints",
            "ok": true,
            "durationMs": 3,
            "result": [
                {
                    "conname": "entity_revision_pkey",
                    "definition": "PRIMARY KEY (id)"
                },
                {
                    "conname": "entity_revision_type_chk",
                    "definition": "CHECK (((entity_type)::text = ANY ((ARRAY['site'::character varying, 'page'::character varying, 'block'::character varying, 'menu'::character varying, 'layout'::character varying])::text[])))"
                },
                {
                    "conname": "entity_revision_version_chk",
                    "definition": "CHECK ((entity_version > 0))"
                }
            ]
        },
        {
            "name": "migration.status",
            "ok": true,
            "durationMs": 96,
            "result": {
                "registryReady": true,
                "ready": true,
                "pendingCount": 0,
                "driftCount": 0,
                "invalidCount": 0,
                "items": [
                    {
                        "key": "20260729_001_entity_versions",
                        "stage": 5,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260729_002_site_menu_layout_versions_and_recycle_bin",
                        "stage": 6,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260729_003_page_sections_audit_retention",
                        "stage": 7,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260729_004_sequences_and_external_jobs",
                        "stage": 9,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260729_005_external_cleanup_and_queue_health",
                        "stage": 10,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260729_006_external_reconciliation_and_alerts",
                        "stage": 11,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260729_007_backups_and_integrity",
                        "stage": 12,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260730_008_migration_registry_and_deployment_runs",
                        "stage": 13,
                        "state": "applied",
                        "fingerprintPassed": true
                    },
                    {
                        "key": "20260730_009_forms_and_seo",
                        "stage": 20,
                        "state": "applied",
                        "fingerprintPassed": true
                    }
                ]
            }
        },
        {
            "name": "site.context",
            "ok": true,
            "durationMs": 5,
            "result": {
                "site": {
                    "id": 13,
                    "name": "Тестовый сайт",
                    "slug": "test",
                    "sectionId": 0,
                    "homePageId": 0,
                    "diskFolderId": 408,
                    "topMenuId": 0,
                    "bitrixGroupId": 7,
                    "bitrixGroupCreatedBy": 1,
                    "bitrixGroupCreatedAt": "2026-04-30 11:46:32.506791",
                    "bitrixGroupUrl": "/workgroups/group/7/",
                    "settings": {
                        "accent": "#2563eb",
                        "logoSize": 60,
                        "logoFileId": 63516,
                        "backgroundMode": "auto",
                        "containerWidth": 1920,
                        "headerLogoMode": "both",
                        "backgroundColor": "#ffffff",
                        "backgroundFileId": 63519,
                        "backgroundRepeat": "repeat",
                        "backgroundPosition": "center center"
                    },
                    "layout": {
                        "leftMode": "blocks",
                        "showLeft": false,
                        "leftWidth": 260,
                        "showRight": false,
                        "rightWidth": 260,
                        "showFooter": true,
                        "showHeader": true
                    },
                    "createdBy": 1,
                    "createdAt": "2026-04-30 11:46:32",
                    "updatedBy": 1,
                    "updatedAt": "2026-05-21 10:31:15",
                    "version": 1
                },
                "pageCount": 3,
                "globalEdit": true
            }
        }
    ],
    "writeTests": [
        {
            "name": "write.page_create.full_path",
            "durationMs": 17,
            "ok": false,
            "exception": "PDOException",
            "message": "SQLSTATE[23514]: Check violation: 7 ОШИБКА:  новая строка в отношении \"page\" нарушает ограничение-проверку \"page_seo_json_chk\"\nDETAIL:  Ошибочная строка содержит (50, 13, SiteBuilder diagnostic page, __diagnostic-50-20260731132257, null, 2147483000, draft, null, 1, 2026-07-31 13:22:57, 1, 2026-07-31 13:22:57, 1, []).",
            "sqlState": "23514",
            "file": "/local/sitebuilder/lib/storage_db.php",
            "line": 133
        },
        {
            "name": "write.page_publish.full_path",
            "durationMs": 29,
            "ok": false,
            "exception": "PDOException",
            "message": "SQLSTATE[23514]: Check violation: 7 ОШИБКА:  новая строка в отношении \"page\" нарушает ограничение-проверку \"page_seo_json_chk\"\nDETAIL:  Ошибочная строка содержит (14, 13, Диск, disk, null, 20, draft, null, 0, 2026-04-30 13:42:26, 1, 2026-07-31 13:22:57.495096, 2, []).",
            "sqlState": "23514",
            "file": "/local/sitebuilder/lib/RevisionService.php",
            "line": 226
        }
    ],
    "notes": [
        "Тесты записи выполняются внутри транзакции и откатываются.",
        "PostgreSQL sequence не откатывается, поэтому после теста возможен пропуск одного ID страницы. Это нормально."
    ]
}
