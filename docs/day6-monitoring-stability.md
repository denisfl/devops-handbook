# День 6. Мониторинг: Prometheus и Grafana

[monitoring] [prometheus] [grafana] [observability]

"Если у вас нет мониторинга, у вас нет продакшена".
Мониторинг отвечает на вопрос "Что сломалось?", а Observability (Наблюдаемость) — на вопрос "Почему это сломалось?".
Сегодня мы построим стандартный индустриальный стек мониторинга.

---

## Архитектура: Pull Model

Мир захватил **Prometheus**. Он работает по принципу **Pull (Тяни)**.
1.  **Exporters**: Маленькие агенты на серверах. Они не отправляют данные, они просто выставляют их на веб-страничке (обычно порт 9100/metrics).
2.  **Prometheus Server**: Раз в 15 секунд обходит список адресов, скачивает эти метрики и складывает в свою Time Series Database (TSDB).
3.  **Grafana**: Красивая веб-панель, которая ходит в Prometheus, берет цифры и рисует графики.

*Почему архитектурный спор важен?*

1.  **Push (Zabbix)**:
    *   *Плюс*: Агент может работать за NAT/Firewall (он сам инициирует соединение).
    *   *Минус*: Если вы случайно развернете 10 000 контейнеров, они одновременно "постучатся" в сервер мониторинга и устроят ему DDoS.
2.  **Pull (Prometheus)**:
    *   *Плюс*: **Контроль нагрузки**. Сервер мониторинга сам решает, когда и с какой скоростью забирать данные. Если он перегружен, он просто замедлит сбор, но не упадет.
    *   *Плюс*: **Обнаружение смерти**. Если сервер перестал отвечать на запросы (Scrape failed), Prometheus сразу помечает его как `DOWN`. В Push-модели тишина может означать "всё хорошо, событий нет" или "сервер сгорел".

---

## Практика: Поднимаем Stack

Мы используем Docker Compose, чтобы поднять полноценную систему мониторинга.

Создайте `docker-compose.yml`:
```yaml
version: '3.8'

services:
  # 1. Сборщик метрик машины (CPU, RAM, Disk)
  node-exporter:
    image: prom/node-exporter:latest
    ports:
      - "9100:9100"

  # 2. База данных и сборщик
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  # 3. Визуализация
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
```

И конфигурацию `prometheus.yml`:
```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'my-server'
    static_configs:
      - targets: ['node-exporter:9100'] # Имя сервиса из Docker Compose
```

### Запускаем и изучаем
1.  `docker-compose up -d`
2.  Откройте **Prometheus** (`localhost:9090`). Введите запрос `up`. Если видите `1`, значит агент доступен.
3.  Откройте **Grafana** (`localhost:3000`). Логин/пароль: `admin/admin`.
    *   Add Data Source -> Prometheus -> URL: `http://prometheus:9090`.
    *   Создайте Dashboard.
    *   Добавьте график. Query: `node_memory_MemAvailable_bytes`.

Вы только что настроили систему, которая используется в Google и Netflix, у себя на ноутбуке.

---

## Что еще нужно знать?

**Золотые Сигналы (The Four Golden Signals)** от Google SRE. Что мониторить в первую очередь?
1.  **Latency**: Как долго отвечает сервис.
2.  **Traffic**: Сколько запросов в секунду (RPS).
3.  **Errors**: Процент ошибок (500-е коды).
4.  **Saturation**: Насколько мы загружены (очереди, память).

Если вы мониторите эти 4 параметра, вы покроете 80% проблем.
