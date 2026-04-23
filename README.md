Pet-проект: Система полной наблюдаемости (Observability Stack)

Этот проект — полноценная система мониторинга, развёрнутая на VDS-сервере (Ubuntu 24.04). Она собирает метрики, логи и отправляет алерты в Telegram. Всё работает в Docker-контейнерах, а сервер защищён по промышленным стандартам.

🧱 Стек технологий
•	Среда: Docker, Docker Compose
•	Метрики: Prometheus, Node Exporter
•	Логи: Loki, Promtail
•	Визуализация: Grafana
•	Алерты: Alertmanager, Telegram Bot API
•	Безопасность: UFW, SSH-ключи, непривилегированный пользователь (gleb)
 
1. Подготовка сервера (Hardening)

Здесь мы превратили только что арендованный сервер в защищённую крепость.

1.1. Создание рабочего пользователя

Работать под root через SSH — плохая практика. Мы создали пользователя gleb и дали ему право на sudo без пароля (это нужно для скриптов и удобства).

adduser gleb --gecos ""
usermod -aG sudo gleb
Для sudo без пароля: visudo → gleb ALL=(ALL) NOPASSWD: ALL

1.2. Вход по SSH-ключу

Пароли можно подобрать, ключи — нет. Мы скопировали публичный ключ с локального ПК на сервер.
bash
# На сервере (под root)
mkdir -p /home/gleb/.ssh
echo "ПУБЛИЧНЫЙ_КЛЮЧ_ИЗ_ФАЙЛА_НА_ПК" >> /home/gleb/.ssh/authorized_keys
chmod 700 /home/gleb/.ssh
chmod 600 /home/gleb/.ssh/authorized_keys
chown -R gleb:gleb /home/gleb/.ssh
Теперь можно заходить без пароля: ssh gleb@<IP>

1.3. Финализация защиты SSH (/etc/ssh/sshd_config)

Мы запретили вход по паролю, вход для root, ограничили список пользователей и сменили стандартный порт на 2224.

Финальный конфиг /etc/ssh/sshd_config:
conf
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers gleb
Port 2224

🚧 Проблема: После правки основного конфига сервер всё ещё пускал по паролю.

🔍 Диагностика: sudo sshd -T | grep -i passwordauth показал yes. Это значит, что другой файл переопределяет настройку.

✅ Решение: В Ubuntu есть папка /etc/ssh/sshd_config.d/. Порядок чтения: сначала идёт основной конфиг, потом файлы из этой папки по алфавиту. Файл от хостера (40-hosting.conf) содержал PasswordAuthentication yes и побеждал. Мы исправили его на no, и проблема ушла.

1.4. Bash-алиас для быстрого входа (на локальном ПК)

Чтобы не вводить каждый раз IP и порт, на своём ПК (Windows) мы создали файл ~\.ssh\config:
text
Host myserver
    HostName 103.90.75.91
    Port 2224
    User gleb
    IdentityFile ~\.ssh\id_ed25519

Теперь подключение одной командой: ssh myserver.
 
2. Установка и настройка Docker

2.1. Зачем это нужно

Docker позволяет упаковывать приложения в изолированные «контейнеры».
Мы не просто установили Docker, а добавили официальный репозиторий с проверкой GPG-ключа — это гарантирует, что пакеты настоящие.

2.2. Установка (ключевые шаги)

bash
sudo apt update && sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
# Добавление репозитория и установка
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# Разрешаем пользователю gleb работать с Docker без sudo
sudo usermod -aG docker $USER
newgrp docker

2.3. Создание инфраструктуры для мониторинга

Мы подготовили «дом» для данных и изолированную сеть:
bash
sudo mkdir -p /opt/monitoring/{prometheus/data,grafana/data,loki,alertmanager}
sudo chown -R gleb:gleb /opt/monitoring
docker network create monitoring_net
 
3. Полный docker-compose.yml

Этот файл — сердце проекта. Он описывает все наши сервисы: Prometheus, Grafana, Node Exporter, Loki, Promtail, Alertmanager.
<details> <summary><b>Кликни, чтобы развернуть итоговый файл</b></summary>
yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    user: "1000:1000"  # Работаем от нашего пользователя, чтобы не было проблем с правами
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/data:/prometheus
      - ./prometheus/alerts.yml:/etc/prometheus/alerts.yml
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    ports:
      - "9090:9090"
    networks:
      - monitoring_net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    user: "1000:1000"
    volumes:
      - ./grafana/data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring_net

  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    user: "1000:1000"
    ports:
      - "9100:9100"
    networks:
      - monitoring_net

  loki:
    image: grafana/loki:latest
    container_name: loki
    user: "10001:10001" # У Loki специфичный UID внутри контейнера
    volumes:
      - ./loki/config:/etc/loki
      - ./loki/data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    ports:
      - "3100:3100"
    networks:
      - monitoring_net

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    # Работает от root внутри контейнера, чтобы иметь доступ к /var/log хоста
    volumes:
      - /var/log:/var/log
      - ./promtail/config:/etc/promtail
    command: -config.file=/etc/promtail/promtail-config.yml
    ports:
      - "9080:9080"
    networks:
      - monitoring_net

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    user: "1000:1000"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
    ports:
      - "9093:9093"
    networks:
      - monitoring_net

networks:
  monitoring_net:
    external: true
</details>

🧠 Ключевые концепции для собеседования:
volumes — нужны, чтобы данные (метрики, логи) сохранялись на диске хоста и не исчезали при удалении контейнера.
ports — пробрасывают сетевые порты из изолированной сети контейнеров на хост, открывая доступ к сервисам.
user: "1000:1000" — мы явно указали, что процесс в контейнере должен работать от нашего пользователя (gleb). Это решило все проблемы с правами на запись в папки.

 
4. Prometheus + Node Exporter (Метрики)

4.1. Конфиг Prometheus (prometheus.yml)

Этот файл говорит Прометею, у кого и как часто спрашивать метрики.
yaml
global:
  scrape_interval: 15s # Раз в 15 секунд забираем метрики

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'alertmanager:9093' # Куда слать алерты

rule_files:
  - "/etc/prometheus/alerts.yml" # Файл с правилами алертов

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090'] # Прометей следит сам за собой

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100'] # Имя контейнера = DNS-имя в сети

💡 Ключевая концепция:
Благодаря Docker-сети monitoring_net мы можем обращаться к контейнерам по их именам. node_exporter:9100 автоматически разрешится во внутренний IP контейнера. Это стандартный подход.
 
5. Loki + Promtail (Логи)

Логи — это второй столп наблюдаемости. Мы настроили их сбор и хранение.

5.1. Конфиг Loki (loki-config.yml)

yaml
auth_enabled: false
server:
  http_listen_port: 3100
common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules

5.2. Конфиг Promtail (promtail-config.yml)

Promtail читает логи с хоста и отправляет их в Loki.
yaml
server:
  http_listen_port: 9080
clients:
  - url: http://loki:3100/loki/api/v1/push # Адрес Loki
scrape_configs:
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: varlogs # Метка, по которой будем искать логи
          __path__: /var/log/*.log # Какие файлы читать на хосте

5.3. Основы языка запросов LogQL

•	{job="varlogs"} |= "error" → найти все логи с меткой varlogs, содержащие слово error.
•	{job="varlogs"} | detected_level = "error" → найти логи, у которых уровень error (Promtail определяет его автоматически).
 
6. Alertmanager + Telegram (Алерты)

Третий столп — proactive monitoring. Мы не просто смотрим на графики, а получаем сообщения, когда что-то идёт не так.

6.1. Шаги по регистрации Telegram-бота

1.	В Telegram у @BotFather создать нового бота (/newbot) и получить токен.
2.	Найти своего бота в поиске и отправить ему любое сообщение.
3.	На сервере выполнить curl "https://api.telegram.org/bot<ТОКЕН>/getUpdates" и из ответа скопировать свой chat_id.

6.2. Конфиг Alertmanager (alertmanager.yml)

yaml
global:
  resolve_timeout: 5m # Если проблема ушла, сообщим через 5 минут
  telegram_api_url: "https://api.telegram.org"

route:
  receiver: 'telegram-notifications'
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h

receivers:
  - name: 'telegram-notifications'
    telegram_configs:
      - bot_token: 'СЮДА_ВСТАВИТЬ_ТОКЕН'
        chat_id: СЮДА_ВСТАВИТЬ_ID
        message: |
          🚨 *Alert:* {{ .GroupLabels.alertname }}
          📊 *Status:* {{ .Status }}
          📝 *Description:* {{ .CommonAnnotations.description }}

6.3. Правила алертов (alerts.yml)

Мы создали два базовых правила, которые следят за ресурсами сервера:

yaml
groups:
  - name: host_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокая загрузка CPU на {{ $labels.instance }}"
          description: "CPU загрузка > 80% в течение 5 минут (текущая: {{ $value }}%)"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Высокая загрузка памяти на {{ $labels.instance }}"
          description: "Память занята > 85% в течение 5 минут (текущая: {{ $value }}%)"
 
7. Термины и вопросы для подготовки к собеседованию

📖 Глоссарий
•	Target (цель): Сервис, у которого Prometheus собирает метрики (например, node_exporter:9100).
•	Job: Группа однотипных целей. Все метрики от цели получают метку (label) job="node_exporter".
•	Volume (том): Механизм Docker для хранения данных контейнера на диске хоста.
•	LogQL: Язык запросов к логам в Loki (похож на PromQL для метрик).
•	Alert Rule: Условие в Prometheus, при котором срабатывает алерт (например, CPU > 80%).



