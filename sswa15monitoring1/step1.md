На этом шаге Вы запустите Prometheus. Prometheus — это система мониторинга с открытым кодом, работающая по pull-модели сбора метрик. Prometheus также предоставляет возможность запрашивать данные с помощью языка запросов PromQL.

Prometheus написан на языке программирования Go и распространяется в виде бинарного запускаемого файла. Чтобы запустить Prometheus достаточно запустить исполняемый файл и передать ему конфигурацию.

Минимальная конфигурация для Prometheus представляет собой файл в формате yaml. В файле есть две секции: global — для глобальных настроек и scrape_configs — конфигурация скрапинга.

Prometheus с некоторой периодичностью мониторит и получает метрики от сервисов в формате Prometheus expose format. Это процесс называется скрапингом (scaping, от английского scrape — выскабливать). А сервисы, которые должен мониторить Prometheus таргетами (targets от английского target — цель). Поэтому основные настройки Prometheus, которые мы будем использовать в конфигурации, будут относится к этому процессу.

Вот пример простейшего файла настроек для Prometheus. Введите строки ниже в файл prometheus.yml. Вы можете сделать это вручную или нажав кнопку «Copy to Editor».

<pre class="file" data-filename="prometheus.yml" data-target="replace">
global:
  scrape_interval:     15s
scrape_configs:
- job_name: app
  metrics_path: '/metrics'
  static_configs:
    - targets: ['127.0.0.1:8000']
</pre>

Давайте разберём настройки из этого файла.

## Глобальные настройки 

scrape_interval: 15s - это интервал, с которым Prometheus будет ходить за метриками в таргеты по умолчанию. 

Настройки, описывающие, конкретные таргеты, находятся в разделе scrape_configs. Это раздел представляет собой список задач (job). Задача в контексте Prometheus — это коллекция таргетов, имеющих одно и то же предназначение, плюс настройки скрапинга. Например, приложение может быть в виде 3 экземпляров сервисов, расположенных на разных нодах. То есть в терминах Prometheus для мониторинга этого приложения ему нужна 1 задача и 3 таргета.

Параметры задачи (job) — это прежде всего название job_name. А для описания списка таргетов воспользуемся статическим файлом настроек, то есть опишем список таргетов прямо в файле конфигурации.

В упражнении будет один таргет — это локально запущенный сервис на порту 8000. Мы его так и запишем в таргетах: '127.0.0.1'.

Также необходимо указать путь, по которому Prometheus будет забирать метрики в параметре metrics_path.

Для запуска сервиса Prometheus воспользуйтесь докером и официальным образом prom/prometheus и настройте путь к конфигурационному файлу из контейнера:

`
docker run -d --net=host --name=prometheus \
   -v /root/prometheus.yml:/etc/prometheus/prometheus.yml \
   prom/prometheus \
   --config.file=/etc/prometheus/prometheus.yml \
   --storage.tsdb.path=/prometheus \
   --web.console.libraries=/usr/share/prometheus/console_libraries \
   --web.console.templates=/usr/share/prometheus/consoles \
   --web.route-prefix=$(cat /usr/local/etc/sbercode-prefix)-9090/ \
   --web.external-url=http://127.0.0.1/$(cat /usr/local/etc/sbercode-prefix)-9090/
`{{execute}}

Проверьте, работает ли Prometheus зайдя по ссылке на [дашборд Prometheus](https://[[HOST_SUBDOMAIN]]-9090-[[KATACODA_HOST]].environments.katacoda.com/).

Если зайти в раздел Status -> Targets, то там Вы увидите список задач и таргетов: одну задачу app и один таргет в ней `http://localhost:8000/metrics`. Ровно так, как было в настройках. После первого скрейпа состояние таргета изменится с UNKNOWN на DOWN, потому что ещё не реализована сервис, который бы отдавал метрики.

![TargetDown](./assets/katacoda_prom_target_down.png)

На следующем шаге Вы реализуете сервис, воспользовавшись клиентской библиотекой от Prometheus.
