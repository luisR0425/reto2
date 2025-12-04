# Reto 2 – Sistema de Alerta Temprana - Monitoreo de Flota Vehicular (Serverless en AWS)

Este documento describe la arquitectura propuesta para el reto de **recibir eventos de flota**, **detectar emergencias** y **enviar correos**.

---

## 1. Visión General

El sistema expone un endpoint HTTP (`POST /event-notification`) que recibe eventos de posición y un único evento de tipo `Emergency` durante la prueba de carga con **k6**.  

La solución:

- Debe procesar **1000 eventos en 30 segundos con 100 % de éxito**.
- Debe enviar un correo de alerta para el evento `Emergency` en menos de **30 segundos** desde su recepción.
- Debe respetar los límites de **15 requests/segundo** en API Gateway y **10 instancias concurrentes** de cómputo.

---

## 2. Decisiones de Arquitectura

Las decisiones de esta solución se tomaron priorizando **desempeño (latencia/throughput)** y **disponibilidad**.

### 2.1 Estilo Arquitectónico

- **Estilo principal:** arquitectura **serverless** basada en eventos en la nube (AWS).
- **Patrón C&C dominante:** canal de mensajes síncrono (HTTP) para ingestión + invocaciones asíncronas para trabajo pesado (envío de correo).
- **Módulos bien definidos:** API de entrada, procesamiento de eventos, envío de correo y observabilidad, alineados con la recomendación de separar responsabilidades y favorecer el cambio local.

**Justificación:**  
Este estilo permite escalar automáticamente con mínima operación, absorber picos cortos de carga (prueba k6) y mantener baja la latencia percibida sin gestionar servidores.

### 2.2 Componentes y Servicios

1. **AWS API Gateway (HTTP API)**
   - Expone `POST /prod/event-notification`.
   - Configurado con `throttling_rate_limit = 15 rps` y un **burst alto (≈1000–2000)** para cumplir la restricción de 15 rps.
   - Actúa como *front door* seguro (HTTPS, validación básica).
   - Alternativas: ALB mas costoso, EC2 requiere gestion

2. **Lambda**
   - Serverless con escalado automatico. Limite de concurrencia configurable a 10 instancias.
   - Alternativas: ECS/Fargate over-engineering, EC2 costoso 24/7

4. **Amazon SES**
   - Servicio gestionado de correo saliente.
   - Evita gestionar SMTP propio y mejora confiabilidad y latencia de entrega.
   - Alternativas: SNS+Email menos control, SMTP externo mas latencia

5. **CloudWatch Logs**
   - Almacena los logs de ambas Lambdas:
     - Recepción del evento `Emergency`.
     - Envío exitoso del correo.
   - Sirve como evidencia para el entregable de **Logs de ejecución**.

6. **Amazon SQS**
   - Desacopla recepción del procesamiento. Buffer que absorbe picos sin perder mensajes.
   - Alternativas: Kinesis mas complejo, EventBridge latencia mayor

6. **IAM**
   - Roles mínimos para cada Lambda:
     - `IngestEvent`: escribir logs + invocar `SendEmergencyEmail`.
     - `SendEmergencyEmail`: escribir logs + llamar a SES.
   - Siguiendo la táctica de **“least privilege”** para seguridad.

---

## 3. Atributo de Calidad más Importante

### 3.1 Atributo priorizado: **Desempeño (Performance)**

En este reto, el objetivo principal es:

> *Procesar 1000 eventos en 30 segundos, detectar el evento de emergencia y enviar el correo en menos de 30 segundos, sin errores.*

¿Por que Rendimiento? El sistema de alerta temprana debe notificar emergencias en el menor tiempo posible. Una alerta tardia en una situacion de panico vehicular puede significar la diferencia entre una respuesta efectiva y una tragedia.

Este objetivo combina:

- **Throughput**: volumen de requests/segundo que el sistema puede soportar.
- **Latencia**: tiempo entre recepción del evento y respuesta/envío de correo.

Trade-offs considerados:
- Costo vs Velocidad: Serverless tiene mayor costo por ejecucion, pero elimina latencia de cold start prolongado.
- Complejidad vs Rendimiento: SQS anade un paso, pero garantiza no perder mensajes bajo alta carga.

### 3.2 Otros atributos relevantes

- **Disponibilidad:** no perder el evento `Emergency` y evitar errores 5xx/429 durante la prueba.
- **Escalabilidad:** soportar flota mayor en el futuro aumentando concurrencia y/o añadiendo colas/streams.
- **Modificabilidad/Integrabilidad:** poder cambiar el proveedor de correo o añadir persistencia sin reescribir la API.

---

## 4. Escenarios de Arquitectura (Quality Attribute Scenarios)

### 4.1 Escenario P1 – Throughput y latencia del endpoint

- **Fuente:** herramienta de pruebas k6 (actuando como múltiples clientes externos).
- **Estímulo:** se envían **1000 peticiones HTTP POST** con payloads válidos a `/prod/event-notification` en una ventana de **30 segundos**.
- **Entorno:** operación normal, API Gateway con `rate = 15 rps`, Lambdas desplegadas.
- **Artefacto:** API Gateway + Lambda `IngestEvent`.
- **Respuesta:**  
  - Todas las solicitudes son aceptadas y procesadas.  
  - La Lambda `IngestEvent` responde `HTTP 200` a cada request.  
  - El evento `Emergency` genera una invocación asíncrona a `SendEmergencyEmail`.
- **Medida de respuesta:**  
  - **1000/1000** respuestas 2xx, sin 4xx/5xx por throttling o errores.  
  - **Promedio de latencia** del endpoint `< 300 ms`.  
  - **P95 de latencia** `< 500 ms`.

---

### 4.2 Escenario P2 – Latencia de la notificación de emergencia

- **Fuente:** el mismo generador k6, al enviar el único evento con `type = "Emergency"`.
- **Estímulo:** llega un POST con `type = "Emergency"` y datos de vehículo/posición.
- **Entorno:** operación en modo normal, bajo carga (restantes 999 eventos de posición).
- **Artefacto:** Lambda `IngestEvent`, Lambda `SendEmergencyEmail`, SES.
- **Respuesta:**  
  - `IngestEvent` detecta el tipo `Emergency` y registra log de recepción.  
  - `IngestEvent` invoca asíncronamente `SendEmergencyEmail`.  
  - `SendEmergencyEmail` construye y envía el correo vía SES.  
  - El operador puede ver el correo en Gmail.
- **Medida de respuesta:**  
  - Tiempo entre `receivedAt` y `sentAt` en logs `< 30 segundos` (objetivo ideal \< 15s).  
  - El correo llega correctamente (sin rebotes) a la bandeja de entrada configurada.

---

### 4.3 Escenario A1 – Disponibilidad durante la prueba

- **Fuente:** mismos clientes de la prueba de carga.
- **Estímulo:** ráfaga de 1000 solicitudes en 30 s.
- **Entorno:** picos de carga de corta duración, sin fallas de infraestructura.
- **Artefacto:** API Gateway + Lambdas.
- **Respuesta:** el sistema mantiene el servicio operativo, sin caídas ni restart manual.
- **Medida de respuesta:**  
  - 100 % de disponibilidad durante la ejecución de k6.  
  - 0 errores de despliegue / timeouts de plataforma.

---

## 5. Tácticas de Arquitectura

### 5.1 Tácticas para Desempeño

1. **Introduce Concurrency**
   - Lambda procesa múltiples mensajes en paralelo en batches de 10.
   - Hasta 10 instancias simultáneas.
   - Parámetro: `Lambda Trigger Batch Size = 10`.

2. **Multiple Copies**
   - El auto-scaling de Lambda crea réplicas.
   - SQS almacena copias del mensaje hasta la confirmación.
   - Parámetro: `Reserved Concurrency = 10`.

3. **Bound Queue Sizes**
   - API Gateway limita la tasa de entrada a 15 req/s.
   - SQS absorbe picos sin saturar el sistema.
   - Parámetros: `Usage Plan Rate = 15`, `Burst = 2000`.

---

### 5.2 Tácticas para Disponibilidad y Recuperación

4. **Retry / Rollback**
   - Si Lambda falla al procesar, SQS reintenta automáticamente.
   - `Visibility Timeout = 30s`.
   - Táctica: detección de fallas + recuperación.

5. **Use an Intermediary**
   - SQS actúa como intermediario desacoplando productores de consumidores.
   - Cola de mensajes que permite desacoplamiento temporal y espacial.
   - Táctica: desacoplamiento temporal y espacial.

6. **Record/Playback**
   - CloudWatch Logs registra la hora exacta de cada evento Emergency y envío de correo.
   - Permite monitoreo y trazabilidad del flujo.
   - Táctica: monitoreo y trazabilidad.

---

## 6. Diagramas de Arquitectura (C4)

A continuación se presentan los diagramas en formato C4.

### Diagrama de Contexto:
![Diagrama de contexto](C4-Diagrama-de-arquitectura-Diagrama-de-Contexto.png)

### Diagrama de Contenedores:
![Diagrama de contexto](C4-Diagrama-de-arquitectura-Diagrama-de-Contenedores.png)

### Diagrama de Componentes:

