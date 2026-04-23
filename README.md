Проблема уже не в API.
bootstrap у тебя успешный, значит дальше ломается именно отображение состояния в script.js.

Я нашел баг: в loadFolder() ты делаешь:

1. this.renderState('empty') если папка пустая


2. а потом в finally вызывается this.setLoading(false)


3. а setLoading(false) внутри делает renderState(null)



Из-за этого состояние сразу стирается, и ты видишь пустой белый блок.


---

Что исправить

Файл

/local/sitebuilder/components/disk/script.js

Найди функцию:

DiskComponent.prototype.setLoading = function (loading) {
  this.state.loading = !!loading;
  this.renderState(loading ? 'loading' : null);
};

И замени на:

DiskComponent.prototype.setLoading = function (loading) {
  this.state.loading = !!loading;

  var loadingNode = this.root.querySelector('[data-state="loading"]');
  if (loadingNode) {
    loadingNode.hidden = !loading;
  }
};


---

Почему это чинит

Теперь:

при loading = true будет показываться только индикатор загрузки

при loading = false он просто скроется

empty/error/no-access/no-root больше не будут затираться



---

Что должно получиться после правки

Так как у тебя rootFolderId = 324 и, скорее всего, папка пока пустая, после обновления страницы ты должен увидеть:

Здесь пока нет файлов и папок.

А если загрузишь файл или создашь папку — появится список.


---

Если хочешь сразу сделать поведение аккуратнее

Можно еще заменить renderState() на более надежный вариант, но сначала достаточно этой правки.

После замены:

1. сохрани script.js


2. сделай Ctrl + F5


3. обнови страницу



Если после этого будет пусто, следующим сообщением пришли response запроса list.