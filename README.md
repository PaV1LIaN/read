https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess
{
  "ok": false,
  "error": "METHOD_NOT_ALLOWED"
}

console 
const fd = new FormData();
fd.append('siteId', '11');

fetch('/local/sitebuilder/api.php?action=site.syncAccess', {
  method: 'POST',
  body: fd
})
  .then(r => r.json())
  .then(console.log)
  .catch(console.error);
Promise {<pending>}
VM16:4  POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess 403 (Forbidden)
