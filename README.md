
{ok: false, data: [], meta: [], error: "VERSION_CONFLICT",…}
data
: 
[]
details
: 
{entityType: "block", entityId: 162, expectedVersion: 75, currentVersion: 78}
error
: 
"VERSION_CONFLICT"
message
: 
"Объект был изменён в другой вкладке."
meta
: 
[]
ok
: 
false

script.js?v=9:188  POST https://portal24.itsnn.ru/local/sitebuilder/components/disk/api.php?action=saveSettings 409 (Conflict)
DiskComponent.api @ script.js?v=9:188
DiskComponent.saveSettings @ script.js?v=9:3629
(anonymous) @ script.js?v=9:2198
(anonymous) @ settings-v2.js?v=4:653
script.js?v=9:3645 Error: Объект был изменён в другой вкладке.
    at DiskComponent.saveSettings (script.js?v=9:3631:15)
    at async HTMLButtonElement.<anonymous> (script.js?v=9:2198:9)
DiskComponent.saveSettings @ script.js?v=9:3645
await in DiskComponent.saveSettings
(anonymous) @ script.js?v=9:2198
(anonymous) @ settings-v2.js?v=4:653
script.js?v=9:188 Fetch failed loading: POST "https://portal24.itsnn.ru/local/sitebuilder/components/disk/api.php?action=saveSettings".
