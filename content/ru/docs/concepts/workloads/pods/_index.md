---
title: "Поды"
api_metadata:
- apiVersion: "v1"
  kind: "Под"
content_type: concept
weight: 10
no_list: true
---

<!-- overview -->

_Поды_ (_Pods_) — это самые маленькие развертываемые вычислительные единицы, которые можно создавать и которыми 
можно управлять в Kubernetes.

_Под_ (от англ. «pod», что означает стаю китов или гороховый стручок) — это группа из одного или нескольких 
{{< glossary_tooltip text="контейнеров" term_id="container" >}} с общими хранилищами и сетевыми ресурсами, 
а также спецификация для запуска контейнеров.
Содержимое Пода всегда находится в одном месте, имеет совместное расписание и выполняется в общем 
контексте. Под по своей сути представляется специфическим для приложения "логическим хостом": он содержит
один или несколько контейнеров приложений, которые относительно тесно связаны между собой.
Приложения вне облачной инфраструктуры, выполняемые на одной физической или виртуальной машине, 
аналогичны облачным приложениям, выполняемым на одном логическом хосте.

Помимо контейнеров приложений Под может содержать 
{{< glossary_tooltip text="init-контейнеры" term_id="init-container" >}}, которые запускаются при 
старте Пода. Вы также можете внедрить 
{{< glossary_tooltip text="эфемерные контейнеры" term_id="ephemeral-container" >}} 
для отладки запущенного Пода.

<!-- body -->

## Что такое Под? {#what-is-a-pod}

{{< note >}}
Вам нужно установить [исполняемую среду контейнеров](/docs/setup/production-environment/container-runtimes/) 
на каждый узел кластера, чтобы Поды могли там работать.
{{< /note >}}

Общий контекст Пода - это набор пространств имен Linux, cgroups и, возможно, других аспектов изоляции - 
всего того, что изолирует и {{< glossary_tooltip text="контейнер" term_id="container" >}}. Внутри контекста Пода 
отдельные приложения могут иметь дополнительные субизоляции.

Под похож на набор контейнеров с общими пространствами имен и общими томами файловой системы.

Поды в кластере Kubernetes используются двумя основными способами:

* **Поды, которые запускают один контейнер**. Модель "один контейнер на Под" является наиболее 
  распространенным вариантом использования Kubernetes; в этом случае вы можете рассматривать Под как 
  обертку вокруг одного контейнера; Kubernetes управляет Подами, а не непосредственно контейнерами.
* **Поды, которые запускают несколько контейнеров, обязанных работать вместе**. Под может инкапсулировать 
  приложение, состоящее из [нескольких расположенных рядом контейнеров](#how-pods-manage-multiple-containers), 
  которые тесно связаны между собой и нуждаются в совместном использовании ресурсов. Эти расположенные 
  рядом контейнеры образуют единое целое.
  
  Группировка нескольких совместно расположенных и совместно управляемых контейнеров в одном Поде - это 
  довольно сложный кейс. Этот паттерн следует использовать только в особых случаях, когда 
  ваши контейнеры тесно связаны между собой.

  Запускать несколько контейнеров для обеспечения репликации (устойчивости 
  или производительности) не требуется. Если вам нужно несколько реплик, смотрите раздел 
  [Ресурсы рабочей нагрузки](/ru/docs/concepts/workloads/controllers/).

## Использование Подов {#using-pods}

Ниже приведен пример Пода, состоящего из контейнера, в котором запущен образ `nginx:1.14.2`.

{{% code_sample file="pods/simple-pod.yaml" %}}

Чтобы создать описанный выше Под, выполните следующую команду:
```shell
kubectl apply -f https://k8s.io/examples/pods/simple-pod.yaml
```

Поды обычно создаются не напрямую, а при создании ресурсов рабочей нагрузки. 
Дополнительные сведения о том, как использовать Поды с ресурсами рабочей нагрузки, см. в разделе 
[Работа с Подами](#working-with-pods).

### Ресурсы рабочей нагрузки для управления подами {#workload-resources-for-managing-pods}

Обычно не нужно создавать Поды напрямую, даже если это одиночный Под. Вместо этого они создаются 
с помощью ресурсов рабочей нагрузки, таких как 
{{< glossary_tooltip text="Deployment" term_id="deployment" >}} или 
{{< glossary_tooltip text="Job" term_id="job" >}}. Если для Подов важно хранить состояние (т.е. они stateful), 
рассмотрите ресурс {{< glossary_tooltip text="StatefulSet" term_id="statefulset" >}}.


Каждый Под предназначен для запуска одного экземпляра приложения. Если вы хотите горизонтально 
масштабировать приложение (обеспечить больше ресурсов за счет запуска большего количества экземпляров), 
следует использовать несколько Подов, по одному для каждого экземпляра. В Kubernetes это обычно называется 
_репликацией_. Реплицированные Поды обычно создаются и управляются как группа ресурсом рабочей нагрузки и его 
{{< glossary_tooltip text="контроллером" term_id="controller" >}}.

Подробнее о том, как Kubernetes использует ресурсы рабочей нагрузки и их контроллеры для масштабирования и 
автовосстановления приложений, читайте в разделе [Поды и контроллеры](#pods-and-controllers).

Поды изначально предоставляют два вида общих ресурсов для входящих в них контейнеров: [сеть](#pod-networking) 
и [хранилище](#pod-storage).

## Работа с Подами {#working-with-pods}

Создание отдельных Подов в Kubernetes — даже одиночных Подов — случается редко. Это связано 
с тем, что Поды разработаны как относительно эфемерные, одноразовые сущности. Когда Под создается 
(непосредственно вами или косвенно {{< glossary_tooltip text="контроллером" term_id="controller" >}}), 
новый Под планируется к запуску на одном из {{< glossary_tooltip text="узлов" term_id="node" >}} кластера. 
Под остается на этом узле до тех пор, пока Под не завершит выполнение, объект Пода не будет удален, 
Под не будет вытеснен из-за нехватки ресурсов, или узел не выйдет из строя.

{{< note >}}
Не следует путать перезапуск контейнера в Поде с перезапуском Пода. Под - это не процесс, а среда для 
запуска контейнера(ов). Под существует до тех пор, пока не будет удален.
{{< /note >}}

Имя Пода должно быть действительным значением 
[имени поддомена DNS](/ru/docs/concepts/overview/working-with-objects/names#dns-subdomain-names), но это 
может привести к неожиданным результатам для имени хоста Пода. Для лучшей совместимости имя должно 
соответствовать более строгим правилам для 
[имен меток DNS](/ru/docs/concepts/overview/working-with-objects/names#dns-label-names).

### ОС Пода {#pod-os}

{{< feature-state state="stable" for_k8s_version="v1.25" >}}

В поле `.spec.os.name` следует установить значение `windows` или `linux`, чтобы указать ОС, на которой 
будет работать под. На данный момент Kubernetes поддерживает только эти две операционные системы. 
В будущем этот список может быть расширен.

В Kubernetes v{{< skew currentVersion >}} значение `.spec.os.name` не влияет на  то, как
{{< glossary_tooltip text="kube-scheduler" term_id="kube-scheduler" >}} выбирает узел для запуска
Пода. В любом кластере, где на работающих узлах используются разные операционные системы, необходимо
установить правильный лейбл [kubernetes.io/os](/docs/reference/labels-annotations-taints/#kubernetes-io-os)
для каждого узла и определить поды с `nodeSelector`, указывающим на лейбл операционной системы.
Планировщик kube-scheduler распределяет поды по узлам, основываясь ещё и на других критериях,
поэтому выбор подходящего узла с ОС, которая нужна контейнерам данного пода, не всегда будет
успешным. [Pod Security Standards](/docs/concepts/security/pod-security-standards/) также используют
это поле, чтобы избежать применения политик, которые не относятся к данной операционной системе.

### Поды и контроллеры {#pods-and-controllers}

Вы можете использовать ресурсы рабочей нагрузки для создания и управления несколькими Подами. Контроллер 
ресурса управляет репликацией и развертыванием, а также автоматическим восстановлением в случае отказа 
Подов. Например, если узел выходит из строя, контроллер замечает, что Поды на этом узле перестали работать, 
и создает замену Подов. Планировщик размещает заменяющий Под на здоровый узел.

Вот несколько примеров ресурсов рабочей нагрузки, которые управляют одним или несколькими Подами:
* {{< glossary_tooltip text="Deployment" term_id="deployment" >}}
* {{< glossary_tooltip text="StatefulSet" term_id="statefulset" >}}
* {{< glossary_tooltip text="DaemonSet" term_id="daemonset" >}}

### Шаблоны Подов {#pod-templates}

Контроллеры {{< glossary_tooltip text="ресурсов рабочей нагрузки" term_id="workload" >}} создают Поды 
из _шаблонов Подов_ и управляют этими Подами в соответствии с вашими потребностями.

PodTemplates - это спецификации для создания Подов, которые включаются в ресурсы рабочей нагрузки, 
такие как [Deployments](/docs/concepts/workloads/controllers/deployment/), 
[Jobs](/docs/concepts/workloads/controllers/job/) и 
[DaemonSets](/docs/concepts/workloads/controllers/daemonset/).

Каждый контроллер для ресурса рабочей нагрузки использует шаблон `PodTemplate` внутри объекта рабочей 
нагрузки для создания реальных Подов. `PodTemplate` является частью желаемого состояния того ресурса 
рабочей нагрузки, который вы использовали для запуска своего приложения.

При создании Пода вы можете включить в шаблон Пода 
[переменные окружения](/docs/tasks/inject-data-application/define-environment-variable-container/) 
для контейнеров, запускаемых в Поде.

Приведенный ниже пример представляет собой манифест для простого Job с `template`, который запускает 
один контейнер. Контейнер в этом Поде печатает сообщение, затем приостанавливается.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: hello
spec:
  template:
    # Это шаблон Пода
    spec:
      containers:
      - name: hello
        image: busybox:1.28
        command: ['sh', '-c', 'echo "Hello, Kubernetes!" && sleep 3600']
      restartPolicy: OnFailure
    # Здесь шаблон Пода заканчивается
```

Изменение шаблона пода или переход на новый шаблон пода не оказывает прямого влияния на уже существующие 
Поды. Если вы измените pod template для ресурса рабочей нагрузки, этот ресурс должен будет создать замену 
Подам, использующим обновленный шаблон.

Например, контроллер StatefulSet следит за тем, чтобы запущенные Поды соответствовали текущему pod template 
для каждого объекта StatefulSet. Если вы отредактируете StatefulSet, чтобы изменить его pod template, 
StatefulSet начнет создавать новые Поды на основе обновленного шаблона. В конце концов, все старые 
Поды заменяются новыми, и обновление завершается.

Каждый ресурс рабочей нагрузки реализует свои собственные правила обработки изменений в шаблонах Подов. 
Если вы хотите узнать больше о StatefulSet, ознакомьтесь с 
[Updating StatefulSets](/docs/tutorials/stateful-application/basic-stateful-set/#updating-statefulsets) 
в учебном пособии "StatefulSet Basics".

На Узлах {{< glossary_tooltip term_id="kubelet" text="kubelet" >}} не знает про шаблоны Подов и связанные с ними обновления Подов,
никак не управляет ими; эти подробности для него абстрагированы. Такая абстракция и 
разделение интересов упрощает семантику системы и делает возможным расширение поведения кластера без 
изменения существующего кода.

## Обновление и замена Пода {#pod-update-and-replacement}

Как уже говорилось в предыдущем разделе, при изменении шаблона Пода для ресурса рабочей нагрузки 
контроллер создает новые Поды на основе обновленного шаблона вместо того, чтобы обновлять или 
патчить существующие Поды.

Kubernetes не мешает управлять Подами напрямую. Можно обновить некоторые поля уже работающего Пода 
прямо «на месте». Однако операции обновления Пода, такие как 
[`patch`](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#patch-pod-v1-core) и 
[`replace`](/docs/reference/generated/kubernetes-api/{{< param "version" >}}/#replace-pod-v1-core), 
имеют некоторые ограничения:

- Большинство метаданных о Поде неизменяемы. Например, вы не можете изменить поля `namespace`, `name`, 
  `uid` или `creationTimestamp`; поле `generation` уникально. Оно принимает только те обновления, 
  которые увеличивают текущее значение поля.
- Если установлено значение `metadata.deletionTimestamp`, то в список `metadata.finalizers` не может 
  быть добавлена новая запись.
- Обновления Пода не могут изменять поля, кроме `spec.containers[*].image`, 
  `spec.initContainers[*].image`, `spec.activeDeadlineSeconds` или `spec.tolerations`. 
  Для `spec.tolerations` можно только добавлять новые записи.
- При обновлении поля `spec.activeDeadlineSeconds` допускается два типа обновлений:

  1. установка положительного значения для неназначенного поля;
  2. обновление поля с положительного числа на меньшее неотрицательное число.

## Совместное использование ресурсов и коммуникация {#resource-sharing-and-communication}

Поды позволяют обмениваться данными и взаимодействовать между входящими в них контейнерами.

### Хранение в Подах {#pod-storage}

Под может определять набор общих {{< glossary_tooltip text="томов" term_id="volume" >}} хранения. 
Все контейнеры в Поде могут получить доступ к общим томам, что позволяет им обмениваться данными. 
Тома также позволяют сохранять данные в Поде в случае необходимости перезапуска одного из контейнеров. 
Дополнительные сведения о том, как Kubernetes реализует общее хранилище и делает его доступным для 
Подов, см. в разделе [Storage](/docs/concepts/storage/).

### Сеть в Подах {#pod-networking}

Каждому Поду назначается уникальный IP-адрес для каждого семейства адресов. Каждый контейнер в Поде 
разделяет сетевое пространство имен, включая IP-адрес и сетевые порты. Внутри Пода (и **только** в нем) 
контейнеры, входящие в Под, могут взаимодействовать друг с другом, используя `localhost`. Когда 
контейнеры в Поде общаются с сущностями *за пределами Пода*, они должны координировать использование 
общих сетевых ресурсов (например, портов). Внутри Пода контейнеры имеют общий IP-адрес и порт, и могут 
найти друг друга через localhost. Контейнеры в Поде также могут взаимодействовать друг с другом, 
используя стандартные межпроцессные взаимодействия, такие как семафоры SystemV или общая память POSIX. 
Контейнеры в разных Подах имеют разные IP-адреса и не могут взаимодействовать по IPC на уровне ОС без 
специальной настройки. Контейнеры, которые хотят взаимодействовать с контейнером, запущенным в другом 
Поде, могут использовать IP-сети для коммуникации.

Контейнеры в Поде воспринимают системное имя хоста как то же самое, что и сконфигурированный `name` 
для Пода. Подробнее об этом в разделе 
[Сеть в кластере](/ru/docs/concepts/cluster-administration/networking/).

## Привилегированный режим для контейнеров {#privileged-mode-for-containers}

{{< note >}}
Для того чтобы эта настройка была актуальной, ваша 
{{< glossary_tooltip text="иcполняемая среда контейнеров" term_id="container-runtime" >}} должна 
поддерживать концепцию привилегированного контейнера.
{{< /note >}}

Любой контейнер в поде может работать в привилегированном режиме, чтобы использовать 
административные возможности операционной системы, которые иначе были бы недоступны. Это доступно 
как для Windows, так и для Linux.

### Привилегированные контейнеры в Linux {#linux-privileged-containers}

В Linux любой контейнер в Поде может включить привилегированный режим с помощью флага `privileged` 
(Linux) в [контексте безопасности](/docs/tasks/configure-pod-container/security-context/) 
спецификации контейнера. Это полезно для контейнеров, которые хотят использовать возможности 
администрирования операционной системы, такие как работа с сетевым стеком или доступ к аппаратным 
устройствам.

### Привилегированные контейнеры в Windows {#windows-privileged-containers}

{{< feature-state for_k8s_version="v1.26" state="stable" >}}

В Windows вы можете создать 
[Windows HostProcess pod](/docs/tasks/configure-pod-container/create-hostprocess-pod), установив 
флаг `windowsOptions.hostProcess` в контексте безопасности спецификации пода. Все контейнеры в этих 
подах должны работать как контейнеры Windows HostProcess. HostProcess-поды запускаются 
непосредственно на хосте и могут использоваться для выполнения административных задач, как это 
делается с привилегированными контейнерами Linux.

## Статические Поды {#static-pods}

_Статические Поды_ управляются непосредственно демоном kubelet на определенном узле, без наблюдения 
со стороны {{< glossary_tooltip text="API-сервера" term_id="kube-apiserver" >}}. В то время как 
большинство Подов управляются управляющим слоем (Control Plane) (например, 
{{< glossary_tooltip text="Deployment" term_id="deployment" >}}), для статических Подов kubelet 
непосредственно контролирует каждый статический Под (и перезапускает его в случае сбоя).

Статические Поды всегда привязаны к одному {{< glossary_tooltip term_id="kubelet" >}} на определенном 
узле. Основное применение статических Подов - запуск самодостаточного управляющего слоя (Control Plane): 
другими словами, использование kubelet для контроля отдельных 
[компонентов управляющего слоя](/ru/docs/concepts/overview/components/#control-plane-components).

Kubelet автоматически пытается создать {{< glossary_tooltip text="зеркальный Под" term_id="mirror-pod" >}} 
на API-сервере Kubernetes для каждого статического Пода. Это означает, что запущенные на узле Поды 
видны на API-сервере, но управлять ими оттуда нельзя. Дополнительную информацию см. в руководстве 
[Create static Pods](/docs/tasks/configure-pod-container/static-pod).

{{< note >}}
В `spec` статического Пода нельзя ссылаться на другие объекты API (например, 
{{< glossary_tooltip text="ServiceAccount" term_id="service-account" >}}, 
{{< glossary_tooltip text="ConfigMap" term_id="configmap" >}}, 
{{< glossary_tooltip text="Secret" term_id="secret" >}} и т.д.).
{{< /note >}}

## Поды с несколькими контейнерами {#how-pods-manage-multiple-containers}

Поды предназначены для поддержки нескольких взаимодействующих процессов (в виде контейнеров), которые 
образуют единое целое. Контейнеры в Поде автоматически размещаются и планируются на одной и той же 
физической или виртуальной машине в кластере. Контейнеры могут совместно использовать ресурсы и 
зависимости, взаимодействовать друг с другом и координировать, когда и как они завершаются.

<!--intentionally repeats some text from earlier in the page, with more detail -->

Поды в кластере Kubernetes используются двумя основными способами:

* **Поды, которые запускают один контейнер**. Модель "один контейнер на Под" является наиболее 
  распространенным вариантом использования Kubernetes; в этом случае вы можете рассматривать Под как 
  обертку вокруг одного контейнера; Kubernetes управляет Подами, а не непосредственно контейнерами.
* **Поды, которые запускают несколько контейнеров, обязанных работать вместе**. Под может инкапсулировать 
  приложение, состоящее из нескольких расположенных рядом контейнеров, которые тесно связаны между 
  собой и нуждаются в совместном использовании ресурсов. Эти совместно расположенные контейнеры образуют 
  единую целостную единицу обслуживания - например, один контейнер обслуживает данные, хранящиеся в 
  общем томе, а отдельный {{< glossary_tooltip text="sidecar-контейнер" term_id="sidecar-container" >}} 
  обновляет эти файлы. Под объединяет эти контейнеры, ресурсы хранения и эфемерный сетевой идентификатор 
  в единое целое.

Например, у вас может быть контейнер, выполняющий роль веб-сервера для файлов на общем томе, и 
отдельный [sidecar-контейнер](/docs/concepts/workloads/pods/sidecar-containers/), который обновляет эти 
файлы из удаленного источника, как показано на следующей схеме:

{{< figure src="/images/docs/pod.svg" alt="Pod creation diagram" class="diagram-medium" >}}

В некоторых Подах есть {{< glossary_tooltip text="init-контейнеры" term_id="init-container" >}}, 
а также {{< glossary_tooltip text="app-контейнеры" term_id="app-container" >}}. По умолчанию 
init-контейнеры запускаются и завершаются до запуска app-контейнеров.

У вас также могут быть [sidecar-контейнеры](/docs/concepts/workloads/pods/sidecar-containers/), 
которые реализуют вспомогательные сервисы для основного приложения Пода (например, service mesh).

{{< feature-state feature_gate_name="SidecarContainers" >}}

[Feature gate](/docs/reference/command-line-tools-reference/feature-gates/) `SidecarContainers` включен по умолчанию и 
позволяет указать 
`restartPolicy: Always` для init-контейнеров. Установка политики перезапуска `Always` гарантирует, 
что контейнеры, для которых вы ее задали, будут рассматриваться как _sidecar-контейнеры_, которые 
будут работать в течение всего времени существования Пода. Контейнеры, которые вы явно определили как 
sidecar-контейнеры, запускаются раньше Пода основного приложения и остаются запущенными до тех пор, 
пока Под не будет выключен.

## Probes контейнера {#container-probes}

_Probe_ - это диагностика, периодически выполняемая kubelet для контейнера. Для проведения диагностики 
kubelet может вызывать различные действия:

- `ExecAction` (выполняется с помощью иcполняемой среды контейнеров)
- `TCPSocketAction` (проверку производит kubelet)
- `HTTPGetAction` (проверку производит kubelet)

Подробнее о [probes](/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) вы можете прочитать 
в документации Pod Lifecycle.

## {{% heading "whatsnext" %}}

* Ознакомьтесь с [жизненным циклом Пода](/docs/concepts/workloads/pods/pod-lifecycle/).
* Ознакомьтесь с [RuntimeClass](/docs/concepts/containers/runtime-class/) и с тем, как его 
  использовать для настройки различных Подов с различными конфигурациями исполняемой среды контейнеров.
* Ознакомьтесь с [PodDisruptionBudget](/docs/concepts/workloads/pods/disruptions/) и с тем, как с его 
  помощью можно управлять доступностью приложений во время сбоев.
* Под - это ресурс верхнего уровня в Kubernetes REST API. Определение объекта 
  {{< api-reference page="workload-resources/pod-v1" >}} подробно описывает данный ресурс.
* [The Distributed System Toolkit: Patterns for Composite Containers](/blog/2015/06/the-distributed-system-toolkit-patterns/) 
  объясняет общие компоновки для Подов с более чем одним контейнером.
* Ознакомьтесь с 
  [Pod topology spread constraints](/docs/concepts/scheduling-eviction/topology-spread-constraints/).

Чтобы понять, почему Kubernetes оборачивает общий API Пода в другие ресурсы (такие как 
{{< glossary_tooltip text="StatefulSets" term_id="statefulset" >}} или 
{{< glossary_tooltip text="Deployments" term_id="deployment" >}}), вы можете прочитать о 
технологиях, предшествовавших Kubernetes, включая:

* [Aurora](https://aurora.apache.org/documentation/latest/reference/configuration/#job-schema)
* [Borg](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/)
* [Marathon](https://github.com/d2iq-archive/marathon)
* [Omega](https://research.google/pubs/pub41684/)
* [Tupperware](https://engineering.fb.com/data-center-engineering/tupperware/).
