HierarchyRequestError: Failed to execute 'appendChild' on 'Node': The new child element contains the parent.
    at DiskComponent.arrangeSettingsModal (script.js?v=9:3515:15)
    at DiskComponent.openSettingsModal (script.js?v=9:3446:12)
    at async HTMLButtonElement.<anonymous> (script.js?v=9:2184:9)
DiskComponent.openSettingsModal @ script.js?v=9:3450
await in DiskComponent.openSettingsModal
(anonymous) @ script.js?v=9:2184
settings-v2.js?v=7:520  POST https://portal24.itsnn.ru/local/sitebuilder/components/disk/api.php?action=saveSettings 409 (Conflict)
window.fetch @ settings-v2.js?v=7:520
DiskComponent.api @ script.js?v=9:188
DiskComponent.saveSettings @ script.js?v=9:3629
(anonymous) @ script.js?v=9:2198
(anonymous) @ settings-v2.js?v=7:1540
settings-v2.js?v=7:591 [SiteBuilder Disk] block version refreshed 36 → 38; retry saveSettings once.
settings-v2.js?v=7:520 Fetch failed loading: POST "https://portal24.itsnn.ru/local/sitebuilder/components/disk/api.php?action=saveSettings".
window.fetch @ settings-v2.js?v=7:520
DiskComponent.api @ script.js?v=9:188
DiskComponent.saveSettings @ script.js?v=9:3629
(anonymous) @ script.js?v=9:2198
(anonymous) @ settings-v2.js?v=7:1540
settings-v2.js?v=7:601  POST https://portal24.itsnn.ru/local/sitebuilder/components/disk/api.php?action=saveSettings 409 (Conflict)
window.fetch @ settings-v2.js?v=7:601
await in window.fetch
DiskComponent.api @ script.js?v=9:188
DiskComponent.saveSettings @ script.js?v=9:3629
(anonymous) @ script.js?v=9:2198
(anonymous) @ settings-v2.js?v=7:1540
settings-v2.js?v=7:601 Fetch failed loading: POST "https://portal24.itsnn.ru/local/sitebuilder/components/disk/api.php?action=saveSettings".
window.fetch @ settings-v2.js?v=7:601
await in window.fetch
DiskComponent.api @ script.js?v=9:188
DiskComponent.saveSettings @ script.js?v=9:3629
(anonymous) @ script.js?v=9:2198
(anonymous) @ settings-v2.js?v=7:1540
script.js?v=9:3645 Error: Объект был изменён в другой вкладке.
    at DiskComponent.saveSettings (script.js?v=9:3631:15)
    at async HTMLButtonElement.<anonymous> (script.js?v=9:2198:9)
