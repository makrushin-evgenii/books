Collector лёгко настраивается, для старта можно использовать базовую конфигурацию без изменений. Производителен и стабилен под разными нагрузками. Сам пример observable-сервиса. Может расширяться без изменений в коде ядра. Универсален для обработки метрик, логов и трассировок.

# Quick start
Симулировать нагрузку можно через telemetrygen.

# Install the Collector
Для обновления конфигурации нужен перезапуск.

# Deployment
- No Collector: приложение шлёт телеметрию напрямую в хранилища. Просто, но нужно менять код приложения и не все OT SDK так умеют.
- Agent: рядом с приложением запущен сайдкар Collector, который собирает и перекидывает телеметрию. Немасштабируемо, негибко (всегда 1 app : 1 coll). 
- Gateway: Collector-ы запущены как самостоятельный сервис. Высокая гибкость и масштабируемость. Возможен рост задержки доставки при последовательной работе нескольких коллекторов. Выше, относительно агента и безколекторной схемы, потребление.
Есть возможность собирать коллекторы в несколько этапов. Например, чтобы направлять телеметрию разных клиентов в определенные коллекторы:
> For use cases where the processing of the telemetry data processing has to happen in a specific collector, you would use a two-tiered setup
Можно использовать внешний балансировщик типа Nginx, можно балансирующий exporter.

# Configuration
Конфигурация определяет поток обработки. Есть валидатор конфигураций `otelcol validate --config=customconfig.yaml`.
В каждой конфигурации настраивается четыре компонента: ресиверы, процессоры, экспортеры, коннекторы. После их настройки, включаются в секции `service.pipelines`. У каждого компонента может быть несколько экземпляров, разделенных именем.
## Receivers
Могут быть push и pull. Могут поддерживать сразу несколько [источников](https://opentelemetry.io/docs/concepts/signals/). В конфигурации **должен** быть включен хотя бы один ресивер.
## Processors
Могут фильтровать, отбрасывать, пересчитывать, изменять обрабатываемые данные. В конфигурации может не быть включенных процессоров.
## Exporters
Могут быть push и pull. Могут поддерживать сразу несколько хранилищ. В конфигурации **должен** быть включен хотя бы один экспортёр.
## Connectors
Соединяют два потока. В конфигурации первого потока указывается как exporter, в конфигурации второго - как receiver.
## Service section
В этой секции включаются и выстраиваются в нужный порядок сконфигурированные выше компоненты. Так же можно установить дополнения в подсекции `extensions`, например авторизацию oidc. [И включить телеметрию самого Collector-а](https://opentelemetry.io/docs/collector/internal-telemetry/#activate-internal-telemetry-in-the-collector)

# Management
[Есть штуковина для управления агентами](https://github.com/open-telemetry/opamp-spec/blob/main/specification.md). Позволяет опрашивать и обновлять агенты. Нам не интересна, так как большеподходит gateway-деплой.

# Distributions
Если нам не подойдёт один из готовых дистрибутивов, свой нужно строить с помощью [ocb](https://opentelemetry.io/docs/collector/custom-collector/)

# Internal telemetry
По стандарту умеет писать метрики в пром и логи в стдерр. Есть три уровня детализации метрик.
*Почему-то именно тут упоминули, что большинство экспортеров используют очереди и ретраи*

# Troubleshooting
Тут собраны общие рекомендации по поддержке. [В том числе использовать очереди и тп](https://github.com/open-telemetry/opentelemetry-collector/tree/main/exporter/exporterhelper#configuration). Очереди могут храниться в памяти и в постоянном хранилище. В любом хранилище, определенном в секциях storage.

# Scaling the Collector
Нужно обрабатывать типы телеметрии в отдельных группах инстансов для независимого масштабирования. Большинство компонент не имеют состояния и легко масштабируются. Когда масштабироваться: 
- Нужно, если зашкалила метрика `otelcol_processor_refused_spans` процессора `memory_limiter` или (с условием) метрика `otelcol_exporter_queue_size` больше `otelcol_exporter_queue_capacity` - коллектор не успевает обрабатывать поток.
- Не нужно, если `otelcol_exporter_queue_size` близка к `otelcol_exporter_queue_capacity` - возможно, пропускная способность хранилища меньше коллектора. Тогда нужно масштабировать хранилище, а не коллектор.
*Тут снова упоминается схема из двух коллекторов:*
> *... when your workloads run on Kubernetes, you might want to use DaemonSets to have a Collector on the same physical node as your workloads and a remote central Collector responsible for pre-processing the data before sending the data to the storage ...*

# Transforming telemetry
Можно фильтровать, добавлять и удалять поля, переименовывать метрики (может пригодиться чтобы сохранить формат как сейчас), добавлять атрибуты ресурсов (например, для идентификации). Для сложных трансформаций есть transformprocessor

# Architecture
В случае деплоя типа gateway, в k8s node запускается DaemonSet с Collector-агентом. Он собирает телеметрию с приложений и отправляет её в Collector-сервис, который её обработает и доставит в хранилища. 

# Building a custom collector
OCB - утилита кодогенерации для сборки собственных коллекторов на Golang.

# Building custom components
... Пока не вижу смысла читать ...

# Registry
[Библиотека готовых компонент](https://opentelemetry.io/ecosystem/registry/?language=collector).
