# Knative Eventing Demo - KCD Lima 2025

## Introducción

Esta guía te ayudará a desplegar una demo de Knative Eventing sobre Kubernetes, utilizando Kafka como sistema de mensajería. El objetivo es mostrar el flujo de eventos usando componentes de Knative y visualizar los mensajes en tiempo real. Este tutorial fue preparado para el evento [KCD Lima 2025](https://community.cncf.io/lima/), como parte de las actividades prácticas.

## Prerrequisitos

Antes de comenzar, asegúrate de tener instalado y configurado lo siguiente:

- Un clúster de Kubernetes (local o en la nube). Estas instrucciones están validadas usando Minikube
- Acceso de administrador al clúster
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [kn](https://github.com/knative/client) (Knative Client)
- Acceso a Internet para descargar los manifiestos

---

## 1. Instala Knative Serving

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.18.1/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.18.1/serving-core.yaml
```

## 2. Instala el Ingress Controller (Kourier)

```bash
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.18.0/kourier.yaml
```

### 2.1 Configura Kourier como clase de ingress por defecto

```bash
kubectl patch configmap/config-network --namespace knative-serving --type merge --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

### 2.2 Configura el dominio por defecto usando sslip.io

```bash
kubectl patch configmap/config-domain --namespace knative-serving --type merge --patch '{"data":{"127-0-0-1.sslip.io":""}}'
```

---

## 3. Prueba rápida de Kourier

### 3.1 Despliega un servicio de ejemplo

```bash
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.18.1/serving-samples/hello-world/helloworld-go.yaml
```

### 3.2 Obtén la URL del servicio

```bash
kubectl get ksvc helloworld-go
```

Busca el valor en la columna `URL`, por ejemplo:  
`http://helloworld-go.default.127-0-0-1.sslip.io`

### 3.3 Realiza una petición HTTP

```bash
curl http://helloworld-go.default.127-0-0-1.sslip.io
```

Deberías ver una respuesta similar a:

```
Hello World!
```

---

## 4. Instala Knative Eventing

```bash
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.18.2/eventing-crds.yaml
kubectl apply -f https://github.com/knative/eventing/releases/download/knative-v1.18.2/eventing-core.yaml
```

---

## 5. Instala Kafka con Strimzi

```bash
kubectl create ns kafka
kubectl apply -f https://strimzi.io/install/latest?namespace=kafka -n kafka
kubectl apply -f https://strimzi.io/examples/latest/kafka/kafka-single-node.yaml -n kafka
```

---

## 6. Instala los componentes de Knative Kafka

```bash
kubectl apply -f https://github.com/knative-extensions/eventing-kafka-broker/releases/download/knative-v1.18.0/eventing-kafka-controller.yaml
kubectl apply -f https://github.com/knative-extensions/eventing-kafka-broker/releases/download/knative-v1.18.0/eventing-kafka-channel.yaml
kubectl apply -f https://github.com/knative-extensions/eventing-kafka-broker/releases/download/knative-v1.18.0/eventing-kafka-broker.yaml
kubectl apply -f https://github.com/knative-extensions/eventing-kafka-broker/releases/download/knative-v1.18.0/eventing-kafka-source.yaml
```

---

## 7. Crea un topic en Kafka

```bash
kubectl -n kafka run kafka-topic \
  --image=quay.io/strimzi/kafka:latest-kafka-3.5.1 \
  --restart=Never --rm -it \
  -- bin/kafka-topics.sh --bootstrap-server my-cluster-kafka-bootstrap:9092 --create --topic my-topic
```

---

## 8. Despliega un Knative Service consumidor de eventos

```bash
kc apply -f - <<EOF
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: event-display
spec:
  template:
    spec:
      containers:
      - image: gcr.io/knative-releases/knative.dev/eventing-contrib/cmd/event_display
EOF
```

---

## 9. (Opcional) Despliega cloudevents-player con kn

```bash
kn service create cloudevents-player \
  --image quay.io/ruben/cloudevents-player:latest
```

---

## 10. Crea un Broker y Trigger

```bash
kubectl apply -f kafka-config-map.yaml
kubectl apply -f kafka-broker.yaml
kubectl apply -f kafka-trigger-event-display.yaml
```

---

## 11. Crea un KafkaSource apuntando al Broker

```bash
kubectl apply -f kafka-source.yaml
```

---

## 12. Envía un mensaje al topic

```bash
kubectl -n kafka run kafka-producer -it --rm --image=quay.io/strimzi/kafka:latest-kafka-3.5.1 \
  -- bin/kafka-console-producer.sh --broker-list my-cluster-kafka-bootstrap:9092 --topic my-topic
# Escribe el mensaje y presiona ENTER:
> {"message":"Hola desde la demo"}
```

---

## 13. Observa los logs del servicio consumidor

```bash
kubectl logs -l serving.knative.dev/service=event-display --tail=50 -f
```

---

## 14. Flujo de eventos con Sockeye

Antes de visualizar los eventos con `event-display`, puedes usar Sockeye para verificar que los mensajes están llegando correctamente a través de Knative Eventing y Kafka. Sockeye es una herramienta que muestra los eventos CloudEvents en tiempo real.

### 14.1 Despliega Sockeye

```bash
kubectl apply -f https://github.com/n3wscott/sockeye/releases/download/v0.8.3/release.yaml
```

### 14.2 Crea un Trigger para Sockeye

Crea un archivo llamado `kafka-trigger-sockeye.yaml` con el siguiente contenido:

```yaml
apiVersion: eventing.knative.dev/v1
kind: Trigger
metadata:
  name: sockeye-trigger
spec:
  broker: default
  filter:
    attributes:
      type: dev.knative.kafka.event
  subscriber:
    ref:
      apiVersion: v1
      kind: Service
      name: sockeye
```

Aplica el trigger:

```bash
kubectl apply -f kafka-trigger-sockeye.yaml
```

### 14.3 Visualiza los eventos en Sockeye

Obtén la URL de Sockeye:

```bash
kubectl get ksvc sockeye
```

Abre la URL en tu navegador y observa los eventos que llegan desde Kafka a través de Knative Eventing.

---

## Notas

- Puedes cambiar los nombres de los servicios y topics según tu preferencia.
- Revisa los manifiestos `kafka-trigger-*.yaml` y `kafka-source.yaml` para adaptarlos a tus necesidades.
- Más información sobre Knative: [knative.dev](https://knative.dev/)
- Más información sobre Strimzi: [strimzi.io](https://strimzi.io/)
- Más información sobre KCD Lima: https://kcd-lima-peru-2025.sessionize.com/
- Slides de la presentación: https://speakerdeck.com/hguerrero/simplificando-la-gestion-de-eventos-con-knative-
