# Resumen de Estudio Completo: Prueba 4 (Sistemas Operativos y Redes)

Este documento constituye una guía de estudio integral diseñada para dominar los contenidos y competencias evaluadas en la **Prueba 4** del curso **Sistemas Operativos y Redes (ICC3201)**. El material está organizado siguiendo estrictamente el temario oficial del curso, estructurado en tres grandes partes: **Comprensión de Contenidos (40%)**, **Diseño de un Protocolo de Aplicación (35%)** y **Programación de Red en Python (25%)**.

---

# Parte 1: Comprensión de Contenidos (40%)

## 1. Sistema de Nombres de Dominio (DNS)

El **Domain Name System (DNS)** es una base de datos distribuida y jerárquica que permite traducir nombres de dominio legibles por humanos (como `www.uandes.cl`) en direcciones IP numéricas (IPv4 o IPv6) requeridas por los protocolos de la capa de red.

### 1.1 Jerarquía de Servidores DNS

El espacio de nombres de DNS está estructurado como un árbol invertido:
1. **Servidores Raíz (Root Nameservers)**: Representan la cima de la jerarquía (denotada por un punto `.`). Existen 13 direcciones IP lógicas para los servidores raíz (identificadas de la `a.root-servers.net` a la `m.root-servers.net`), implementadas geográficamente mediante la técnica de enrutamiento *anycast*. Su función principal es indicar qué servidores son autoritativos para cada dominio de nivel superior (TLD).
2. **Servidores de Dominios de Nivel Superior (TLD Nameservers)**: Administran las extensiones de primer nivel. Se dividen en:
   - **gTLD (generic TLD)**: Dominios genéricos como `.com`, `.org`, `.edu`, `.net`.
   - **ccTLD (country-code TLD)**: Dominios de código de país como `.cl` (Chile, administrado por NIC Chile), `.uk`, `.us`.
3. **Servidores Autoritativos (Authoritative Nameservers)**: Son los servidores que poseen la información oficial y actualizada de una zona específica. Por ejemplo, los servidores autoritativos de `uandes.cl` contienen los registros de recursos (`A`, `AAAA`, `MX`, etc.) para todos los nombres pertenecientes a ese dominio.

---

### 1.2 Delegación de Zonas y Registros Glue

- **Zona DNS**: Es una porción contigua del árbol de nombres cuya administración es delegada a una entidad específica. Una zona publica sus registros a través de sus servidores autoritativos.
- **Delegación**: Ocurre cuando una zona padre delega la autoridad de un subdominio a otra entidad. La zona padre no guarda los registros internos del hijo; solo publica **registros NS (Name Server)** que apuntan a los nombres de los servidores autoritativos del hijo. Por ejemplo, los servidores de la zona `.cl` responden con una referencia (referral) hacia los servidores de `uandes.cl`.

#### El Problema de la Circularidad y Registros Glue (Pegamento)
Cuando los servidores autoritativos de una zona tienen nombres que pertenecen a la misma zona que administran (por ejemplo, los servidores de `uchile.cl` se llaman `ns1.uchile.cl` y `ns2.uchile.cl`), se produce una dependencia circular:
- Para consultar a `ns1.uchile.cl`, primero se necesita su dirección IP.
- Pero su dirección IP está guardada dentro de la zona `uchile.cl`, a la cual no se puede acceder sin contactar a `ns1.uchile.cl`.

**Solución**: Los **registros Glue (pegamento)**. La zona padre (`.cl`) almacena obligatoriamente, junto con los registros `NS`, los registros `A` o `AAAA` con las direcciones IP de dichos servidores autoritativos. Si los servidores autoritativos están en un dominio externo (como `uandes.cl` usando `cloudflare.com`), no se requieren registros glue en la zona padre.

---

### 1.3 Consultas Iterativas vs. Consultas Recursivas

1. **Consulta Recursiva**: El cliente solicita la resolución a un servidor (típicamente el **resolver local** de la red o del ISP) y este asume toda la responsabilidad de encontrar la respuesta final, devolviendo el resultado exacto o un error.
2. **Consulta Iterativa**: El servidor consultado no realiza búsquedas adicionales; si no tiene la respuesta autoritativa, responde con una **referencia (referral)** al siguiente servidor jerárquico que puede tener la información. El resolver local realiza consultas iterativas sucesivas pasando por el Raíz, luego por el TLD y finalmente por el Autoritativo.

---

### 1.4 Rol del Resolver Local, Caché y TTL

- **Resolver Local**: Es un servidor intermedio que atiende las peticiones recursivas de las estaciones de trabajo de una red local o ISP.
- **Caché DNS**: Para acelerar la resolución y evitar sobrecargar la jerarquía global, el resolver almacena localmente los registros obtenidos de consultas previas.
- **Time to Live (TTL)**: Campo presente en cada registro DNS que indica, en segundos, cuánto tiempo los resolvers y clientes pueden guardar ese registro en su caché antes de tener que consultar nuevamente al servidor autoritativo.
  - **TTL Alto (ej. 86400 s / 24 hrs)**: Reduce la latencia y el tráfico de red, pero ralentiza la propagación de cambios de IP.
  - **TTL Bajo (ej. 300 s / 5 min)**: Permite cambios rápidos de dirección IP (útil en balanceo de carga o migraciones), a costa de un mayor tráfico de consultas.

---

### 1.5 Interpretación de `dig` y `dig +trace`

La herramienta CLI `dig` (Domain Information Grok) permite auditar y diagnosticar zonas DNS:

- `dig example.com A`: Realiza una consulta al resolver local configurado en el sistema operativo.
- `dig example.com +trace`: Simula el comportamiento de un resolver, realizando una **resolución iterativa manual desde los servidores raíz** paso a paso hasta llegar al autoritativo.

#### Flags y Encabezados Clave en la Salida de `dig`
- **`status: NOERROR`**: La consulta finalizó exitosamente.
- **`status: NXDOMAIN`**: Non-Existent Domain; el nombre buscado no existe en la zona autoritativa.
- **`qr` (Query/Response)**: Si está presente, el mensaje es una respuesta.
- **`aa` (Authoritative Answer)**: Indica que la respuesta proviene directamente del servidor autoritativo de la zona.
- **`rd` (Recursion Desired)**: El cliente solicitó una búsqueda recursiva.
- **`ra` (Recursion Available)**: El servidor soporta consultas recursivas.

---

### 1.6 Registros de Recursos DNS (Resource Records - RR)

Cada registro DNS se estructura con los campos: `nombre TTL clase tipo datos`.

| Tipo de Registro | Propósito y Función | Ejemplo de Contenido |
| :--- | :--- | :--- |
| **SOA** *(Start of Authority)* | Obligatorio (1 por zona). Define datos administrativos y parámetros de sincronización para servidores secundarios. | `@ IN SOA ns1.example.com. admin.example.com. ( 2026110201 7200 900 1209600 300 )` |
| **NS** *(Name Server)* | Identifica los servidores autoritativos de la zona. Presente en la propia zona y en la zona padre para la delegación. | `@ IN NS ns1.example.com.` |
| **A** | Asocia un nombre FQDN a una dirección IPv4 de 32 bits. | `www IN A 192.0.2.20` |
| **AAAA** | Asocia un nombre FQDN a una dirección IPv6 de 128 bits. | `www IN AAAA 2001:db8::20` |
| **CNAME** *(Canonical Name)* | Define un alias que apunta a otro nombre canónico. **Regla estricta**: Un nombre con CNAME no puede tener ningún otro registro (no se puede usar en el apex `@`). | `tienda IN CNAME tiendas.proveedor.net.` |
| **MX** *(Mail Exchanger)* | Especifica los servidores de correo para el dominio con una **preferencia numérica** (el valor más bajo es el servidor prioritario). El dato debe ser un FQDN, no una IP. | `@ IN MX 10 correo1.example.com.` |
| **TXT** | Contiene texto plano arbitrario. Se usa principalmente para políticas de seguridad de correo (**SPF**) y verificación de propiedad. | `@ IN TXT "v=spf1 mx -all"` |

#### Desglose de los Campos del Registro SOA
1. **Servidor Primario**: El servidor donde se edita la zona máster (`ns1.example.com.`).
2. **Correo del Responsable**: Formato con punto en lugar de arroba (`admin.example.com.` $\rightarrow$ `admin@example.com`).
3. **Serial**: Número de versión de la zona (ej. AAAAMMDDnn). Si cambia, los secundarios transfieren la zona.
4. **Refresh**: Frecuencia (en segundos) con la que el secundario consulta si el serial cambió.
5. **Retry**: Tiempo de espera antes de reintentar si el primario no responde.
6. **Expire**: Tiempo máximo que el secundario sigue respondiendo sin contactar al primario.
7. **Minimum**: TTL para respuestas negativas (caché de NXDOMAIN).

#### Registros Obligatorios al Registrar un Dominio Nuevo
- **En la Zona Propia**: Exactamente un registro `SOA` y al menos un registro `NS`. Además, se añaden los registros `A`/`AAAA` para el nombre de la zona y subdominios (`www`, etc.).
- **En la Zona Padre**: Registros `NS` para la delegación y registros `Glue` (`A`/`AAAA`) únicamente si los nombres de los servidores autoritativos están contenidos dentro de la propia zona.

---

## 2. Capa de Transporte (TCP y UDP)

La capa de transporte proporciona comunicación lógica entre procesos que se ejecutan en hosts distintos.

### 2.1 Multiplexación y Demultiplexación

- **Multiplexación en el Emisor**: Reúne datos de múltiples sockets de aplicación, añade encabezados de transporte (incluyendo puertos de origen y destino) y entrega los segmentos a la capa de red.
- **Demultiplexación en el Receptor**: Examina los campos del encabezado de transporte para entregar el segmento al socket correcto.

#### Identificación de Sockets UDP vs. TCP
- **Socket UDP**: Identificado por una **tupla de 2 elementos**: `(IP Destino, Puerto Destino)`. Cualquier datagrama que llegue con ese puerto e IP destino será entregado al mismo socket UDP, sin importar su origen.
- **Socket TCP**: Identificado por una **cuádrupla de 4 elementos**: `(IP Origen, Puerto Origen, IP Destino, Puerto Destino)`. 
  - Un servidor TCP posee un **socket de escucha (welcome socket)** asociado al puerto público (ej. 80 o 443).
  - Cada vez que una conexión es aceptada mediante `accept()`, el sistema operativo crea un **nuevo socket de conexión** específico para esa cuádrupla. Por esto, un servidor TCP puede atender miles de conexiones concurrentes en el mismo puerto local.

---

### 2.2 Comparación de Servicios: UDP vs. TCP

| Característica | UDP (User Datagram Protocol) | TCP (Transmission Control Protocol) |
| :--- | :--- | :--- |
| **Modelo de Servicio** | Sin conexión (no orientada a conexión). | Orientado a conexión (Handshake de 3 vías). |
| **Naturaleza del Tráfico** | Datagramas discretos e independientes. | Flujo continuo de bytes sin fronteras de mensaje. |
| **Confiabilidad** | No confiable (sin garantías de entrega o retransmisión). | Confiable (garantiza entrega, orden y sin duplicados mediante ACKs). |
| **Control de Flujo / Congestión**| No posee. | Sí posee (Ventana de Recepción $rwin$ y Ventana de Congestión $cwnd$). |
| **Overhead y Latencia** | Mínimo (encabezado de 8 bytes, sin retardo de handshake). | Mayor (encabezado de 20 bytes mínimo, 1.5 RTT inicial de handshake). |
| **Casos de Uso Típicos** | DNS, streaming en tiempo real, multicast/broadcast. | HTTP/HTTPS, transferencia de archivos (FTP, SSH), correo (SMTP). |

---

### 2.3 Parada y Espera (Stop-and-Wait) vs. Pipelining

- **Parada y Espera**: El emisor transmite un único segmento y debe detenerse a esperar el ACK del receptor antes de enviar el siguiente. Si la capacidad del enlace es grande o el RTT es alto, la utilización de la red es extremadamente baja.
- **Pipelining (Segmentación en Cauce)**: El emisor permite enviar múltiples segmentos consecutivos sin esperar confirmaciones individuales, manteniendo una "ventana" de paquetes en tránsito.
  - **Go-Back-N**: El receptor solo acepta segmentos en orden estricto (descarta desordenados) y envía ACKs acumulativos. Ante una pérdida, el emisor debe retransmitir todos los segmentos desde el no confirmado.
  - **Selective Repeat**: El receptor almacena en buffer los segmentos fuera de orden y acusa recibo individualmente. El emisor retransmite únicamente los segmentos perdidos.

---

### 2.4 Cálculo de Tasa de Transmisión (Throughput)

La tasa de transmisión alcanzable por un protocolo de transporte depende de la ventana de transmisión ($W$), el tamaño máximo de segmento ($MSS$) y el Round-Trip Time ($RTT$):

$$\text{Throughput} \le \frac{\text{Window Size}}{RTT} = \frac{W \times MSS}{RTT}$$

#### Ejemplo Desarrollado
Si la ventana de envío es de $W = 64\text{ KB} = 65,536\text{ bytes}$ y el $RTT = 100\text{ ms} = 0.1\text{ s}$:

$$\text{Throughput} = \frac{65,536 \times 8\text{ bits}}{0.1\text{ s}} = \frac{524,288\text{ bits}}{0.1\text{ s}} = 5,242,880\text{ bps} \approx 5.24\text{ Mbps}$$

---

### 2.5 Control de Congestión en TCP Reno

TCP regula la tasa de envío ajustando dinámicamente la ventana de congestión ($cwnd$), de modo que la ventana efectiva sea $\min(cwnd, rwin)$.

#### Las Fases Principales
1. **Partida Lenta (Slow Start)**:
   - Al iniciar la conexión, $cwnd = 1\text{ MSS}$.
   - Por cada ACK recibido que confirme nueva data, $cwnd$ se incrementa en $1\text{ MSS}$. Esto genera un **crecimiento exponencial** de $cwnd$ (se duplica cada RTT).
   - Termina cuando $cwnd \ge ssthresh$ o ante un evento de pérdida.

2. **Evitación de Congestión (Congestion Avoidance)**:
   - Se activa cuando $cwnd \ge ssthresh$.
   - Crecimiento lineal: $cwnd$ se incrementa en $1\text{ MSS}$ por cada RTT completo (aproximadamente $cwnd = cwnd + \frac{MSS \times MSS}{cwnd}$ por cada ACK).

3. **Reacción ante Eventos de Pérdida (Congestión)**:
   - **Mild Congestion (3 ACKs Duplicados / Fast Retransmit)**:
     - Indica que se perdió un segmento, pero los siguientes siguen llegando al receptor.
     - TCP realiza **retransmisión rápida** del segmento faltante.
     - $ssthresh = \max\left(\frac{cwnd}{2}, 2 \times MSS\right)$.
     - En TCP Reno, $cwnd = ssthresh$ (entra directo a Congestion Avoidance, conocido como Fast Recovery).
   - **Severe Congestion (Expiración del Temporizador / RTO Timeout)**:
     - Indica congestión grave en la red (no llegan ni los ACKs).
     - Retransmite el primer segmento no confirmado.
     - $ssthresh = \max\left(\frac{cwnd}{2}, 2 \times MSS\right)$.
     - $cwnd$ se reinicia a $1\text{ MSS}$ y TCP vuelve a iniciar **Slow Start**.

---

## 3. Capa de Red e IPv4

La capa de red se encarga del direccionamiento lógico y del traslado de datagramas desde el host emisor hasta el host receptor a través de múltiples enlaces de red.

### 3.1 Notación CIDR, Máscara Decimal y Rangos de Direcciones

Una dirección IPv4 posee 32 bits representados en cuatro octetos decimales (ej. `172.16.45.130`). La **notación CIDR** (Classless Inter-Domain Routing) indica mediante `/p` la cantidad de bits asignados al prefijo de red, mientras que los $h = 32 - p$ bits restantes corresponden a los hosts.

#### Conversión de Prefijos a Máscaras Decimales
Para obtener la máscara decimal a partir del prefijo `/p`:
- Los octetos completos de 8 bits valen `255`.
- El octeto donde termina el prefijo se calcula sumando las potencias de 2 asociadas a los bits en 1 de izquierda a derecha.

| Bits en 1 en Octeto | Binario | Valor Decimal del Octeto |
| :---: | :---: | :---: |
| 1 | `10000000` | 128 |
| 2 | `11000000` | 192 |
| 3 | `11100000` | 224 |
| 4 | `11110000` | 240 |
| 5 | `11111000` | 248 |
| 6 | `11111100` | 252 |
| 7 | `11111110` | 254 |
| 8 | `11111111` | 255 |

*Ejemplos*:
- `/20` ($8 + 8 + 4$ bits): `255.255.240.0`
- `/26` ($8 + 8 + 8 + 2$ bits): `255.255.255.192`

---

### 3.2 Dirección de Red, Broadcast y Hosts Asignables

Para cualquier subred con prefijo `/p` y $h = 32 - p$ bits de host:
1. **Dirección de Red**: Todos los $h$ bits de host valen `0`. Se obtiene mediante la operación binaria `IP AND Máscara`. No es asignable a un host.
2. **Dirección de Broadcast**: Todos los $h$ bits de host valen `1`. Todo datagrama enviado a ella es recibido por todos los hosts de la subred. No es asignable a un host.
3. **Hosts Asignables**: Total de direcciones utilizables = $2^h - 2$.
4. **Rango de Hosts Asignables**: Desde `(Dirección de Red + 1)` hasta `(Dirección de Broadcast - 1)`.

#### Atajo del Tamaño de Bloque
El tamaño de bloque en el octeto donde termina el prefijo se calcula como:

$$\text{Tamaño de Bloque} = 256 - \text{Valor del octeto de máscara}$$

Las subredes dentro de ese octeto siempre comienzan en múltiplos exactos del tamaño de bloque.

#### Ejemplo Resuelto Completo
Dada la dirección `172.16.45.130/26`:
- Prefijo $p = 26 \Rightarrow h = 32 - 26 = 6$ bits de host.
- Máscara: `255.255.255.192`.
- Octeto en división: cuarto octeto.
- Tamaño de bloque: $256 - 192 = 64$.
- Múltiplos del bloque en el cuarto octeto: `0`, `64`, `128`, `192`.
- La IP `.130` cae en el bloque que inicia en `128`.
- **Dirección de Red**: `172.16.45.128`
- **Dirección de Broadcast**: $128 + 64 - 1 = 191 \Rightarrow$ `172.16.45.191`
- **Rango Asignable**: `172.16.45.129` a `172.16.45.190`
- **Hosts Asignables**: $2^6 - 2 = 64 - 2 = 62\text{ hosts}$

---

### 3.3 División de Redes mediante VLSM (Variable Length Subnet Masking)

El diseño tradicional asigna bloques de tamaño fijo (FLSM), generando un alto desperdicio de direcciones IP (por ejemplo, en enlaces punto a punto entre routers donde solo se necesitan 2 IPs). **VLSM** permite aplicar máscaras de longitud variable ajustadas exactamente a la necesidad de cada subred.

#### Algoritmo Paso a Paso para Resolver Ejercicios de VLSM
1. **Determinar los bits de host $h$ requeridos para cada subred**: Buscar el menor entero $h$ tal que $2^h - 2 \ge \text{Hosts requeridos}$. El nuevo prefijo será $p = 32 - h$.
2. **Ordenar las subredes de MAYOR a MENOR capacidad requerida**. Esto es estrictamente obligatorio para asegurar que cada bloque comience en un múltiplo natural de su tamaño sin solapamientos ni fragmentación.
3. **Asignar direcciones consecutivamente**: Empezar desde la IP base dada e ir sumando el tamaño de bloque correspondiente a cada subred.

#### Caso Práctico Resuelto (Estilo Evaluación)
*Enunciado*: Dada la red base `192.168.0.0/24`, diseñar un esquema VLSM para:
- Subred A (Estudiantes): 80 hosts.
- Subred B (Profesores): 20 hosts.
- Subred C (Invitados): 10 hosts.
- Subred D (Enlace Router-Router): 2 hosts.

*Solución paso a paso*:
1. **Subred A (80 hosts)**:
   - $2^h - 2 \ge 80 \Rightarrow h = 7$ ($2^7 - 2 = 126 \ge 80$).
   - Prefijo: $32 - 7 = /25$. Máscara: `255.255.255.128`. Bloque: 128.
   - **IP Red**: `192.168.0.0/25`
   - **Rango**: `192.168.0.1` a `192.168.0.126`
   - **Broadcast**: `192.168.0.127`

2. **Subred B (20 hosts)**:
   - Siguiente IP disponible: `192.168.0.128`.
   - $2^h - 2 \ge 20 \Rightarrow h = 5$ ($2^5 - 2 = 30 \ge 20$).
   - Prefijo: $32 - 5 = /27$. Máscara: `255.255.255.224`. Bloque: 32.
   - **IP Red**: `192.168.0.128/27`
   - **Rango**: `192.168.0.129` a `192.168.0.158`
   - **Broadcast**: `192.168.0.159`

3. **Subred C (10 hosts)**:
   - Siguiente IP disponible: `192.168.0.160`.
   - $2^h - 2 \ge 10 \Rightarrow h = 4$ ($2^4 - 2 = 14 \ge 10$).
   - Prefijo: $32 - 4 = /28$. Máscara: `255.255.255.240`. Bloque: 16.
   - **IP Red**: `192.168.0.160/28`
   - **Rango**: `192.168.0.161` a `192.168.0.174`
   - **Broadcast**: `192.168.0.175`

4. **Subred D (2 hosts - Enlace WAN)**:
   - Siguiente IP disponible: `192.168.0.176`.
   - $2^h - 2 \ge 2 \Rightarrow h = 2$ ($2^2 - 2 = 2 \ge 2$).
   - Prefijo: $32 - 2 = /30$. Máscara: `255.255.255.252`. Bloque: 4.
   - **IP Red**: `192.168.0.176/30`
   - **Rango**: `192.168.0.177` a `192.168.0.178`
   - **Broadcast**: `192.168.0.179`

---

### 3.4 Reenvío (Forwarding) vs. Enrutamiento (Routing)

| Dimensión | Reenvío (Forwarding) | Enrutamiento (Routing) |
| :--- | :--- | :--- |
| **Definición** | Acción local de trasladar un paquete desde una interfaz de entrada a una interfaz de salida en un router. | Proceso global que calcula las rutas completas de extremo a extremo a través de la red. |
| **Plano de Red** | **Plano de Datos (Data Plane)**. | **Plano de Control (Control Plane)**. |
| **Escala de Tiempo** | Extremadamente rápido (nanosegundos a microsegundos), implementado habitualmente en hardware (ASIC/TCAM). | Más lento (segundos a minutos), ejecutado por software. |
| **Estructura** | Utiliza la **Tabla de Reenvío (Forwarding Table)**. | Construye la **Tabla de Enrutamiento (Routing Table)** mediante protocolos como OSPF, BGP o rutas estáticas. |

#### Regla del Prefijo Más Largo (Longest Prefix Match - LPM)
Cuando un datagrama llega a un router, este consulta la IP de destino en su tabla de reenvío. Si la IP coincide con múltiples entradas, el router **selecciona strictly la entrada que posea el prefijo más largo (`/p` mayor)**, ya que representa la ruta más específica.

---

# Parte 2: Diseño de un Protocolo de Aplicación (35%)

En la Parte 2 de la prueba se solicita especificar formalmente un protocolo de capa de aplicación a partir de un escenario dado (ej. sistema de transferencias, billetera digital, pedidos de comida, pizarra compartida).

---

## 1. Metodología de Especificación de Protocolos

Para obtener el puntaje completo, la especificación debe incluir 8 secciones fundamentales:

### 1.1 Identificación de Roles y Tipos de Host
Define de forma clara qué entidades participan en la red:
- **Cliente / Emisor / Escritor**: Inicia las interacciones o genera los datos.
- **Servidor / Receptor / Observador**: Atiende peticiones, mantiene el estado global o procesa transacciones.

### 1.2 Elección del Servicio de Transporte (TCP vs. UDP)
Debe justificarse técnicamente la elección:
- **TCP**: Para mensajes de control, autenticación, transacciones financieras o transferencias de archivos donde se exige cero pérdida, ordenamiento y confiabilidad.
- **UDP (Broadcast/Multicast)**: Para descubrimiento de pares en la red local o difusión de estado en tiempo real (ej. trazos de pizarra) con baja latencia.

### 1.3 Patrones de Interacción
- **Solicitud-Respuesta (Request-Response)**: Un extremo envía una petición y espera una confirmación o respuesta obligatoria antes de continuar.
- **Mensajes No Solicitados (Unsolicited / Push)**: Un extremo difunde anuncios periódicos o eventos sin requerir una solicitud previa.

### 1.4 Definición Exhaustiva de Tipos de Mensajes
Cada mensaje debe documentarse con:
1. **Nombre Simbólico**: (ej. `ANNOUNCE`, `OFFER`, `ACCEPT`, `REJECT`, `DATA`, `DONE`, `ERROR`).
2. **Dirección y Transporte**: (ej. TCP Cliente $\rightarrow$ Servidor).
3. **Descripción Funcional**.
4. **Campos**: Nombre, tipo de dato, significado.
5. **Ejemplo Codificado**: Sintaxis exacta (ej. JSON).

### 1.5 Formato y Delimitación de Mensajes en TCP
Como TCP es un flujo continuo de bytes, el diseño **debe especificar cómo se delimitan los mensajes**:
- **Opción A (Líneas terminadas en `\n`)**: Para mensajes de control legibles.
- **Opción B (Encabezado de Largo Explícito)**: Preceder cada mensaje JSON con un entero sin signo de 4 bytes en orden de red (*big-endian*).
- **Transporte de Datos Binarios**: Encapsular o separar los bytes crudos de los archivos indicando `chunk_size` explícito en un encabezado `DATA_HEADER` para no romper el enmarcado de control.

### 1.6 Diagramas y Tablas de Estados
Especificar la máquina de estados finitos (FSM) del cliente y del servidor.

- **IDLE** $\rightarrow$ Enviar `OFFER` $\rightarrow$ **WAITING_RESPONSE**
- **WAITING_RESPONSE** $\rightarrow$ Recibir `ACCEPT` $\rightarrow$ **SENDING**
- **WAITING_RESPONSE** $\rightarrow$ Recibir `REJECT` / Timeout $\rightarrow$ **REJECTED / FAILED**
- **SENDING** $\rightarrow$ Enviar último bloque $\rightarrow$ **COMPLETED**

### 1.7 Manejo de Excepciones y Errores
Definir la respuesta ante: mensajes mal formados, timeouts de red, desconexiones abruptas, falta de espacio en disco o recursos agotados (enviando mensajes `ERROR` o aplicando *rollback* de transacciones).

### 1.8 Ejemplo de Traza de Comunicación
Mostrar una secuencia paso a paso indicando el intercambio de mensajes entre los extremos para un caso exitoso y para un caso de error.

---

# Parte 3: Programación de una Aplicación de Red en Python (25%)

La Parte 3 evalúa la capacidad de implementar o completar programas de sockets en Python 3 utilizando la biblioteca estándar (`socket`, `threading`, `struct`, `json`, `os`).

---

## 1. La API de Sockets en Python

### Resumen de Funciones Clave de la API
- `socket.socket(family, type)`: Crea un socket. Para IPv4 se usa `AF_INET`; para TCP `SOCK_STREAM` y para UDP `SOCK_DGRAM`.
- `sock.bind((ip, puerto))`: Asocia el socket a una interfaz e IP local (`""` escucha en todas las interfaces).
- `sock.listen()`: Habilita el socket TCP para escuchar conexiones entrantes.
- `sock.accept()`: Bloquea hasta recibir una conexión TCP. Retorna `(conn_sock, client_addr)`.
- `sock.recv(n)`: Lee hasta $n$ bytes de un socket TCP. **Retorna `b""` si el remoto cerró la conexión**.
- `sock.sendall(datos)`: Envía **todos** los bytes garantizando la transmisión completa. (No usar `send`).
- `sock.recvfrom(n)`: Lee un datagrama UDP de hasta $n$ bytes. Retorna `(datos, origen_addr)`.
- `sock.sendto(datos, (ip, puerto))`: Envía un datagrama UDP al destino.
- `sock.setsockopt(SOL_SOCKET, SO_REUSEADDR, 1)`: Permite reutilizar inmediatamente un puerto local al reiniciar el servidor.
- `sock.setsockopt(SOL_SOCKET, SO_BROADCAST, 1)`: Permite transmitir paquetes de difusión UDP.
- `sock.settimeout(segundos)`: Configura un tiempo límite para operaciones bloqueantes, lanzando `socket.timeout`.

---

## 2. El Desafío del Flujo de Bytes en TCP y la Clase Wrapper `Conexion`

En TCP no existen fronteras de mensajes. Tres llamadas a `sock.sendall()` en el emisor pueden llegar consolidadas en una sola llamada a `sock.recv()` en el receptor. 

Para resolver esto, se envuelve el socket TCP en una clase wrapper con un buffer (`bytearray`), tal como se presenta en la guía del curso (`conexion.py`):

```python
import struct

class Conexion:
    """Envuelve un socket TCP y mantiene un buffer con los bytes no procesados."""
    def __init__(self, sock):
        self.sock = sock
        self.buf = bytearray()

    def _llenar(self):
        datos = self.sock.recv(4096)
        if not datos:
            raise ConnectionError("El remoto cerró la conexión")
        self.buf += datos

    def leer_linea(self, max_largo=4096):
        """Lee y retorna una línea terminada en b'\\n', decodificada como UTF-8."""
        while b"\n" not in self.buf:
            if len(self.buf) > max_largo:
                raise ValueError("Línea demasiado larga")
            self._llenar()
        linea, _, resto = self.buf.partition(b"\n")
        self.buf = bytearray(resto)
        return linea.decode("utf-8")

    def leer_exacto(self, n):
        """Retorna exactamente n bytes, bloqueando hasta completarlos."""
        while len(self.buf) < n:
            self._llenar()
        datos = bytes(self.buf[:n])
        del self.buf[:n]
        return datos

    def copiar_a(self, f, n, bloque=64 * 1024):
        """Copia exactamente n bytes desde la conexión hacia un archivo f por bloques."""
        if self.buf:
            usar = min(n, len(self.buf))
            f.write(self.buf[:usar])
            del self.buf[:usar]
            n -= usar
        while n > 0:
            datos = self.sock.recv(min(bloque, n))
            if not datos:
                raise ConnectionError("Conexión interrumpida durante la copia")
            f.write(datos)
            n -= len(datos)

def enviar_mensaje_largo_explicito(sock, datos: bytes):
    """Envía un mensaje precedido por su longitud en 4 bytes (Big-Endian)."""
    encabezado = struct.pack("!I", len(datos))
    sock.sendall(encabezado + datos)

def recibir_mensaje_largo_explicito(conn: Conexion, max_largo=1024*1024):
    """Recibe un mensaje estructurado con enviar_mensaje_largo_explicito."""
    encabezado_bytes = conn.leer_exacto(4)
    (largo,) = struct.unpack("!I", encabezado_bytes)
    if largo > max_largo:
        raise ValueError("Mensaje excede el tamaño máximo permitido")
    return conn.leer_exacto(largo)
```

---

## 3. Servidor de Archivos Concurrente y Robusto (Multithreading)

A continuación se presenta la implementación de referencia para un servidor de archivos concurrente que procesa solicitudes `PUT` con encabezados JSON y transfiere archivos binarios en bloques de 64 KB sin recargar la memoria RAM (`p6_archivos.py`):

```python
import json
import os
import socket
import sys
import threading
from conexion import Conexion

PORT = 50006
DESTINO = "subidos"
MAX_SIZE = 1024 * 1024 * 1024  # Límite máximo: 1 GB
TIMEOUT = 10.0                 # Timeout de inactividad

def responder_error(sock, addr, motivo):
    print(f"[{addr}] Error en solicitud: {motivo}")
    try:
        sock.sendall(f"ERR {motivo}\n".encode("utf-8"))
    except OSError:
        pass  # El cliente ya se desconectó

def atender_cliente(sock, addr):
    sock.settimeout(TIMEOUT)
    conn = Conexion(sock)
    path = None
    try:
        linea = conn.leer_linea()
        verbo, _, payload = linea.partition(" ")
        if verbo != "PUT":
            raise ValueError("Se esperaba el comando PUT")
        
        meta = json.loads(payload)
        # Sanitización de seguridad contra Path Traversal (ej. ../../etc/passwd)
        name = os.path.basename(str(meta["name"]))
        size = meta["size"]
        
        if not name or not isinstance(size, int) or not (0 <= size <= MAX_SIZE):
            raise ValueError("Metadatos de archivo inválidos")
            
        path = os.path.join(DESTINO, name)
        with open(path, "wb") as f:
            conn.copiar_a(f, size)
            
        sock.sendall(b"OK\n")
        print(f"[{addr}] Archivo recibido con éxito: {name} ({size} bytes)")
        path = None  # Indica recepción exitosa sin limpieza final
        
    except KeyError as e:
        responder_error(sock, addr, f"Falta el campo obligatorio: {e}")
    except (ValueError, TypeError) as e:
        responder_error(sock, addr, str(e))
    except (ConnectionError, socket.timeout) as e:
        print(f"[{addr}] Error de comunicación: {e}")
    finally:
        # Si la transferencia falló o fue cancelada, se elimina el archivo incompleto
        if path is not None and os.path.exists(path):
            os.remove(path)
        sock.close()

def servidor():
    os.makedirs(DESTINO, exist_ok=True)
    srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    srv.bind(("", PORT))
    srv.listen()
    print(f"Servidor concurrente TCP escuchando en puerto {PORT}...")
    
    while True:
        sock, addr = srv.accept()
        # Se genera un hilo independiente por cada cliente para lograr concurrencia
        threading.Thread(target=atender_cliente, args=(sock, addr), daemon=True).start()

if __name__ == "__main__":
    servidor()
```

---

## 4. Descubrimiento de Dispositivos por Difusión UDP (Broadcast)

Programa para anunciar y mantener la lista de pares activos en la LAN mediante UDP Broadcast (`p8_anuncios.py`):

```python
import json
import socket
import time

PORT = 50008
INTERVALO = 2.0
EXPIRACION = 3 * INTERVALO

def anunciador(alias):
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_BROADCAST, 1)
    msg = json.dumps({"alias": alias}).encode("utf-8")
    print(f"Iniciando anuncios UDP broadcast como '{alias}'...")
    while True:
        sock.sendto(msg, ("<broadcast>", PORT))
        time.sleep(INTERVALO)

def oyente():
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(("", PORT))
    sock.settimeout(1.0)
    vistos = {}  # origen (ip, puerto) -> (alias, timestamp)
    print("Oyente de anuncios activo...")
    
    while True:
        try:
            datos, origen = sock.recvfrom(2048)
            alias = json.loads(datos.decode("utf-8"))["alias"]
            if origen not in vistos:
                print(f"[NUEVO DISPOSITIVO] {alias} desde {origen}")
            vistos[origen] = (alias, time.monotonic())
        except socket.timeout:
            pass
        except (ValueError, KeyError, TypeError):
            pass
            
        # Limpieza de dispositivos que dejaron de anunciarse (Timeout)
        ahora = time.monotonic()
        for origen, (alias, ultimo_visto) in list(vistos.items()):
            if ahora - ultimo_visto > EXPIRACION:
                print(f"[DESCONECTADO] {alias} ({origen}) expiró")
                del vistos[origen]
```

---

## 5. Lista de Errores Frecuentes en Programación de Sockets

Al implementar sockets en la prueba, **evitar estrictamente los siguientes errores**:

1. **Suponer que un `recv` equivale a un mensaje completo**: En TCP, `recv()` entrega cualquier cantidad de bytes disponibles (entre 1 y el buffer pedido). Siempre implementar delimitadores (`\n` o largo explícito).
2. **Ignorar el retorno `b""` de `recv()`**: Cuando el socket remoto cierra la conexión, `recv()` retorna `b""`. No verificar este caso genera bucles infinitos de CPU al 100%.
3. **Utilizar `send()` en lugar de `sendall()`**: `send()` puede transmitir solo una parte de los bytes entregados y requiere verificar el retorno. `sendall()` garantiza el envío de todo el bloque.
4. **Olvidar configurar `SO_REUSEADDR`**: Si el servidor se reinicia rápidamente, el puerto quedará bloqueado en estado `TIME_WAIT`, lanzando `OSError: [Errno 98] Address already in use`.
5. **No sanitizar nombres de archivo recibidos**: Aceptar nombres como `"../../archivo"` permite escribir en directorios del sistema. Usar siempre `os.path.basename()`.
6. **Cargar archivos completos en memoria**: Hacer `recv(size)` donde `size` es de cientos de MB satura la memoria RAM. Procesar siempre en bloques de tamaño fijo (ej. 64 KB) con un bucle.
7. **Bloquear el hilo principal en `accept()`**: Atender solicitudes extensas dentro del bucle de `accept()` impide recibir a nuevos clientes. Usar `threading.Thread(..., daemon=True)`.

---

# Checklist Final de Auto-evaluación del Temario

Revisa cada punto antes de la prueba para asegurar el dominio completo de la materia:

### Parte 1: DNS, Transporte y Red
- [x] Jerarquía de servidores DNS (Raíz, TLD, Autoritativos) y modelo de árbol.
- [x] Delegación de zona, registros `NS` y solución de la circularidad mediante registros `Glue`.
- [x] Diferencia entre consulta iterativa y recursiva, y flujo completo desde el host cliente.
- [x] Rol del resolver local, mecanismo de caché y efecto del parámetro `TTL`.
- [x] Interpretación de comandos `dig` y `dig +trace` (flags `aa`, `qr`, status `NOERROR`, `NXDOMAIN`).
- [x] Propósito y sintaxis de registros `SOA`, `NS`, `A`, `AAAA`, `CNAME`, `MX` y `TXT`.
- [x] Registros obligatorios en la zona propia (`SOA`, `NS`) y configuración en la zona padre.
- [x] Multiplexación y demultiplexación: Tupla UDP (2 elementos) vs. Cuádrupla TCP (4 elementos).
- [x] Justificación técnica de la elección entre UDP y TCP según los requerimientos de la aplicación.
- [x] Análisis comparativo entre Stop-and-Wait y Pipelining (Go-Back-N vs. Selective Repeat).
- [x] Cálculo de throughput máximo: $\text{Throughput} = \frac{\text{Window}}{RTT}$.
- [x] Control de congestión TCP Reno: Partida Lenta (Slow Start) y Evitación de Congestión.
- [x] Reacción ante 3 ACKs duplicados (Fast Retransmit) vs. Expiración de Timer (Timeout).
- [x] Evolución gráfica de $cwnd$, actualización del umbral $ssthresh$ y fases.
- [x] Notación CIDR, conversión a máscara decimal y cálculo de octetos.
- [x] Cálculo de Dirección de Red, Broadcast, Rango Asignable y número de hosts ($2^h - 2$).
- [x] Pertenencia de una IP a una subred y validación de asignabilidad.
- [x] Subnetting VLSM: algoritmo de ordenamiento de mayor a menor y atajo de tamaño de bloque ($256 - \text{máscara}$).
- [x] Diferencia entre Reenvío (Forwarding, Plano de Datos) y Enrutamiento (Routing, Plano de Control).
- [x] Regla del Prefijo Más Largo (Longest Prefix Match - LPM) en tablas de reenvío.

### Parte 2: Diseño de Protocolos de Aplicación
- [x] Definición formal de roles de host y responsabilidades en la arquitectura.
- [x] Selección y justificación del protocolo de transporte (TCP confiable vs. UDP rápido/difusión).
- [x] Caracterización de patrones de interacción (Solicitud-Respuesta vs. Mensajes no solicitados).
- [x] Especificación exhaustiva de tipos de mensaje, direcciones, campos y sintaxis JSON.
- [x] Estrategias de enmarcado en TCP (Líneas `\n`, largo explícito `struct` y transporte binario).
- [x] Modelado de la máquina de estados finitos (FSM) de cliente y servidor con transiciones.
- [x] Tratamiento de condiciones de error, timeouts, cancelación y mensajes `ERROR`.
- [x] Construcción de trazas completas de intercambio de mensajes para casos de éxito y error.

### Parte 3: Programación de Sockets en Python
- [x] Dominio de llamadas a la API `socket`: `bind`, `listen`, `accept`, `recv`, `sendall`, `recvfrom`, `sendto`.
- [x] Comprensión de TCP como flujo continuo de bytes sin fronteras.
- [x] Implementación de lectura de mensajes delimitados por `\n` usando buffer en `Conexion`.
- [x] Implementación de lectura exacta de $N$ bytes con `struct.pack("!I", len)` y `struct.unpack`.
- [x] Detección correcta de desconexión del cliente cuando `recv()` retorna `b""`.
- [x] Transferencia eficiente de archivos grandes por bloques de 64 KB sin sobrecargar memoria.
- [x] Concurrencia en servidores TCP utilizando `threading.Thread(target=..., daemon=True)`.
- [x] Sanitización de rutas de archivo con `os.path.basename()` y validación de JSON (`try/except`).
- [x] Manejo de difusión UDP (Broadcast) con `SO_BROADCAST` y timeouts de inactividad con `time.monotonic()`.
- [x] Prevención de errores comunes de sockets (`SO_REUSEADDR`, `sendall`, manejo de `socket.timeout`).
