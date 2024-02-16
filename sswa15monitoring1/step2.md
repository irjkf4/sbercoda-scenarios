Теперь добавим логику подсчета метрик в нашем приложении.

Prometheus сам забирает метрики у приложения по протоколу HTTP. То есть, чтобы начать мониторить приложение с помощью Prometheus необходимо реализовать сервис, который бы отдавал метрики в формате Prometheus (/metrics).

Например, в сервисе нам необходимо мониторить количество запросов в секунду (RPS) и распределение времени ответа (latency).

Как посчитать эти метрики?

В Prometheus есть несколько основных типов метрик: «обычные» (меры), счётчики и гистограммы. Для того, чтобы рассчитывать распределения используется тип «Гистограмма», а для вычисления скорости изменения чего-либо (например, количество запросов в секунду — RPS) используются счётчики (counters).

Добавим файл metrics.py, в котором воспользуемся клиентской библиотекой Prometheus. Скорее всего для разных фреймворков уже существуют готовые плагины, но мы воспользуемся библиотекой, чтобы немного разобраться, как всё работает под капотом.

Откройте вкладку файла ./app/metrics.py в редакторе и введите в него код ниже, либо нажмите кнопку «Copy to Editor»:

<pre class="file" data-filename="./app/metrics.py" data-target="replace">
import time
from prometheus_client import Counter, Histogram, Info
from flask import request

METRICS_REQUEST_LATENCY = Histogram(
    "app_request_latency_seconds", "Application Request Latency", ["method", "endpoint"]
)

METRICS_REQUEST_COUNT = Counter(
    "app_request_count",
    "Application Request Count",
    ["method", "endpoint", "http_status"],
)

METRICS_INFO = Info("app_version", "Application Version")
</pre>

В данном коде мы определяем две метрики: 
* app_requst_latency_seconds — время ответа в секундах. Поскольку мы хотим считать распределение ответа и квантили (перцентили), то эта метрика должна быть гистограммой. Для каждой метрики мы можем определить набор меток для неё. Например, метки method и endpoint.

* app_request_count — это количество запросов. Поскольку мы хотим считать RPS, то это должен быть counter. И для этой метрики определим метки: method, endpoint, http_status.

В Pometheus есть неявное (неформальное) соглашение о названии метрик. Метрики типа счётчик (counter) должны заканчиваться на _total. Клиентская библиотека это сделает за нас, и в итоге, когда мы будем делать запросы к Prometheus, метрика будет называться app_requests_count_total. А гистограмма внутри Prometheus представляет собой целый набор метрик, но использовать мы будет метрику app_requst_latency_seconds_bucket.

Таким образом, мы просто определили, какие метрики мы хотим считать. Теперь перейдём к их подсчёту. Воспользуемся для этого библиотекой от Prometheus.

Создаем middleware, которое выполняется до запроса и после. До запроса запоминаем время его начала. После запроса высчитываем request_latency и увеличиваем счетчик запросов, передавая нужные метки.

Откройте закладку файла ./app/metrics.py в редакторе и добавьте код ниже, либо нажмите кнопку «Copy to Editor»:

<pre class="file" data-filename="./app/metrics.py" data-target="append">

def before_request():
    request._prometheus_metrics_request_start_time = time.time()

def after_request(response):
    request_latency = time.time() - request._prometheus_metrics_request_start_time
    METRICS_REQUEST_LATENCY.labels(request.method, request.path).observe(
        request_latency
    )
    METRICS_REQUEST_COUNT.labels(
        request.method, request.path, response.status_code
    ).inc()
    return response

def register_metrics(app, app_version=None, app_config=None):
    app.before_request(before_request)
    app.after_request(after_request)
    METRICS_INFO.info({"version": "1", "config": "develop"})

</pre>

Импортируйте middleware:

<pre class="file" data-filename="./app/app.py" data-target="insert" data-marker="from flask import Flask, abort">
from flask import Flask, abort

from metrics import register_metrics
</pre>

Регистрируйте middleware:

<pre class="file" data-filename="./app/app.py" data-target="insert" data-marker="if __name__ == '__main__':">
if __name__ == '__main__':
    register_metrics(app)
</pre>

Далее необходимо добавить сервис /metrics, по которому Prometheus будет забирать метрики. Для того, чтобы отдать данные в формате Prometheus снова используем библиотечную функцию:

<pre class="file" data-filename="./app/app.py" data-target="insert" data-marker="if __name__ == '__main__':">

@app.route('/metrics')
def metrics():
    from prometheus_client import generate_latest
    return generate_latest()

if __name__ == '__main__':
</pre>

Теперь необходимо пересобрать приложение:

`
docker build --network=host -t app:metrics app/
`{{execute}}

Запустите приложение:

`
docker kill app-v1
docker run -d --net=host --name=app-metrics app:metrics 
`{{execute}}

Проверьте, что оно работает и /probe может нам ответить:

`
curl localhost:8000/probe
`{{execute}}

Проверьте, что метрики в формате Prometheus появились, используя curl запрос:

`
curl localhost:8000/metrics
`{{execute}}


Теперь, если открыть [дашборд](https://[[HOST_SUBDOMAIN]]-9090-[[KATACODA_HOST]].environments.katacoda.com/targets) Prometheus, можно увидеть, что статус таргета стал UP.

![TargetUp](./assets/katacoda_prom_target_up.png)

На следующем шаге рассмотрим, как теперь получить данные из Prometheus c помощью PromQL.
