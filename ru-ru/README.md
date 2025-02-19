# Что случится, если ... Kubernetes edition!

Представте себе, что я хочу развернуть nginx в kubernetes кластере. Я бы набрал что то подобное в своем терминале:

```bash
kubectl create deployment nginx --image=nginx --replicas=3
```

и нажал enter. Через несколько секунд, я должен увидеть три пода nginx развернутых на моих worker nodes. Это работает автомагически, и это здорово! Но что на самом деле происходит под "капотом"?

Одна из замечательных особенностей Kubernetes заключается в том, что он управляет развертыванием workloads в инфраструктуре с помощью удобного для пользователя API. 
Сложность скрыта простыми абстракциями. Чтобы полностью понять ценность, которую kubernetes предлагает нам, нужно также понять его внутреннюю структуру.
Это руководство проведет вас через полный жизненный цикл запроса от клиента(kubectl) к kubelet, при необходимости ссылаясь на исходный код, чтобы проиллюстрировать то, что происходит.

This is a living document. If you spot areas that can be improved or rewritten, contributions are welcome!

Некоторые термины и выражения не будут переведены с английского языка, дабы не потерять смысловую свзяь с источником.

## Содержание

1. [kubectl](#kubectl)
   - [Validation and generators](#validation-and-generators)
   - [API uheggs and version negotiation](#api-groups-and-version-negotiation)
   - [Client auth](#client-auth)
1. [kube-apiserver](#kube-apiserver)
   - [Authentication](#authentication)
   - [Authorization](#authorization)
   - [Admission control](#admission-control)
1. [etcd](#etcd)
1. [Initializers](#initializers)
1. [Control loops](#control-loops)
   - [Deployments controller](#deployments-controller)
   - [ReplicaSets controller](#replicasets-controller)
   - [Informers](#informers)
   - [Scheduler](#scheduler)
1. [kubelet](#kubelet)
   - [Pod sync](#pod-sync)
   - [CRI and pause containers](#cri-and-pause-containers)
   - [CNI and pod networking](#cni-and-pod-networking)
   - [Inter-host networking](#inter-host-networking)
   - [Container startup](#container-startup)
1. [Wrap-up](#wrap-up)

## kubectl

### Validation and generators

OK, давайте начнем. Мы только что нажали Enter в нашем терминале. Что теперь?

Первое, что сделает kubectl, — это выполнит некоторую проверку на стороне клиента. Это гарантирует что некоторые запросы будут всегда завершаться неудачно (прим. создание неподдерживаемого ресурса или использование [неправильного названия image](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L264)) и не будет отправлены на kube-apiserver. Это повышает производительность системы за счет снижения ненужной нагрузки.

После проверки, kubectl начинает готовить HTTP-запрос, который отправляет на kube-apiserver. Все попытки доступа или изменения состояния в системе Kubernetes осуществляются через API серевер, который, в свою очередь, взаимодействует с etcd. Kubectl - это также API клиент. Чтобы создать HTTP-запрос, kubectl использует нечто под названием [generators](https://github.com/kubernetes/kubernetes/blob/426ef9335865ebef43f682da90796bd8bf976637/docs/devel/kubectl-conventions.md#generators) - это абстракция, которая отвечает за сериализацию.

Что может быть неочевидно, так это то, что вы действительно можете указать несколько типов ресурсов с помощью `kubectl run`, не только Deployments. Чтобы это работало, kubectl будет [подразумевать](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L319-L339) тип ресурса, если имя генератора не было указано явно с использованием флага --generator.

Например, ресурсы имеющие `--restart-policy=Always` считаются - Deployments, а те, у кого `--restart-policy=Never` считаются как - Pods. kubectl также выяснит, нужно ли запускать другие действия, такие как запись команд (for rollouts or auditing), или это команда с опцией - dry run (указан `--dry-run` флаг).

Поняв, что мы хотим создать Deployment, kubectl будет использовать `DeploymentAppsV1` генератор для генерации [runtime object](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/generate/versioned/run.go#L237) из предоставленных нами параметров. "runtime object" - это общий термин для ресурса.

### API groups and version negotiation

Прежде чем мы продолжим, стоит отметить, что Kubernetes использует версионированный API, который разделен на «группы API». Группа API предназначена для категоризации схожих ресурсов, чтобы о них было легче рассуждать. Он также обеспечивает лучшую альтернативу монолитному API. Группа API deployment называется `apps`, и его самая последняя версия `v1`. Вот почему вам нужен тип `apiVersion: apps/v1` в верхней части ваших манифестов deployment.

После того как kubectl сгенерирует runtime object, он начинает [поиск подходящей API группы и версии](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L674-L686) для этого и тогда [собирает версионный клиент](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L705-L708) который знает о различной семантике REST для искомого ресурса. Этот этап обнаружения называется согласованием версии и включает в себя сканирование kubectl `/apis` путей к удаленному API для получения всех возможных групп API. Поскольку kube-apiserver предоставляет свой документ схемы (в формате OpenAPI) по этому пути, клиентам легко выполнить собственное обнаружение.

Чтобы улучшить производительность, kubectl также [кеширует схему OpenAPI](https://github.com/kubernetes/kubernetes/blob/v1.14.0/staging/src/k8s.io/cli-runtime/pkg/genericclioptions/config_flags.go#L234) в `~/.kube/cache/discovery` директорию. Если вы хотите увидеть обнаружение API в действии, попробуйте удалить этот каталог и запустить команду с `-v` флагом. Вы увидите все HTTP-запросы, которые пытаются найти эти версии API. Их много!

Последний шаг – это фактическая [отправка](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L709) HTTP запроса. Как только это произойдет и придет успешный ответ, kubectl распечатает сообщение об успехе. [на основе желаемого формата вывода](https://github.com/kubernetes/kubernetes/blob/v1.14.0/pkg/kubectl/cmd/run/run.go#L459).

### Client auth

Одна вещь, о которой мы не упомянули на предыдущем шаге, — это аутентификация клиента (она обрабатывается до отправки HTTP-запроса), поэтому давайте посмотрим на это сейчас.

Чтобы успешно отправить запрос, kubectl должен иметь возможность аутентификации. Учетные данные пользователя почти всегда хранятся в `kubeconfig` файле, который находится на диске, но этот файл может храниться в разных местах. Чтобы найти его, kubectl делает следующее:

- если `--kubeconfig` флаг определен, он используется.
- если `$KUBECONFIG` переменная окружения опеределена, используется она.
- в противном случае посмотрит в [рекомендуемый домашний каталог](https://github.com/kubernetes/client-go/blob/release-1.21/tools/clientcmd/loader.go#L43) такой как `~/.kube`, и использует первый найденный файл.

После анализа файла kubectl определит текущий контекст, который будет использоваться, текущий кластер, на который указывает контекст, и любую информацию аутентификации, связанную с текущим пользователем.. Если пользователь предоставил специфичные флаги (такие как `--username`) они имеют приоритет и переопределяют те, которые указаны в kubeconfig. Получив эту информацию, kubectl заполняет конфигурацию клиента, чтобы он мог соответствующим образом оформить HTTP-запрос:

- x509 сертификаты отправляются с помощью [`tls.TLSConfig`](https://github.com/kubernetes/client-go/blob/82aa063804cf055e16e8911250f888bc216e8b61/rest/transport.go#L80-L89) (это также включает корневой центр сертификации)
- bearer tokens [отправляются](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L314) в "Authorization" HTTP заголовке
- имя пользователя и пароль [отправляется](https://github.com/kubernetes/client-go/blob/c6f8cf2c47d21d55fa0df928291b2580544886c8/transport/round_trippers.go#L223) через HTTP basic authentication
- процесс аутентификации OpenID заранее обрабатывается пользователем вручную, создавая токен, который отправляется как bearer tokens

## kube-apiserver

### Аутентификация

Итак, наш запрос отправлен! Что дальше? Здесь в работу включается kube-apiserver. Как мы уже упоминали, kube-apiserver - это основной интерфейс, который клиенты и компоненты системы используют для сохранения и извлечения состояния кластера. Для выполнения своей функции он должен иметь возможность проверить, что клиент является тем, за кого себя выдает. Этот процесс называется аутентификацией.

Как apiserver аутентифицирует запросы? При первом запуске сервер анализирует все [CLI флаги](https://kubernetes.io/docs/admin/kube-apiserver/) предоставленные пользователем, и составляет список подходящих аутентификаторов. Рассмотрим пример: если был передан `--client-ca-file` , он добавляет аутентификатор x509; если он обнаруживает, что указан `--token-auth-file`, он добавляет аутентификатор токена в список. Каждый раз, когда получен запрос, запрос к api серверу [проходит через цепочку аутентификаторов до тех пор, пока один из них не пройдет](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/union/union.go#L54):

- [x509 handler](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/x509/x509.go#L60) проверяет, что HTTP-запрос закодирован с ключом TLS, подписанным корневым сертификатом CA
- [bearer token handler](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/pkg/authentication/request/bearertoken/bearertoken.go#L38) проверяет, что предоставленный токен (указанный в заголовке авторизации HTTP) существует в файле на диске, указанном в `--token-auth-file`
- [basicauth handler](https://github.com/kubernetes/apiserver/blob/51bebaffa01be9dc28195140da276c2f39a10cd4/plugin/pkg/authenticator/request/basicauth/basicauth.go#L37) также обеспечивает соответствие учетных данных HTTP-запроса базовой аутентификации, своему локальному состоянию.

Если запрос не проходит проверку ни в одном из аутентификаторов, [запрос завершается неудачей](https://github.com/kubernetes/apiserver/blob/20bfbdf738a0643fe77ffd527b88034dcde1b8e3/pkg/authentication/request/union/union.go#L71) и возвращается агрегированная ошибка. Если аутентификация проходит успешно, заголовок  `Authorization` удаляется из запроса, и [информация о пользователе добавляется](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authentication.go#L71-L75) в его контекст. Это дает возможность будущим шагам, таким как авторизация и контроллеры доступа, получить доступ к ранее установленной идентичности пользователя.

### Авторизация

Хорошо, запрос отправлен, и kube-apiserver успешно проверил, что мы те, за кого себя выдаем. Мы можем быть теми, за кого себя выдаем, но есть ли  у нас разрешения на выполнение этого действия? Аутентификация и авторизация - это не одно и то же. Для того чтобы продолжить, kube-apiserver должен авторизовать нас.

Способ, которым kube-apiserver осуществляет авторизацию, очень похож на аутентификацию: на основе входных флагов он собирает цепочку авторизаторов, которые будут выполняться на каждый входящий запрос. Если все авторизаторы отклоняют запрос, то это приводит к ответу  `Forbidden` и запрос [не проходит дальше](https://github.com/kubernetes/apiserver/blob/e30df5e70ef9127ea69d607207c894251025e55b/pkg/endpoints/filters/authorization.go#L60). Если хотя бы один авторизатор разрешает запрос, то он продолжает выполнение.

Некоторые примеры авторизаторов, которые поставляются с v1.8:

- [webhook](https://github.com/kubernetes/apiserver/blob/d299c880c4e33854f8c45bdd7ab599fb54cbe575/plugin/pkg/authorizer/webhook/webhook.go#L143), который взаимодействует с внешним HTTP(S)-сервисом, расположенным вне кластера;
- [ABAC](https://github.com/kubernetes/kubernetes/blob/77b83e446b4e655a71c315ad3f3890dc2a220ccf/pkg/auth/authorizer/abac/abac.go#L223), который применяет политики, определенные в статическом файле;
- [RBAC](https://github.com/kubernetes/kubernetes/blob/8db5ca1fbb280035b126faf0cd7f0420cec5b2b6/plugin/pkg/auth/authorizer/rbac/rbac.go#L43), который применяет роли RBAC, добавленные администратором как ресурсы Kubernetes.
- [Node](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/plugin/pkg/auth/authorizer/node/node_authorizer.go#L67), который гарантирует, что kubelet(ы), могут получить доступ только к ресурсам, размещенным на самом узле.

Проверьте метод `Authorize` для каждого авторизатора, чтобы увидеть, как они работают!

### Admission control


Хорошо, к этому моменту мы уже прошли аутентификацию и были авторизованы kube-apiserver. Что осталось? С точки зрения kube-apiserver, он уверен в том, кто мы и разрешает нам продолжить, но в Kubernetes другие компоненты системы имеют четкие мнения о том, что должно и не должно происходить. Здесь в работу вступают [admission controllers](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#what-are-they).

В то время как авторизация сосредоточена на ответе на вопрос, имеет ли пользователь разрешение, контроллеры допуска перехватывают запрос, чтобы гарантировать, что он соответствует общим ожиданиям и правилам кластера. Admission controllers являются последним бастионом контроля перед сохранением объекта в etcd, поэтому они охватывают оставшиеся системные проверки, чтобы гарантировать, что действие не приводит к неожиданным или негативным результатам.

Принцип работы admission controllers похож на работу аутентификаторов и авторизаторов, но есть одно отличие: в отличие от цепочек аутентификаторов и авторизаторов, если работа одинго admission controller завершается ошибкой, вся цепочка разрушается, и запрос завершается неудачей.

Что действительно круто в дизайне admission controllers, так это фокус на их расширяемость. Каждый контроллер хранится как плагин в  [директории `plugin/pkg/admission`](https://github.com/kubernetes/kubernetes/tree/master/plugin/pkg/admission), и создан чтобы удовлетворять небольшому интерфейсу. Каждый из них затем компилируется в основной двоичный файл kubernetes.

Admission controllers обычно классифицируются на управление ресурсами, безопасность, установку значений по умолчанию и согласование ссылок. Вот несколько примеров admission controllers, которые отвечают за управление ресурсами:

- `InitialResources` устанавливает предельные значения ресурсов по умолчанию для контейнера на основе прошлого использования;
- `LimitRanger` устанавливает значения по умолчанию для реквестов и лимитов ресурсов контейнера или накладывает лимиты на определенные ресурсы (не более 2 ГБ памяти, по умолчанию 512 МБ);
- `ResourceQuota` вычисляет и отклоняет количество объектов (pods, rc, service load balancers) или общее потребление ресурсов  (cpu, memory, disk) в namespace.

## etcd

К этому моменту Kubernetes полностью проверил входящий запрос и разрешил ему продолжить свое существование. На следующем этапе kube-apiserver десериализует HTTP-запрос, конструирует из него объекты времени выполнения (что-то вроде обратного процесса генераторов kubectl), и сохраняет их в хранилище данных. Давайте разберем это немного подробнее.

Как kube-apiserver узнает, что делать, когда он принимает наш запрос? Есть довольно сложная последовательность шагов, которые происходят перед обслуживанием каких-либо запросов. Давайте начнем с начала, с момента первого запуска двоичного файла:

1. Когда запускается двоичный файл `kube-apiserver`, он [создает серверную цепочку](https://github.com/kubernetes/kubernetes/blob/1795a98eebe58fcce3b9b0a8af35d10bf91cee5b/cmd/kube-apiserver/app/server.go#L174), которая позволяет агрегации apiserver. Это, по сути, способ поддержки нескольких apiserver'ов (нас это не касается).
1. Когда это происходит, [создается общий api сервер](https://github.com/kubernetes/kubernetes/blob/1795a98eebe58fcce3b9b0a8af35d10bf91cee5b/cmd/kube-apiserver/app/server.go#L210) который служит в качестве реализации по умолчанию.
1. Сгенерированная схема OpenAPI заполняет [конфигурацию api сервера](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/config.go#L149).
1. Затем kube-apiserver итерирует по всем группам API, указанным в схеме, и настраивает [storage provider](https://github.com/kubernetes/kubernetes/blob/c7a1a061c3dc5acabcc0c35b3b96a6935dccf546/pkg/master/master.go#L410) для каждой из них, которая служит в качестве общей абстракции хранилища. Это то, к чему обращается kube-apiserver, когда он получает доступ или изменяет состояние ресурса.
1. Для каждой группы API он также итерируется по каждой из версий группы и  [устанавливает REST сопоставление](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/groupversion.go#L92) для каждого HTTP-маршрута. Это позволяет kube-apiserver сопоставлять запросы и перенаправлять их на соответствующую логику, как только он находит совпадение.
1. Для нашего конкретного случая регистрируется [POST handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/installer.go#L710) , который в свою очередь будет делегировать выполнение [обработчику создания ресурса](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37).


На этом этапе kube-apiserver полностью осведомлен о существующих маршрутах и имеет внутреннее отображение того, какие обработчики и поставщики хранилищ вызывать, если запрос совпадает. Теперь давайте представим, что наш HTTP-запрос прилетел:

1. Если цепочка обработчиков может сопоставить запрос с установленным шаблоном (т.е. с маршрутами, которые мы зарегистрировали), она [отправляется выделенному обработчику](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/handler.go#L143) зарегистрированному для этого маршрута. В противном случае используется [обработчик на основе пути](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L248) (это происходит, когда вы вызываете  `/apis`). Если для этого пути не зарегистрировано обработчиков, вызывается [not found handler](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/server/mux/pathrecorder.go#L254) что приводит к 404 ошибке.
1. К счастью для нас, у нас есть зарегистрированный маршрут под названием  [`createHandler`](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L37)! Что он делает? В первую очередь он декодирует HTTP-запрос и выполняет базовую проверку, такую, что предоставленный JSON соответствует нашему ожиданию версионированного ресурса API.
1. Проводится аудит и конечная проверка [административного доступа](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L93-L104).
1. Ресурс будет [сохранен в  etcd](https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L111) путем [делегирования к storage provider](https://github.com/kubernetes/apiserver/blob/19667a1afc13cc13930c40a20f2c12bbdcaaa246/pkg/registry/generic/registry/store.go#L327). Обычно ключ etcd будет иметь форму `<namespace>/<name>`, но это конфигурируемо.
1. Ловятся все ошибки создания, и, наконец, storage provider выполняет вызов `get`, чтобы убедиться, что объект действительно создан. Затем вызываются любые обработчики создания и декораторы, если требуется дополнительная финализация.
1. [Формируется HTTP ответ] (https://github.com/kubernetes/apiserver/blob/7001bc4df8883d4a0ec84cd4b2117655a0009b6c/pkg/endpoints/handlers/create.go#L131-L142) и отправляется клиенту.

Это много шагов! Здорово следить за ними, чтобы понять, сколько работы на самом деле выполняет apiserver. Итак, в заключение: наш deployment теперь существует в etcd. Но, вы пока не сможете его увидеть...

## Initializers

After an object is persisted to the datastore, it is not made fully visible by the apiserver or scheduled until a series of [initializers](https://kubernetes.io/docs/admin/extensible-admission-controllers/#initializers) have run. An initializer is a controller that is associated with a resource type and performs logic on the resource before it's made available to the outside world. If a resource type has zero initializers registered, this initialization step is skipped and resources are made visible immediately.

As [many great blog posts](https://ahmet.im/blog/initializers/) have pointed out, this is a powerful feature because it allows us to perform generic bootstrap operations. Examples could be:

- Injecting a proxy sidecar container into Pod that expose port 80, or feature a particular annotation.
- Injecting a volume with test certificates to all Pods in a specific namespace.
- If a Secret is shorter than 20 characters (e.g. a password), prevent its creation.

`initializerConfiguration` objects allow you to declare which initializers should run for certain resource types. Imagine we want a custom initializer to run every time a Pod is created, we'd do something like this:

```
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: InitializerConfiguration
metadata:
  name: custom-pod-initializer
initializers:
  - name: podimage.example.com
    rules:
      - apiGroups:
          - ""
        apiVersions:
          - v1
        resources:
          - pods
```

After creating this config, it will append `custom-pod-initializer` to every Pod's `metadata.initializers.pending` field. The initializer controller would already be deployed and would be routinely scanning for new Pods. When the initializer detects one with its name in the Pod's pending field, it will perform its logic. After it completes its process, it removes its name from the pending list. Only initializers whose name is first in the list may operate on the resource. When all initializers finish and the `pending` field is empty, the object will be considered initialized.

The eagle-eyed of you may have a spotted a potential problem. How can a userland controller process resources if those resources are not made visible by kube-apiserver? To get around this problem, kube-apiserver exposes a `?includeUninitialized` query parameter which returns _all_ objects, even uninitialized ones.

## Control loops

### Deployments controller

By this stage, our Deployment record exists in etcd and any initialization logic has completed. The next stages involve us setting up the resource topology that Kubernetes relies on. When we think about it, a Deployment is really just a collection of ReplicaSets, and a ReplicaSet is a collection of Pods. So how does Kubernetes go about creating this hierarchy from one HTTP request? This is where Kubernetes' built-in controllers take over.

Kubernetes makes strong use of "controllers" throughout the system. A controller is an asynchronous script that works to reconcile the
current state of the Kubernetes system to a desired state. Each controller has a small responsibility and is run in parallel by the `kube-controller-manager` component. Let's introduce ourselves to the first one that takes over, the Deployment controller.

After a Deployment record is stored to etcd and initialized, it is made visible via kube-apiserver. When this new resource is available, it is detected by the Deployment controller, whose job it is to listen out for changes to Deployment records. In our case, the controller [registers a specific callback](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L122) for create events via an informer (see below for more information about what this is).

This handler will be executed when our Deployment first becomes available and will start by [adding the object to an internal work queue](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L170). By the time it gets around to processing our object, the controller will [inspect our Deployment](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L572) and [realise](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/deployment_controller.go#L633) that there are no ReplicaSet or Pod records associated with it. It does this by querying kube-apiserver with label selectors. What's interesting to note is that this synchronization process is state agnostic: it reconciles new records in the same way as existing ones.

After realising none exist, it will begin a [scaling process](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L385) to start resolving state. It does this by rolling out (i.e. creating) a ReplicaSet resource, assigning it a label selector, and giving it the revision number of 1. The ReplicaSet's PodSpec is copied from the Deployment's manifest, as well as other relevant metadata. Sometimes the Deployment record will need to be updated after this as well (for instance if the progress deadline is set).

The [status is then updated](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/deployment/sync.go#L70) and it then re-enters the same reconciliation loop waiting for the deployment to match a desired, completed state. Since the Deployment controller is only concerned about creating ReplicaSets, this reconciliation stage needs to be continued by the next controller, the ReplicaSet controller.

### ReplicaSets controller

In the previous step, the Deployments controller created our Deployment's first ReplicaSet but we still have no Pods. This is where the ReplicaSet controller comes into play! Its job is to monitor the lifecycle of ReplicaSets and their dependent resources (Pods). Like most other controllers, it does this by triggering handlers on certain events.

The event we're interested in is creation. When a ReplicaSet is created (courtesy of the Deployments controller) the RS controller [inspects the state](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L583) of the new ReplicaSet and realizes there is a skew between what exists and what is required. It then seeks to reconcile this state by [bumping the number of pods](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L460) that belong to the ReplicaSet. It starts creating them in a careful manner, ensuring that the ReplicaSet's burst count (which it inherited from its parent Deployment) is always matched.

Create operations for Pods [are also batched](https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/replicaset/replica_set.go#L487), starting with `SlowStartInitialBatchSize` and doubling with each successful iteration in a kind of "slow start" operation. This aims to mitigate the risk of swamping kube-apiserver with unnecessary HTTP requests when there are numerous pod bootup failures (for example, due to resource quotas). If we're going to fail, we might as well fail gracefully with minimal impact on other system components!

Kubernetes enforces object hierarchies through Owner References (a field in the child resource where it references the ID of its parent). Not only does this ensure that child resources are garbage-collected once a resource managed by the controller is deleted (cascading deletion), it also provides an effective way for parent resources to not fight over their children (imagine the scenario where two potential parents think they own the same child!).

Another subtle benefit of the Owner Reference design is that it's stateful: if any controller were to restart, that downtime would not affect the wider system since resource topology is independent of the controller. This focus on isolation also creeps in to the design of controllers themselves: they should not operate on resources they don't explicitly own. Controllers should instead be selective in its ownership assertions, non-interfering, and non-sharing.

Anyway, back to owner references! Sometimes there are "orphaned" resources in the system which usually happens when:

1. a parent is deleted but not its children
2. garbage collection policies prohibit child deletion

When this occurs, controllers will ensure that orphans are adopted by a new parent. Multiple parents can race to adopt a child, but only one will be successful (the others will receive a validation error).

### Informers

As you might have noticed, some controllers like the RBAC authorizer or the Deployment controller need to retrieve cluster state to function. To return to the example of the RBAC authorizer, we know that when a request comes in, the authenticator will save an initial representation of user state for later use. The RBAC authorizer will then use this to retrieve all the roles and role bindings that are associated with the user in etcd. How are controllers supposed to access and modify such resources? It turns out this is a common use case and is solved in Kubernetes with informers.

An informer is a pattern that allows controllers to subscribe to storage events and easily list resources they're interested in. Apart from providing an abstraction which is nice to work with, it also takes care of a lot of the nuts and bolts such as caching (caching is important because it reduces unnecessary kube-apiserver connections, and reduces duplicate serialization costs server- and controller-side). By using this design, it also allows controllers to interact in a threadsafe manner without having to worry about stepping on anybody else's toes.

For more information about how informers work in relation to controllers, check out [this blog post](http://borismattijssen.github.io/articles/kubernetes-informers-controllers-reflectors-stores).

### Scheduler

After all the controllers have run, we have a Deployment, a ReplicaSet and three Pods stored in etcd and available through kube-apiserver. Our pods, however, are stuck in a `Pending` state because they have not yet been scheduled to a Node. The final controller that resolves this is the scheduler.

The scheduler runs as a standalone component of the control plane and operates in the same way as other controllers: it listens out for events and attempts to reconcile state. In this case, it [filters pods](https://github.com/kubernetes/kubernetes/blob/master/plugin/pkg/scheduler/factory/factory.go#L190) that have an empty `NodeName` field in their PodSpec and attempts to find a suitable Node that the pod can reside on.

In order to find a suitable node, a specific scheduling algorithm is used. The way the default scheduling algorithm works is the following:

1. When the scheduler starts, a [chain of default predicates are registered](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go#L65-L81). These predicates are effectively functions that, when evaluated, [filter Nodes](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L117) based on their suitability to host a pod. For example, if the PodSpec explicitly requests CPU or RAM resources, and a Node cannot meet these requests due to lack of capacity, it will be deselected for the Pod (resource capacity is calculated as the _total capacity_ minus the _sum of the resource requests_ of currently running containers).

1. Once appropriate nodes have been selected, a series of [priority functions are run](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/core/generic_scheduler.go#L354-L360) against the remaining Nodes in order to rank their suitability. For example, in order to spread workloads across the system, it will favour nodes that have fewer resource requests than others (since this indicates less workloads running). As it runs these functions, it assigns each node a numerical rank. The highest ranked node is then selected for scheduling.

Once the algorithm finds a node, the scheduler then [creates a Binding](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/scheduler.go#L336-L342) object whose Name and UID match the Pod, and whose ObjectReference field contains the name of the selected Node. This is then [sent to the apiserver](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/plugin/pkg/scheduler/factory/factory.go#L1095) via a POST request.

When kube-apiserver receives this Binding object, the registry deserializes the object and updates the following fields on the Pod object: it [sets the NodeName](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L170) to the one in the ObjectReference, it [adds any relevant annotations](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L174-L176), and [sets](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/registry/core/pod/storage/storage.go#L177-L180) its `PodScheduled` status condition to `True`.

Once the scheduler has scheduled a Pod to a Node, the kubelet on that Node can take over to begin the deployment. Exciting!

**Side note: customising the scheduler:** What's interesting is that both predicate and priority functions are extensible and can be defined by using the `--policy-config-file` flag. This introduces a degree of flexibility. Administrators can also run custom schedulers (controllers with custom processing logic) in standalone Deployments. If a PodSpec contains `schedulerName`, Kubernetes will hand over scheduling for that pod to whatever scheduler that has registered itself under that name.

## kubelet

### Pod sync

Okay, the main controller loop has finished, phew! Let's summarise: the HTTP request passed authentication, authorization, and admission control stages; a Deployment, ReplicaSet, and three Pod resources were persisted to etcd; a series of initializers ran; and, finally, each Pod was scheduled to a suitable node. So far, however, the state we've been reasoning about exists purely in etcd. The next steps involve distributing state across the worker nodes, which is the whole point of a distributed system like Kubernetes! The way this happens is through a component called the kubelet. Let's begin!

The kubelet is an agent that runs on every node in a Kubernetes cluster and is responsible for, among other things, managing the lifecycle of Pods. This means it handles all of the translation logic between the abstraction of a "Pod" (which is really just a Kubernetes concept) and its building blocks, containers. It also handles all of the associated logic around mounting volumes, container logging, garbage collection, and many more important things.

A useful way of thinking about the kubelet is again like a controller! It queries Pods from kube-apiserver every 20 seconds (this is configurable), filtering the ones whose `NodeName` [matches the name](https://github.com/kubernetes/kubernetes/blob/3b66adb8bc6929e1205bcb2bc32f380c39be8381/pkg/kubelet/config/apiserver.go#L34) of the node the kubelet is running on. Once it has that list, it detects new additions by comparing against its own internal cache and begins to synchronise state if any discrepancies exist. Let's take a look at what that synchronization process looks like:

1. If the pod is being created (ours is!), it [registers some startup metrics](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L1519) that is used in Prometheus for tracking pod latency.
1. It then [generates a PodStatus](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1287) object, which represents the state of a Pod's current Phase. The Phase of a Pod is a high-level summary of where the Pod is in its lifecycle. Examples include `Pending`, `Running`, `Succeeded`, `Failed` and `Unknown`. Generating this state is quite complicated, so let's dive into exactly what happens:
    - first, a chain of `PodSyncHandlers` is executed sequentially. Each handler checks whether the Pod should still reside on the node. If any of them decide that the Pod no longer belongs there, the Pod's phase [will change](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L1293-L1297) to `PodFailed` and it will eventually be evicted from the Node. Examples of these include evicting a Pod after its `activeDeadlineSeconds` has exceeded (used during Jobs).
    - next, the Pod's Phase is determined by the status of its init and real containers. Since our containers have not been started yet, the containers are classed as [waiting](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1244). Any Pod with a waiting container has a Phase of [`Pending`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet_pods.go#L1258-L1261).
    - finally, the Pod Condition is determined by the condition of its containers. Since none of our containers have been created by the container runtime yet, it will [set the `PodReady` condition to False](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/status/generate.go#L70-L81).
1. After the PodStatus is generated, it will then be sent to the Pod's status manager, which is tasked with asynchronously updating the etcd record via the apiserver.
1. Next, a series of admission handlers are run to ensure the pod has the correct security permissions. These include enforcing [AppArmor profiles and `NO_NEW_PRIVS`](https://github.com/kubernetes/kubernetes/blob/fc8bfe2d8929e11a898c4557f9323c482b5e8842/pkg/kubelet/kubelet.go#L883-L884). Pods denied at this stage will stay in the `Pending` state indefinitely.
1. If the `cgroups-per-qos` runtime flag has been specified, the kubelet will create cgroups for the pod and apply resource parameters. This is to enable better Quality of Service (QoS) handling for pods.
1. Data directories [are created for the pod](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L772). These include the pod directory (usually `/var/run/kubelet/pods/<podID>`), its volumes directory (`<podDir>/volumes`) and its plugins directory (`<podDir>/plugins`).
1. The volume manager will [attach and wait](https://github.com/kubernetes/kubernetes/blob/2723e06a251a4ec3ef241397217e73fa782b0b98/pkg/kubelet/volumemanager/volume_manager.go#L330) for any relevant volumes defined in `Spec.Volumes`. Depending on the type of volume being mounted, some pods will need to wait longer (e.g. cloud or NFS volumes).
1. All secrets defined in `Spec.ImagePullSecrets` are [retrieved from the apiserver](https://github.com/kubernetes/kubernetes/blob/dd9981d038012c120525c9e6df98b3beb3ef19e1/pkg/kubelet/kubelet_pods.go#L788) so that they can later be injected into the container.
1. The container runtime then runs the container (described in more detail below).

### CRI and pause containers

We're at the point now where most of the set-up is done and the container is ready to be launched. The software that does this launching is called the Container Runtime (`docker` or `rkt` are examples).

In an effort to be more extensible, the kubelet since v1.5.0 has been using a concept called CRI (Container Runtime Interface) for interacting with concrete container runtimes. In a nutshell, CRI provides an abstraction between the kubelet and a specific runtime implementation. Communication happens via [protocol buffers](https://github.com/google/protobuf) (it's like a faster JSON) and a [gRPC API](https://grpc.io/) (a type of API well-suited to performing Kubernetes operations). This is an incredibly cool idea because by using a defined contract between the kubelet and the runtime, the actual implementation details of how containers are orchestrated become largely irrelevant. All that matters is the contract. This allows new runtimes to be added with minimal overhead since no core Kubernetes code needs to change!

Enough digressions, let's get back to deploying our container... When a pod is first started, kubelet [invokes the `RunPodSandbox`](https://github.com/kubernetes/kubernetes/blob/2d64ce5e8e45e26b02492d2b6c85e5ebfb1e4761/pkg/kubelet/kuberuntime/kuberuntime_sandbox.go#L51) remote procedure command (RPC). A "sandbox" is a CRI term to describe a set of containers, which in Kubernetes parlance is, you guessed it, a Pod. The term is deliberately vague so it doesn't lose meaning for other runtimes that may not actually use containers (imagine a hypervisor-based runtime where a sandbox might be a VM).

In our case, we're using Docker. In this runtime, creating a sandbox involves creating a "pause" container. A pause container serves like a parent for all of the other containers in the Pod since it hosts a lot of the pod-level resources that workload containers will end up using. These "resources" are Linux namespaces (IPC, network, PID). If you're not familiar with how containers work in Linux, let's take a quick refresher. The Linux kernel has the concept of namespaces, which allow the host OS to carve out a dedicated set of resources (CPU or memory for example) and offer it to a process as if it's the only thing in the world using them. Cgroups are also important here, since they're the way that Linux governs resource allocation (it's kinda like a cop that polices resource usage). Docker uses both of these Kernel features to host a process that has guaranteed resources and enforced isolation. For more information, check out b0rk's amazing post: [What even is a Container?](https://jvns.ca/blog/2016/10/10/what-even-is-a-container/).

The "pause" container provides a way to host all of these namespaces and allow child containers to share them. By being a part of the same network namespace, one benefit is that containers in the same pod can refer to one another using `localhost`. The _second_ role of a pause container is related to how PID namespaces work. In these types of namespaces, processes form a hierarchical tree and the "init" process at the top takes responsibility for "reaping" dead processes. For more information on how this works, check out this [great blog post](https://www.ianlewis.org/en/almighty-pause-container). After the pause container has been created, it is checkpointed to disk, and started.

### CNI and pod networking

Our Pod now has its bare bones: a pause container which hosts all of the namespaces to allow inter-pod communication. But how does networking work and how is it set up?

When the kubelet sets up networking for a pod it delegates the task to a "CNI" plugin. CNI stands for Container Network Interface and operates in a similar way to the Container Runtime Interface. In a nutshell, CNI is an abstraction that allows different network providers to use different networking implementations for containers. Plugins are registered and the kubelet interacts with them by streaming JSON data (config files are located in `/etc/cni/net.d`) to the relevant CNI binary (located in `/opt/cni/bin`) via stdin. This is an example of the JSON configuration:

```yaml
{
    "cniVersion": "0.3.1",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
```

It also specifies additional metadata for pod, such as its name and namespace via the `CNI_ARGS` environment variable.

What happens next is dependent on the CNI plugin, but let's look at the `bridge` CNI plugin:

1. The plugin will first set up a local Linux bridge in the root network namespace to serve all containers on that host
1. It will then insert an interface (one end of a veth pair) into the pause container's network namespace and attach the other end to the bridge. The best way to think about a veth pair is like a big tube: one side is connected to the container and the other side is in the root network namespace, allowing packets to travel inbetween.
1. It should then assign an IP to the pause container's interface and set up the routes. This will result in the Pod having its own IP address. IP assignment is delegated to the IPAM providers specified to the JSON configuration.
    - IPAM plugins are similar to main network plugins: they are invoked via a binary and have a standardised interface. Each must determine the IP/subnet of the container's interface, along with the gateway and routes, and return this information back to the main plugin. The most common IPAM plugin is called `host-local` and allocates IP addresses out of a predefined set of address ranges. It stores the state locally on the host filesystem, therefore ensuring uniqueness of IP addresses on a single host.
1. For DNS, the kubelet will specify the internal DNS server IP address to the CNI plugin, which will ensure that the container's `resolv.conf` file is set appropriately.

Once the process is complete, the plugin will return JSON data back to the kubelet indicating the result of the operation.

### Inter-host networking

So far we've described how containers connect to the host, but how do hosts communicate? This will obviously happen if two Pods on different machines want to communicate.

This is usually accomplished using a concept called overlay networking, which is a way to dynamically synchronize routes across multiple hosts. One popular overlay network provider is Flannel. When installed, its core responsibility is to provide a layer-3 IPv4 network between multiple nodes in a cluster. Flannel does not control how containers are networked to the host (this is the job of CNI remember), but rather how the traffic is transported _between_ hosts. To do this, it selects a subnet for the host and registers it with etcd. It then keeps a local representation of the cluster routes and encapsulates outgoing packets in UDP datagrams, ensuring it reaches the right host. For more information, check out [CoreOS's documentation](https://github.com/coreos/flannel).

### Container startup

All the networking shenanigans are done and out of the way. What's left? Well we need to actually start out workload containers.

Once the sandbox has finished initializing and is active, the kubelet can begin creating containers for it. It first [starts any init containers](https://github.com/kubernetes/kubernetes/blob/5adfb24f8f25a0d57eb9a7b158db46f9f46f0d80/pkg/kubelet/kuberuntime/kuberuntime_manager.go#L690) as defined in the PodSpec, and will then start the main containers themselves. The process for doing this is:

1. [Pull the image](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L90) for the container. Any secrets that are defined in the PodSpec are used for private registries.
1. [Create the container](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L115) via CRI. It does this by populating a `ContainerConfig` struct (in which the command, image, labels, mounts, devices, environment variables etc. are defined) from the parent PodSpec and then sending that via protobufs to the CRI plugin. For Docker, it deserializes the payload and populates its own config structures to send to the Daemon API. In the process it applies a few metadata labels (such container type, log path, sandbox ID) to the container.
1. It then registers the container with CPU manager, which is a new alpha feature in 1.8 that assigns containers to sets of CPUs on the local node by using the `UpdateContainerResources` CRI method.
1. The container is then [started](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L135).
1. If any post-start container lifecycle hooks are registered, they are [run](https://github.com/kubernetes/kubernetes/blob/5f9f4a1c5939436fa320e9bc5973a55d6446e59f/pkg/kubelet/kuberuntime/kuberuntime_container.go#L156-L170). Hooks can either be of the type `Exec` (executes a specific command inside the container) or `HTTP` (performs a HTTP request against a container endpoint). If the PostStart hook takes too long to run, hangs, or fails, the container will never reach a `running` state.

## Wrap-up

Okay, phew. Done. Finito.

After all this, we should have 3 containers running on one or more worker nodes. All of the networking, volumes and secrets have been populated by the kubelet and made into containers via the CRI plugin.
