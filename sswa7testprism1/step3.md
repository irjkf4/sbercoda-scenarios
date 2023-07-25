Стратегия развертывания **Kubernetes** определяет процедуры создания, обновления и понижения версий приложений **Kubernetes**. В традиционной программной среде развертывание или обновление приложений часто приводит к сбоям и простоям в работе. **Kubernetes** помогает избежать простоев, предоставляя различные стратегии развертывания, которые позволяют выполнять скользящие обновления на нескольких экземплярах приложений.
Стратегии развертывания **Kubernetes** поддерживают различные требования к разработке и развертыванию приложений. Каждая стратегия развертывания **Kubernetes** имеет свои преимущества, поэтому выбор подходящей стратегии зависит только от конкретных потребностей и целей.
Важно понимать, что по умолчанию в **Kubernetes** встроены только стратегии развертывания **RollingUpdate** и **Recreate**. Но и другие типы развертываний можно выполнить в **Kubernetes**, для этого потребуется выполнить дополнительные настройки или использовать специализированный инструментарий.

Теперь давайте посмотрим, как работают *стратегии развертывания*.

## Стратегия обновления RollingUpdate (Ramped)
Данная стратегия постепенного развертывания заключается в медленном развертывании новой версии приложения путем поочередной замены экземпляров до тех пор, пока не будут развернуты необходимое количество экземпляров с новой версией приложения. Обычно это происходит следующим образом: на базе пула версии A за балансировщиком нагрузки разворачивается один экземпляр версии B. Когда сервис готов к приему трафика, экземпляр добавляется в пул. Затем один экземпляр версии A удаляется из пула и выключается.

![Kubernetes Deployments](./assets/k8s-deployments-ramped.gif)
Стратегия скользящего обновления обеспечивает плавный инкрементный переход от старой версии приложения к новой. Когда запускается новый набор реплик, содержащий новую версию, реплики старой версии завершаются. В конечном итоге все **Pods** старой версии завершаются и заменяются **Pods** новой версии. Данная стратегия используется в **Kubernetes** по умолчанию.

В текущем манифесте используется **RollingUpdate**. Давайте обновим версию в манифесте на **v2**

<pre class="file" data-filename="./deployment.yaml" data-target="insert" data-marker="          image: schetinnikov/hello-app:v1">
image: schetinnikov/hello-app:v2</pre>

И применим манифест

`kubectl apply -f deployment.yaml`{{execute T1}}

Состояние деплоймента можно получить с помощью команд:

`kubectl get deploy hello-deployment `{{execute T1}}

Помотрим более подробные сведения о текущей версии приложения в запущенных **Pods**:

`kubectl describe deploy hello-deployment`{{execute T1}}

Видим информацию об образе:
`Image: shetinnikov/hello-app:v2`

Также мы можем откатить *деплоймент*. Для этого достаточно вернуть версию назад.

<pre class="file" data-filename="./deployment.yaml" data-target="insert" data-marker="          image: schetinnikov/hello-app:v2">
image: schetinnikov/hello-app:v1</pre>

И применить манифест 

`kubectl apply -f deployment.yaml`{{execute T1}}

Состояние деплоймента можно получить с помощью команд:

`kubectl get deploy hello-deployment `{{execute T1}}

Дождемся пока *деплоймент* полностью откатится.

Помотрим более подробные сведения о текущей версии приложения в запущенных **Pods**:

`kubectl describe deploy hello-deployment`{{execute T1}}

Видим информацию об образе:
`Image: shetinnikov/hello-app:v1`

## Обновление деплоймента с помощью kubectl set image и kubectl rollout undo

Мы также можем обновить версию *деплоймента* и откатить его с помощью *императивных* команд **kubectl**. 

Для обновления на новую версию можно использовать **kubectl set image**:

`kubectl set image deploy/hello-deployment hello-demo=schetinnikov/hello-app:v2`{{execute T1}}

А чтобы откатить **kubect rollout undo**:

`kubectl rollout undo deploy/hello-deployment`{{execute T1}}

В зависимости от системы, обеспечивающей постепенное развертывание, можно изменить следующие параметры для сокращения времени развертывания:

- **Parallelism, max batch size**: Количество одновременно развертываемых экземпляров.
- **Max surge**: сколько экземпляров необходимо добавить в дополнение к текущему количеству.
- **Max unavailable**: Количество недоступных экземпляров во время процедуры скользящего обновления.

## Выводы о стратегии обновления RollingUpdate (Ramped) 
**Плюсы:**
- Простота настройки.
- Постепенное распространение версий по всем экземплярам.
- Удобно для приложений с состоянием, которые обеспечивают перераспределение данных.

**Минусы:**
- Откат/переход может занимать время.
- Поддержка нескольких API затруднена.
- Отсутствие контроля над трафиком.


## Стратегия обновления Recreate

Теперь посмотрим, как работает стратегия **Recreate**
Это самый простой тип развертывания Kubernetes, который заключается в прекращении работы старых версий и последующем развертывании новых. Эта стратегия оказывает наибольшее влияние на пользователей и приводит к перебоям в работе. Кроме того, откат к предыдущей версии при необходимости занимает больше всего времени, но не требует настройки и специальных знаний от инженеров.
![Kubernetes Deployments](./assets/k8s-deployments-recreate.gif)
Стратегия воссоздания используется для систем, которые не могут функционировать в частично обновленном состоянии, или если лучше иметь время простоя, чем предоставлять пользователям менее качественные услуги.
Чем крупнее обновление, тем больше вероятность того, что при скользящем обновлении возникнет ошибка.
Поэтому для больших обновлений и модернизаций лучше использовать стратегию воссоздания.

Правим манифест

<pre class="file" data-filename="./deployment.yaml" data-target="insert" data-marker="    type: RollingUpdate">
type: Recreate</pre>

и обновляем версию 

<pre class="file" data-filename="./deployment.yaml" data-target="insert" data-marker="          image: schetinnikov/hello-app:v1">
image: schetinnikov/hello-app:v2</pre>

Применяем манифест. 

`kubectl apply -f deployment.yaml`{{execute T1}}

Состояние деплоймента можно получить с помощью команд:

`kubectl get deploy hello-deployment `{{execute T1}}

## Удаление деплоймента

Теперь можем удалить *деплоймент*:

`kubectl delete -f deployment.yaml`{{execute T1}}

Вместе с удалением *деплоймента* будут удалены все *поды*.

## Выводы о стратегии обновления Recreate

**Плюсы:**
- Простота настройки.
- Полностью обновленное состояние приложения.

**Минусы:**
- Высокий эффект на пользователя, время простоя зависит от продолжительности выключения и загрузки приложения.
