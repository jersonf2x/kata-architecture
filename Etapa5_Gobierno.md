# Etapa 5: Gobierno y Liderazgo

*(Arquitectura Empresarial y Operación)*

---

## 5.1 Evaluación de la arquitectura

---

### Principales riesgos y cómo se mitigan

**Riesgo 1: Datos inconsistentes entre sistemas (legacy vs nuevo sistema)**
Durante la migración, puede haber dos sistemas funcionando al mismo tiempo.
Esto puede generar que un pago aparezca distinto en cada uno.

* **Impacto**: errores en saldos, pérdida de confianza, problemas regulatorios
* **Ejemplo**: un pago se registra en un sistema pero no en el otro
* **Mitigación**:

  * Uso de **SAGA** (manejo de procesos distribuidos con estados)
  * **Idempotencia** (evitar duplicados si una operación se repite)
  * **Reconciliación** (comparar sistemas y corregir diferencias)
  * Patrón **Outbox** (garantiza que los eventos se publiquen correctamente)
  * Uso de identificadores de correlación para rastrear operaciones

---

**Riesgo 2: Caída o lentitud de dependencias críticas**
Si un sistema externo (como base de datos, Kafka o un banco) falla, puede afectar todo el sistema.

* **Impacto**: interrupción del servicio o incumplimiento del SLA (99,999%)
* **Ejemplo**: Oracle lento o Kafka saturado
* **Mitigación**:

  * **Bulkhead** (aislar partes del sistema para que no afecten a otras)
  * **Circuit breaker** (cortar llamadas a servicios fallando)
  * Uso de colas para desacoplar procesos
  * **Degradación controlada** (seguir funcionando parcialmente)
  * Escalado automático (**HPA**)
  * Infraestructura altamente disponible (ej. Kafka HA)

---

**Riesgo 3: Falta de trazabilidad (auditoría y cumplimiento)**
No poder rastrear qué pasó con un pago o dónde están los datos.

* **Impacto**: problemas legales (PCI, GDPR), multas, dificultad en auditorías
* **Ejemplo**: no poder eliminar datos personales cuando un usuario lo solicita
* **Mitigación**:

  * Uso de **correlationId** para seguir cada operación
  * **Trazas distribuidas** (OpenTelemetry)
  * Políticas de manejo de datos sensibles (PII)
  * Definición de **lineaje de datos** (de dónde vienen y a dónde van)

---

### Decisión clave de arquitectura (trade-off)

Se tomó una decisión importante:

* ❌ No usar **consistencia fuerte inmediata global** (como 2PC)
* ✅ Usar:

  * **SAGA + compensaciones**
  * **Proyecciones de datos**

Esto permite tener un sistema más **escalable y resiliente**, aunque implica que la consistencia es eventual (no inmediata).

---

### Resultado de esta evaluación

* Los riesgos quedan documentados (por ejemplo en Jira)
* Cada riesgo tiene un responsable (seguridad, operaciones, datos, etc.)
* Se definen métricas claras (ej: “cero diferencias críticas en conciliación”)
* Se revisan en cada fase de migración

---

## 5.2 Gobernanza de APIs (cómo se exponen los servicios)

### Objetivo

Tener una **forma uniforme y controlada de exponer APIs**, sin importar si el backend es nuevo o legacy.

Esto aplica para:

* Canales internos
* Partners externos
* Clientes

---

### Principales políticas

**Seguridad (autenticación y autorización)**

* Uso de **OAuth2 / OIDC**
* Permisos específicos por servicio (**scopes**)
* Comunicación segura entre servicios (**mTLS**)
* Principio de **mínimo privilegio**

---

**Control de uso (carga y abuso)**

* Límites de uso (**rate limiting**) por cliente o IP
* Protección contra abusos
* Respuestas claras cuando se exceden límites (ej: código 429)

---

**Idempotencia**

* Evitar que una operación (como un pago) se procese dos veces
* Se usan claves únicas para identificar cada operación

---

**Versionado de APIs**

* Uso de versiones (ej: `/v1`, `/v2`)
* Cambios incompatibles solo en nuevas versiones
* Avisos previos antes de eliminar versiones antiguas

---

**Manejo de errores**

* Respuestas estándar
* Sin exponer información sensible

---

**Observabilidad**

* Todas las solicitudes se pueden rastrear con un `correlationId`
* Medición de tiempos y fallos
* Alertas automáticas

---

**Protección de datos**

* Cifrado en tránsito (TLS) y en almacenamiento
* Cumplimiento de estándares como PCI

---

**Documentación y descubrimiento**

* Catálogo de APIs (OpenAPI)
* Cada servicio tiene un responsable claro

---

**Integración con terceros**

* Contratos formales
* Ambientes de prueba (sandbox)
* Webhooks seguros con firma e idempotencia

---

### Consejo de API

Un pequeño grupo toma decisiones sobre APIs:

* Arquitectura
* Seguridad
* Operaciones (SRE)
* Producto

Se encarga de:

* Aprobar cambios importantes
* Revisar excepciones
* Controlar cambios incompatibles

---

## 5.3 Gobierno de datos (calidad y trazabilidad)

### Objetivo

Asegurar que los datos:

* Sean **confiables**
* Se puedan **rastrear de punta a punta**
* Cumplan regulaciones (PCI, GDPR)

Todo esto sin caer en una base de datos compartida descontrolada.

---

### Dimensiones de calidad de datos

**Exactitud**

* Datos correctos (montos, moneda, estado del pago)

**Completitud**

* Que no falte información obligatoria

**Oportunidad (timing)**

* Datos disponibles a tiempo según el uso (operación vs reportes)

**Inmutabilidad**

* No se borran registros críticos (como transacciones), se compensan

**Cumplimiento (PII)**

* Protección de datos personales
* Enmascaramiento en ambientes de prueba
* Cumplimiento legal

---

### Lineaje de datos (data lineage)

Un mismo pago puede pasar por muchos lugares:

* API
* Servicio
* Base de datos
* Eventos (Kafka)
* Reportes
* Backups

El **lineaje** permite saber:

* De dónde viene un dato
* Por dónde pasó
* Dónde está actualmente

Esto es clave para auditorías o solicitudes legales.

---

### Reglas y herramientas clave

* Uso de **contratos de datos** (Avro / JSON Schema)
* Identificadores únicos (ej: `paymentId`)
* Catálogo de datos (quién es dueño, si tiene datos sensibles, etc.)
* Procedimientos para:

  * Reprocesar datos
  * Repetir eventos (replay)
  * Eliminar o anonimizar datos según la ley

---

### Manejo por regiones

* Definir dónde viven los datos (ej: país o región principal)
* Controlar cómo se mueven entre países
* Cumplir restricciones legales de transferencia de datos

---

### Rol del liderazgo

El gobierno de datos no es solo técnico, es compartido:

* Negocio
* Seguridad (CISO)
* Datos
* Ingeniería

Para que algo salga a producción (API o evento), debe cumplir:

* Contrato definido
* Calidad mínima
* Trazabilidad básica
* Plan de retiro (si aplica)

