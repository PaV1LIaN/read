Обработка заняла 2m 38s

Нашёл официальный репозиторий: bitrix-tools/bi-builder. В нём готовая сборка Битрикс: Superset, Trino, MySQL, Redis и Nginx.

Перед подключением проверь версии портала: согласно README нужны:

- Главный модуль — 26.750.0+. Ранее ты указывал 25.900.0: если не обновлял, потребуется обновление.
- BI-коннектор — 26.1050.0+.
- BI Конструктор — 26.400.0+.
- Активное расширение лицензии для локального BI Конструктора. 

Серверу нужно минимум 8 CPU, 12 ГБ RAM и 40 ГБ свободного диска. Ниже команды для отдельного сервера с Debian 12/13 без установленного Docker.

1. Установить Docker

Выполняй блоки по очереди в PuTTY. Если команда завершилась ошибкой — дальше не продолжай.

sudo apt update
sudo apt install -y ca-certificates curl git openssl

sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources >/dev/null <<EOF
Types: deb
URIs: https://download.docker.com/linux/debian
Suites: $(. /etc/os-release && echo "$VERSION_CODENAME")
Components: stable
Architectures: $(dpkg --print-architecture)
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

sudo systemctl enable --now docker
sudo docker compose version

Это установка из официального репозитория Docker. 

2. Скачать BI Конструктор и создать настройки

git clone https://github.com/bitrix-tools/bi-builder.git "$HOME/bi-builder"
cd "$HOME/bi-builder"

umask 077
bash generate-env.sh
chmod 600 .env

Скрипт спросит:

- Протокол портала: обычно 1 — HTTPS.
- Адрес существующего Битрикс24, например portal.company.ru, без https:// и без пути.

Пароли и ключи сгенерируются автоматически. Файл .env сохрани вместе с резервной копией; в Git его не загружай.

Диагностический порт ограничим доступом с самого сервера:

sed -i 's/^SUPERSET_PORT=.*/SUPERSET_PORT=127.0.0.1:8088/' .env

3. Подготовить HTTPS

Для рабочего подключения нужен DNS-адрес нового сервера, например bi.company.ru, и доверенный сертификат на него.

mkdir -p "$HOME/bi-builder/ssl"

Через Total Commander загрузи:

- Сертификат с цепочкой → ~/bi-builder/ssl/cert.pem
- Приватный ключ → ~/bi-builder/ssl/key.pem

Если это пока только пробный запуск, вместо загрузки можно выполнить:

cd "$HOME/bi-builder"
bash generate-ssl.sh

Этот скрипт создаёт самоподписанный сертификат на localhost; для штатного подключения портала его потребуется заменить.

4. Запустить

cd "$HOME/bi-builder"
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps -a

Контейнер superset-init должен завершиться с кодом 0 — это штатная инициализация.

Если запуск завершился ошибкой:

sudo docker compose logs --tail=100 superset-init mysql trino nginx

Для скачивания нужны доступ к репозиториям Debian, Docker, GitHub, Docker Hub и quay.io. Для работы — связь BI с порталом, а также доступ портала и браузеров пользователей к BI по HTTPS/443.

5. Подключить в Битрикс24

Открой:

Настройки → Настройки продукта → Настройки модулей → BI-коннектор → BI Конструктор.

Выбери локальный режим, укажи https://bi.company.ru и пароль из файла .env. Посмотреть только нужный пароль:

cd "$HOME/bi-builder"
sed -n '/^BI_BUILDER_ADMIN_PASSWORD=/p' .env

Пароль вводится в настройках портала; присылать его мне не нужно.

Остановить сборку с сохранением данных:

cd "$HOME/bi-builder"
sudo docker compose down

Не добавляй -v: этот параметр удаляет тома с данными BI.