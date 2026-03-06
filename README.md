function togglePageStatus() {
  if (!pageMeta) {
    notify('Страница ещё не загружена');
    return;
  }

  const currentStatus = String(pageMeta.status || 'DRAFT').toUpperCase();
  const nextStatus = (currentStatus === 'PUBLISHED') ? 'DRAFT' : 'PUBLISHED';

  api('page.setStatus', { id: pageId, status: nextStatus })
    .then(res => {
      if (!res || res.ok !== true) {
        notify('Не удалось сменить статус: ' + (res?.error || 'UNKNOWN'));
        console.log('page.setStatus failed', res);
        return;
      }
      notify('Статус изменён: ' + nextStatus);
      loadPageMeta();
    })
    .catch(err => {
      console.log('page.setStatus error', err);
      notify('Ошибка page.setStatus');
    });
}