fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    action: 'site.get',
    siteId: '11',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(async function (r) {
  const text = await r.text();
  console.log('STATUS:', r.status);
  console.log('TEXT:', text);

  try {
    console.log('JSON:', JSON.parse(text));
  } catch (e) {
    console.log('NOT JSON');
  }
})
.catch(console.error);
Promise {<pending>}
fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    action: 'site.syncAccess',
    siteId: '11',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(async function (r) {
  const text = await r.text();
  console.log('STATUS:', r.status);
  console.log('TEXT:', text);

  try {
    console.log('JSON:', JSON.parse(text));
  } catch (e) {
    console.log('NOT JSON');
  }
})
.catch(console.error);
Promise {<pending>}
VM10524:1  POST https://portal24.itsnn.ru/local/sitebuilder/api.php 400 (Bad Request)


{
    "ok": true,
    "site": {
        "id": 11,
        "name": "321",
        "slug": "321-2",
        "homePageId": 0,
        "diskFolderId": 0,
        "topMenuId": 0,
        "settings": {
            "accent": "#2563eb",
            "logoFileId": 0,
            "containerWidth": 1100
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
        "createdAt": "2026-04-24 10:46:10",
        "updatedBy": 1,
        "updatedAt": "2026-04-24 10:46:10"
    },
    "handler": "site",
    "file": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/handlers\/site.php"
}

{
    "ok": false,
    "error": "UNKNOWN_ACTION",
    "action": "site.syncAccess",
    "file": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/index.php"
}





