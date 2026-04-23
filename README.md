markdown
# 🛰️ Pet-проект: Система полной наблюдаемости (Observability Stack)

Этот проект — полноценная система мониторинга, развёрнутая на VDS-сервере (Ubuntu 24.04).  
Она собирает **метрики**, **логи** и отправляет **алерты** в Telegram.  
Всё работает в Docker-контейнерах, а сервер защищён по промышленным стандартам.

### 🧱 Стек технологий
- **Среда:** Docker, Docker Compose
- **Метрики:** Prometheus, Node Exporter
- **Логи:** Loki, Promtail
- **Визуализация:** Grafana
- **Алерты:** Alertmanager, Telegram Bot API
- **Безопасность:** UFW, SSH-ключи, непривилегированный пользователь (`gleb`)

---

## 1. Подготовка сервера (Hardening)

### 1.1. Создание рабочего пользователя
Работать под `root` через SSH — плохая практика. Мы создали пользователя `gleb` и дали ему право на `sudo` без пароля.

```bash
adduser gleb --gecos ""
usermod -aG sudo gleb
Для sudo без пароля: visudo → gleb ALL=(ALL) NOPASSWD: ALL

1.2. Вход по SSH-ключу
Пароли можно подобрать, ключи — нет. Мы скопировали публичный ключ с локального ПК на сервер.

bash
mkdir -p /home/gleb/.ssh
echo "ПУБЛИЧНЫЙ_КЛЮЧ" >> /home/gleb/.ssh/authorized_keys
chmod 700 /home/gleb/.ssh
chmod 600 /home/gleb/.ssh/authorized_keys
chown -R gleb:gleb /home/gleb/.ssh
1.3. Финализация защиты SSH
Мы запретили вход по паролю, вход для root и сменили порт на 2224.

conf
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers gleb
Port 2224
Проблема: Сервер всё ещё пускал по паролю.
Диагностика: sudo sshd -T | grep -i passwordauth показал yes.
Решение: Файл хостера в /etc/ssh/sshd_config.d/ переопределял настройку. Исправили.

1.4. Быстрый вход по SSH
Файл ~/.ssh/config на локальном ПК:

text
Host myserver
    HostName 103.90.75.91
    Port 2224
    User gleb
    IdentityFile ~/.ssh/id_ed25519
2. Установка Docker
bash
sudo apt update && sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
newgrp docker
bash
sudo mkdir -p /opt/monitoring/{prometheus/data,grafana/data,loki,alertmanager}
sudo chown -R gleb:gleb /opt/monitoring
docker network create monitoring_net
3. Итоговый docker-compose.yml
yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    user: "1000:1000"
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
    user: "10001:10001"
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
4. Prometheus + Node Exporter
yaml
global:
  scrape_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - 'alertmanager:9093'

rule_files:
  - "/etc/prometheus/alerts.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node_exporter:9100']
5. Loki + Promtail
Конфиг Loki
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
Конфиг Promtail
yaml
server:
  http_listen_port: 9080
clients:
  - url: http://loki:3100/loki/api/v1/push
scrape_configs:
  - job_name: system
    static_configs:
      - targets: [localhost]
        labels:
          job: varlogs
          __path__: /var/log/*.log
6. Alertmanager + Telegram
yaml
global:
  resolve_timeout: 5m
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
      - bot_token: 'ТОКЕН'
        chat_id: ID
        message: |
          🚨 Alert: {{ .GroupLabels.alertname }}
          Status: {{ .Status }}
          Description: {{ .CommonAnnotations.description }}
Правила алертов
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
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage > 80% (current: {{ $value }}%)"

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage > 85% (current: {{ $value }}%)"
7. Глоссарий
Target — сервис, у которого Prometheus собирает метрики.

Job — группа однотипных целей.

Volume — хранение данных контейнера на диске хоста.

LogQL — язык запросов к логам в Loki.

Alert Rule — условие для срабатывания алерта.

text
