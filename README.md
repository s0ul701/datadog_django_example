# Настройка связки DataDog + Django

[DataDog](https://www.datadoghq.com/) - система мониторинга состояния сервера, обладающая широким спектром возможностей.
Данная система является клиент-серверной, т.е. на контролируемом сервере устанавливается ТОЛЬКО агент, который отсылает метрики на сервера DataDog.

---

## Подготовка

1. [Регистрация](https://app.datadoghq.com/signup) (создание личного кабинета на стороне DataDog, откуда и осуществляется мониторинг)
2. [Получение API-ключа](https://app.datadoghq.eu/account/settings#api)

---

## 1. Подключение Docker-интеграции

Данная интеграция позволяет мониторить состояние запущенных на сервере контейнеров (на достаточно высоком уровне абстракции: CPU, RAM, I/O и т.д.), не анализируя специфичные для запущенных в контейнерах приложений метрики.

***./docker-compose.yml:***

```yaml
version: '3.3'
services:
    ...other services...

    datadog-agent:
        image: datadog/agent:7.26.0-jmx
        env_file:
            - ./datadog/.env
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock:ro  # с помощью вольюмов осуществляется
            - /proc/:/host/proc/:ro                         # сбор метрик с контейнеров/сервера
            - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro

    ...more other services...
```

***./datadog.env:***

```yaml
DD_API_KEY=<YOUR_DATADOG_API_KEY>
DD_SITE=<YOUR_DATADOG_DOMEN>

DD_PROCESS_AGENT_ENABLED=true   # позволяет просматривать процессы сервера/контейнеров в DataDog
```

Результаты настроек доступны по [ссылке](https://app.datadoghq.eu/containers):

*Ссылки*:

1. [Документация](https://docs.datadoghq.com/integrations/faq/compose-and-the-datadog-agent/) по базовой настройке связки DataDog/Docker/Docker Compose;
2. Базовая [документация](https://docs.datadoghq.com/agent/docker/?tab=standard) по Datadog Agent.

<br>

## 2. Подключение Django-интеграции

Данная интеграция позволяет отслеживать трассировку Django-приложения, осуществлять его профилирование, собирать логи и т.д.

***./docker-compose.yml:***

```yaml
version: '3.3'
services:
  server:
    build:
      context: ./
      dockerfile: ./server/Dockerfile
    command: gunicorn config.wsgi -c ./config/gunicorn.py
    ports:
      - 8000:8000
    depends_on:
      - db
      - datadog-agent
    env_file:
      - ./server/.env
    networks:
      - default

  db:
    image: postgres:12.4-alpine
    env_file: ./db/.env
    networks:
      - default

  datadog-agent:
    image: datadog/agent:7.26.0-jmx
    env_file: ./datadog_agent/.env
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /proc/:/host/proc/:ro
      - /sys/fs/cgroup:/host/sys/fs/cgroup:ro
    expose:     # default-порт контейнера Datadog-агента, на который 
      - 8126    # он принимает данные от APM-интеграций
    networks:
      - default


networks:             # кастомная сеть необходима
  default:            # для корректного нахождения контейнера
    driver: bridge    # DataDog-агента контейнером Django
```

***./datadog.env:***

```yaml
DD_API_KEY=<DATADOG_API_KEY>
DD_SITE=<DATADOG_DOMEN>

DD_LOG_LEVEL=warn   # минимальный уровень лог-сообщений для контейнера DataDog-агента
DD_BIND_HOST=0.0.0.0    # разрешение принимать данные от APM-интеграций со всей локальной сети, в которой находится DataDog-агент
```

***./server.env:***

```yaml
...other env variables...

DD_AGENT_HOST=datadog-agent   # имя хоста DataDog-агента
DD_PROFILING_ENABLED=true   # включение механизма профилирования
DD_SERVICE=django       # имя сервиса для отображение в DataDog UI

...more other env variables...
```

Ссылки:
1. [Статья](https://www.datadoghq.com/blog/monitoring-django-performance/) по настройке интеграции Django с DataDog
