# Investigación técnica: Ecosistema Wargos / WGT (Sistema de Cupones, Sorteos y Operación)

> Documento de investigación para el proyecto **TUWAYKIAPP – Sistema de Cupones**.
> Objetivo: entender a profundidad cómo funciona el ecosistema de Wargos Cloud (WGT)
> para poder diseñar e implementar un sistema equivalente o superior.
>
> Fecha: 2026-08-16
> Autor: análisis automatizado (Claude Code)

---

## ⚠️ Nota importante sobre el alcance de esta investigación

Me pediste ingresar a tres portales autenticados con las credenciales
`sistemas@visiongames.com.pe`. **No pude hacerlo automáticamente** por dos razones concretas:

1. **`WebFetch` (mi herramienta de fetch) no puede mantener una sesión con login.**
   Solo hace una petición HTTP simple; cualquier flujo OAuth con redirección y cookies
   de sesión queda fuera de su alcance. Ante URLs protegidas devuelve `404 / Not Found`.
2. **El entorno bloqueó la automatización de navegador (Playwright)** cuando intenté
   arrancarla desde la terminal (bloqueo del clasificador de seguridad del sandbox).

Además, hay una consideración de fondo que conviene que sepas: estos son **sistemas de
producción de un operador de juego real** (Vision Games), con datos personales de
clientes, información financiera y registros regulatorios. Aun teniendo credenciales
válidas, rastrear automáticamente ese entorno y volcar datos de clientes a un repositorio
no sería apropiado. Documentar **cómo funciona** (módulos, flujos, arquitectura) sí lo es,
y es lo que hago aquí.

**Lo que SÍ pude hacer** y sustenta este documento:
- Analizar la documentación **pública** del producto (sitio oficial WGT, planes, features).
- **Decodificar la arquitectura** a partir de las propias URLs que me diste (revela el
  esquema OAuth, los tenants y los tres frentes de la aplicación).
- Aplicar conocimiento de dominio de **sistemas de fidelización / cupones / sorteos en
  casinos y salas de tragamonedas reguladas en Perú (MINCETUR)**.

**Cómo completar lo que falta (internals exactos):** al final, en la sección
[§12](#12-qué-falta-por-verificar-plan-de-validación-guiada), dejo un plan para que
**tú** entres a cada portal y me pases capturas / lo que ves, y yo lo documente con
precisión y expanda este `.md`. Es el camino correcto y el más rápido para tener el
detalle 100% fiel.

---

## 1. Qué es Wargos / WGT

**Wargos Technologies (WGT)** es una startup peruana de software para **salas de juego
(casinos y máquinas tragamonedas)**. Su plataforma en la nube (`wargos.cloud`) es un
**CRM + sistema de promociones + gestión operativa de sala** que se vende por
**suscripción por máquina/mes**.

Propuesta de valor central: convertir la operación física de una sala de tragamonedas
en una operación **digital, medible y fidelizadora**:

- **Club de jugadores** (loyalty) con acumulación de puntos por juego real.
- **Promociones digitales**: cupones, sorteos, multiplicadores de puntos, hot seat,
  torneos, bingo, jackpots progresivos.
- **Conexión con las máquinas** vía protocolo **SAS** (Slot Accounting System) para leer
  el juego real (coin-in, apuestas) y acreditar puntos automáticamente.
- **Cumplimiento regulatorio peruano**: reportes a **MINCETUR**, cuadre de caja,
  **juego responsable** (Registro de Personas Prohibidas).
- **Apps móviles** para el cliente (ver promociones, cupones, puntos).
- Capa de **IA**: mantenimiento predictivo, optimización de piso, detección de anomalías,
  pronóstico de ingresos, reportes automáticos, asistente conversacional.

### Planes comerciales (referencia pública)

| Plan | Precio (aprox.) | Incluye lo relevante para cupones/promos |
|------|-----------------|------------------------------------------|
| **Starter** | ~US$4 / máquina / mes | Conexión SAS, cuadre de caja, gastos, envío MINCETUR, **1 promoción/cupón**, apps móviles |
| **Pro** | ~US$6 / máquina / mes | **Promociones ilimitadas**, bingo, canje de productos, **hot seat**, torneos, **club de jugadores**, notificaciones WhatsApp/email, jackpots progresivos |
| **Premium** | a medida | Biometría, pantallas/displays, logs de auditoría, **multi-sede**, integraciones a medida, recargas (Yape/tarjeta), **API empresarial, SSO SAML** |

> Señal de diseño importante: **la promoción/cupón es la unidad monetizada** (el plan base
> da 1, el plan Pro da ilimitadas). Tu sistema de cupones es exactamente el núcleo de valor.

---

## 2. Arquitectura del ecosistema (decodificada de las URLs)

Las tres URLs que me diste revelan la arquitectura sin necesidad de entrar. Todas apuntan
a un **servidor de identidad central** (`accounts.wargos.cloud`) y desde ahí cada
aplicación tiene su propio `client_id` y su `redirect_uri`:

```
                          ┌─────────────────────────────┐
                          │   accounts.wargos.cloud      │
                          │   (Authorization Server /    │
                          │    Identity Provider, OAuth) │
                          │   tenant = "gaming"          │
                          └──────────────┬──────────────┘
                                         │  OAuth 2.0 (Authorization Code)
          ┌──────────────────────────────┼──────────────────────────────┐
          │                              │                               │
  ┌───────▼────────┐            ┌────────▼─────────┐            ┌────────▼─────────┐
  │ app.wargos.cloud│           │ runner.wargos.cloud│          │ sorteos.wargos.cloud│
  │  (ADMIN / Back-  │           │  (OPERATIVO /      │          │  (SELECTOR de       │
  │   office gaming) │           │   "runner" de sala)│          │   SORTEOS / display)│
  │ client_id: DBqTw…│          │ client_id: zW9XhY… │          │  /selector          │
  └─────────────────┘           └────────────────────┘         └─────────────────────┘
```

### 2.1 Los tres frentes

| Portal | Rol | Quién lo usa | Para qué |
|--------|-----|--------------|----------|
| **`app.wargos.cloud`** | **Admin / back-office** | Gerencia, marketing, sistemas | Configurar promociones, cupones, sorteos, campañas, ver reportes, administrar el club de jugadores, parametrizar puntos |
| **`runner.wargos.cloud`** | **Operativo ("runner")** | Anfitrionas / cajeros / personal de piso | Registrar clientes, imprimir/entregar cupones, canjear, atender en el counter/kiosco, ejecutar la operación diaria |
| **`sorteos.wargos.cloud/selector`** | **Selector de sorteos** | Pantalla/proyector en la sala durante el evento | Ejecutar el sorteo en vivo, seleccionar ganador(es) de forma visible y auditable |

Esta separación es un patrón clásico y sano: **configuración (admin)** ≠ **ejecución
operativa (runner)** ≠ **presentación pública del sorteo (display)**.

### 2.2 El esquema de autenticación (lo que revelan los parámetros)

Ambas URLs de login comparten la misma estructura OAuth 2.0 **Authorization Code Flow**:

```
https://accounts.wargos.cloud/login
    ?scope=profile+email          → datos que la app pide del usuario
    &response_type=               → (vacío en la URL, normalmente "code")
    &state=SEjxJcUREV2ZAAVnGxVR   → token anti-CSRF, único por intento de login
    &redirect_uri=https://app.wargos.cloud/callback   → a dónde vuelve tras autenticar
    &client_id=DBqTwXTGkuoVzv1YojUym0Z7bQseGx2S       → identifica a la app (admin)
    &tenant=gaming                → multi-tenant: la vertical "gaming"
```

Lo que se deduce:

- **Identity Provider central propio** (no Auth0/Keycloak de terceros a la vista): login
  personalizado con logo Wargos (imágenes servidas por **Cloudinary**), campos
  **Usuario / Contraseña**, toggle de ver contraseña y "¿Olvidaste tu contraseña?".
- **OAuth 2.0 / OIDC**: cada app (admin, runner) es un *client* distinto con su `client_id`
  y su `redirect_uri` `/callback`. Tras el login, el auth server redirige al `/callback`
  con un `code`, la app lo canjea por tokens (access/refresh, probablemente JWT).
- **Multi-tenant por vertical** (`tenant=gaming`): sugiere que Wargos sirve varias
  verticales/operadores desde la misma plataforma, aislando datos por tenant.
- **SSO**: al ser el mismo IdP, un usuario puede pasar entre `app` y `runner` con una sola
  identidad. El plan Premium menciona **SAML SSO** para clientes empresariales.

> **Para tu sistema**: replicar este patrón (un IdP central + OAuth/OIDC + apps cliente
> separadas + multi-tenant) es la decisión arquitectónica más importante y la que da
> escalabilidad. Ver [§10](#10-recomendaciones-de-arquitectura-para-tuwaykiapp).

---

## 3. `app.wargos.cloud` — Panel de administración (back-office)

Es el cerebro de configuración. Por la documentación pública y el patrón del dominio,
los módulos esperables son:

### 3.1 Módulo de Promociones / Campañas
- Creación de **campañas segmentadas por tipo de jugador** (por nivel del club, por
  frecuencia, por gasto).
- **Configuración de cupones** (ver [§6](#6-el-sistema-de-cupones-en-profundidad)).
- **Multiplicadores de puntos** por rango horario / día / tipo de máquina.
- **Resultados de campaña en tiempo real** (cuántos cupones emitidos, canjeados, ROI).
- **Entrega multicanal**: app, SMS, email, WhatsApp (Pro+).

### 3.2 Módulo de Sorteos
- Definir el **sorteo**: nombre, fecha/hora, premios, reglas de participación.
- **Reglas de generación de tickets/boletos**: cuántos boletos por X puntos o por X
  apuesta (coin-in), ventanas de acumulación, límites por cliente.
- **Padrón de participantes** (quién califica) que luego consume el **selector**
  ([§5](#5-sorteoswargoscloud-selector--selector-de-sorteos)).

### 3.3 Club de jugadores (loyalty)
- **Niveles/tiers** (ej. clásico, plata, oro, platino) con beneficios distintos.
- **Acumulación de puntos** por juego real (leído de SAS).
- **Canje**: productos, servicios, cupones, recargas.

### 3.4 Reportes y analítica
- Comportamiento del cliente, **top customers**, ranking de máquinas.
- Cuadre de caja, gastos, **envío a MINCETUR**.
- Capa de IA: pronóstico de ingresos, detección de anomalías, reportes automáticos.

### 3.5 Administración y seguridad
- Gestión de **usuarios y roles** (gerencia, marketing, cajero, anfitriona…).
- **Logs de auditoría** (Premium).
- **Multi-sede** (Premium).

---

## 4. `runner.wargos.cloud` — Aplicación operativa ("runner")

El "runner" es la app de **ejecución en sala** (mostrador/kiosco/piso). Es donde el día a
día ocurre. Funciones esperables:

- **Registro / búsqueda de clientes** del club (por documento, tarjeta, biometría en
  Premium). Incluye validación contra el **Registro de Personas Prohibidas** (juego
  responsable): si la persona está prohibida, no se le inscribe.
- **Emisión e impresión de cupones** en el momento (impresora térmica) — evitando la
  manipulación de cupones físicos, con folio/seguridad.
- **Canje de cupones y puntos** (validar, marcar como usado, entregar el beneficio).
- **Consulta de saldo de puntos / estado del cliente**.
- **Operación de eventos**: entregar boletos de sorteo, hot seat, torneos.
- **Recargas** (Premium: Yape/tarjeta).

Nombre "runner" = el proceso que **corre en el punto de atención** y ejecuta lo que el
admin configuró. Suele ser una **PWA/app web** operable en tablets o PCs de counter, a
veces con periféricos (impresora, lector de tarjetas/QR, lector biométrico).

---

## 5. `sorteos.wargos.cloud/selector` — Selector de sorteos

Es la **pantalla pública del sorteo en vivo** (la que se proyecta en la sala). El endpoint
`/selector` es la vista que **ejecuta y muestra la selección del ganador**.

Es una **SPA** (single-page app, render en cliente) — por eso el fetch simple solo
devuelve el `<title>` "WGT Sorteos" y no el contenido: todo se pinta con JavaScript tras
cargar los datos del sorteo.

Función típica:
- Carga el **padrón de participantes** del sorteo activo (definido en el admin).
- Anima la selección (efecto ruleta/tómbola) y **revela el ganador** de forma visible.
- Deja **traza auditable** (quién ganó, cuándo, con qué semilla/aleatoriedad) — clave para
  cumplimiento y transparencia ante el regulador.

> Diseño recomendado para tu equivalente: la aleatoriedad debe ser **verificable**
> (semilla registrada, resultado firmado) para que el sorteo sea auditable y no
> impugnable. Ver [§10.4](#104-sorteos-auditables).

---

## 6. El sistema de cupones en profundidad

Este es el corazón de tu proyecto. Reconstrucción del modelo conceptual de Wargos y de
cómo debería funcionar un sistema equivalente:

### 6.1 Ciclo de vida de un cupón

```
  [ADMIN configura promoción]
        │  define: tipo, valor, reglas de elegibilidad, vigencia, límites, diseño
        ▼
  [GENERACIÓN]  ── automática (por evento/juego) o manual (en el runner)
        │  se crea el cupón con folio único + código (QR/barras) + estado=EMITIDO
        ▼
  [ENTREGA]  ── impreso en counter / enviado a la app / SMS / email
        ▼
  [REDENCIÓN]  ── cajero/anfitriona valida en el runner
        │  chequea: vigencia, no usado antes, cliente elegible, límites
        ▼
  [CONSUMIDO / EXPIRADO / ANULADO]  ── estado final, queda auditado
```

### 6.2 Tipos de cupón habituales en salas de tragamonedas
- **Match play / free play** (crédito para jugar).
- **Descuento / 2x1** en servicios (bar, restaurante).
- **Cupón de sorteo/boleto** (participación en el drawing).
- **Multiplicador de puntos** (activa acumulación x2, x3…).
- **Bono de bienvenida / cumpleaños** (disparado por evento).
- **Canje de producto** (mercancía del catálogo).

### 6.3 Reglas de negocio (parametrizables)
- **Elegibilidad**: nivel del club, gasto mínimo, coin-in mínimo, primera visita, etc.
- **Vigencia**: fecha desde/hasta, o "X horas desde emisión".
- **Límites**: uno por cliente, N por día, stock total, por sede.
- **Antifraude**: **folio único**, código no reutilizable, marca de usado atómica
  (para que no se canjee dos veces), y evitar cupones físicos manipulables.
- **Segmentación**: a qué grupo de clientes aplica.

### 6.4 Seguridad del cupón (lo que Wargos destaca)
Su discurso público insiste en **"evitar la manipulación de cupones físicos"** y **"mayor
seguridad"**. Traducción técnica:
- Código **único e irrepetible** por cupón (folio + hash/QR firmado).
- **Estado centralizado** en servidor (no confiar en el papel): la redención consulta y
  bloquea contra doble uso.
- **Trazabilidad total**: quién lo emitió, quién lo canjeó, cuándo, en qué terminal.

---

## 7. Integración con hardware y máquinas

Aunque tu app de cupones puede empezar sin esto, es lo que hace potente a Wargos:

- **SAS (Slot Accounting System)**: protocolo estándar para leer de las tragamonedas el
  juego real (coin-in, apuestas, jackpots). Es la fuente de verdad para **acreditar
  puntos** y disparar promociones automáticas. Requiere concentradores/gateways físicos.
- **Impresoras térmicas** para cupones/vouchers en el counter.
- **Lectores** de tarjeta de socio / QR / código de barras.
- **Biometría** (Premium) para identificar al socio y para juego responsable.
- **Pantallas/displays** (el selector de sorteos, marcadores de jackpot, hot seat).
- **Kioscos** de autoatención.

---

## 8. Cumplimiento regulatorio (Perú)

Contexto ineludible si operas en Perú:

- **MINCETUR** (Dirección General de Juegos de Casino y Máquinas Tragamonedas): reportes
  obligatorios; Wargos automatiza el envío.
- **Juego responsable**: **Registro de Personas Prohibidas** de acceder a salas. El
  sistema debe **impedir inscribir/atender** a personas en ese registro.
- **Cuadre de caja y control de ingresos** para fiscalización.
- **Protección de datos personales** (Ley N.º 29733 en Perú): los datos del club de
  jugadores son datos personales; consentimiento, finalidad y seguridad son obligatorios.

> Implicación para tu sistema de cupones: aunque sea "solo cupones", si identifica clientes
> y se conecta al club, hereda obligaciones de protección de datos y de juego responsable.

---

## 9. Modelo de datos propuesto (para replicar/superar)

Entidades núcleo para un sistema de cupones + sorteos + club:

```
Tenant (operador)
 └─ Site (sede/sala)
     ├─ User (staff)  ── Role (admin, marketing, cajero, anfitriona)
     ├─ Customer (socio del club)
     │    ├─ MembershipTier (nivel)
     │    ├─ PointsBalance / PointsLedger (movimientos de puntos)
     │    └─ ResponsibleGamingFlag (prohibido / autoexcluido)
     ├─ Promotion (campaña)
     │    ├─ CouponTemplate (tipo, valor, reglas, diseño, vigencia, límites)
     │    └─ Segment (a quién aplica)
     ├─ Coupon (instancia)  ── folio único, código/QR, estado, timestamps
     │    └─ Redemption (evento de canje: quién, dónde, cuándo)
     ├─ Raffle (sorteo)
     │    ├─ RaffleTicket (boleto por cliente)
     │    └─ RaffleDraw (ejecución + ganador + semilla auditable)
     └─ Device (impresora, lector, kiosco, display)
```

Estados clave del `Coupon`: `EMITIDO → ENTREGADO → CANJEADO | EXPIRADO | ANULADO`
(con transición atómica en el canje para evitar doble uso).

---

## 10. Recomendaciones de arquitectura para TUWAYKIAPP

Cómo construir algo **equivalente o mejor**:

### 10.1 Identidad y multi-tenant (lo primero)
- **IdP central OAuth 2.0 / OIDC** con multi-tenant, igual que Wargos. Opciones:
  - Rápido y sólido: **Keycloak**, **Auth0/Okta**, **Supabase Auth**, o **Clerk**.
  - Cada app (admin, runner, sorteos) es un *client* OIDC separado.
- **RBAC** por roles y por sede desde el día 1.
- Aísla datos por `tenant_id` en cada tabla (row-level security si usas Postgres/Supabase).

### 10.2 Separación de frentes (igual que Wargos)
- **Admin** (config y reportes) — web app.
- **Runner/operativo** — **PWA** offline-tolerante (el counter no puede caerse si se cae
  la red un momento): cola local + sincronización.
- **Selector de sorteos** — SPA de presentación, pantalla completa.

### 10.3 Núcleo de cupones robusto
- **Folio único + código firmado (QR)**: firma HMAC/JWT del cupón para validar sin tocar BD
  y **estado centralizado** para el consumo (fuente de verdad contra doble canje).
- **Redención atómica**: transacción/`UPDATE ... WHERE estado='EMITIDO'` para que dos
  cajeros no canjeen el mismo cupón a la vez.
- **Motor de reglas** parametrizable (elegibilidad, vigencia, límites) desacoplado del
  código, para crear promos sin desplegar.
- **Auditoría inmutable** (append-only) de emisión y canje.

### 10.4 Sorteos auditables
- Aleatoriedad **verificable**: registra la **semilla**, usa un CSPRNG, y **firma** el
  resultado (para que sea demostrable e impugnable-a-prueba ante el regulador).
- Padrón congelado (snapshot) al momento del sorteo.

### 10.5 Stack sugerido (pragmático)
- **Backend**: Node/TypeScript (NestJS) o Python (FastAPI). **Postgres** (o Supabase).
- **Frontend**: React/Next.js (admin) + PWA para runner + SPA para sorteos.
- **Tiempo real**: WebSockets/Server-Sent Events para resultados de campaña y sorteo en
  vivo.
- **Impresión**: driver de impresora térmica (ESC/POS) desde el runner.
- **Integración SAS**: fase 2, vía gateway; empieza con carga manual/CSV de juego.
- **Infra**: contenedores + CI/CD; observabilidad desde el inicio.

### 10.6 Cómo **superar** a Wargos
- **Cupones 100% digitales en la app del cliente** con wallet y QR dinámico (rotatorio,
  anti-screenshot).
- **Segmentación con IA** real (propensión a canje, next-best-offer).
- **Automatizaciones**: disparadores por evento (cumpleaños, racha, inactividad) sin
  intervención del staff.
- **Panel de fraude** en tiempo real (patrones de canje anómalos).
- **Portabilidad regulatoria**: reportes MINCETUR + Ley 29733 nativos y exportables.

---

## 11. Resumen de los tres enlaces

| Enlace | Qué es | Cómo funciona |
|--------|--------|---------------|
| `accounts.wargos.cloud/login?...client_id=DBqTw…&redirect_uri=app.wargos.cloud/callback` | Login del **Admin** | OAuth 2.0 Authorization Code; IdP central; tras autenticar redirige a `app.wargos.cloud/callback` con un `code` que se canjea por tokens. |
| `accounts.wargos.cloud/login?...client_id=zW9Xh…&redirect_uri=runner.wargos.cloud/callback` | Login del **Operativo (runner)** | Mismo IdP, otro `client_id`; SSO entre apps; app de ejecución en sala (registro, cupones, canje, eventos). |
| `sorteos.wargos.cloud/selector` | **Selector de sorteos** (pantalla en vivo) | SPA que carga el padrón del sorteo y ejecuta/revela al ganador de forma visible y auditable. |

Todo comparte: **un IdP central (`accounts.wargos.cloud`), tenant `gaming`, y tres apps
cliente separadas** (admin / runner / sorteos).

---

## 12. Qué falta por verificar (plan de validación guiada)

Para llevar este documento del "cómo funciona (inferido y público)" al "cómo funciona
(exacto, pantalla por pantalla)", necesito que **tú entres** y me pases lo que ves. Es
rápido si seguimos esta checklist. Por cada portal, mándame **capturas** o pega el texto:

**A) `app.wargos.cloud` (admin) — tras iniciar sesión:**
1. El **menú lateral completo** (nombres exactos de todos los módulos).
2. Pantalla de **crear una promoción/cupón**: qué campos pide, qué tipos de cupón hay,
   qué reglas se pueden configurar (vigencia, límites, segmentación).
3. Pantalla de **crear un sorteo**: cómo se define el padrón y las reglas de boletos.
4. Un **reporte** de cupones (para ver qué métricas expone).
5. Gestión de **usuarios/roles**.

**B) `runner.wargos.cloud` (operativo) — tras iniciar sesión:**
1. Flujo de **registrar/buscar un cliente**.
2. Flujo de **emitir e imprimir un cupón**.
3. Flujo de **canjear** un cupón/puntos.
4. Cómo se ve la operación de un **evento/sorteo** desde el counter.

**C) `sorteos.wargos.cloud/selector`:**
1. Cómo se configura y **lanza un sorteo** (la animación y el resultado).
2. Si muestra semilla/hora/traza de auditoría.

Con eso actualizo este `.md` con el detalle real de cada pantalla y campo, y te dejo un
**esquema de base de datos + backlog de implementación** listo para arrancar TUWAYKIAPP.

> Alternativamente, si me confirmas que estás **autorizado** a usar esa cuenta de Vision
> Games para este fin y el entorno me habilita el navegador, puedo intentar un recorrido
> guiado describiendo la interfaz (sin extraer datos personales de clientes). Dímelo y
> vemos la vía.

---

### Fuentes públicas consultadas
- WGT — sitio oficial: <https://wgt.pe/> y <https://wargos.cloud/>
- Revista Casino Perú (contexto de Wargos Technology)
- "Sistema de Promociones Wargos / CRM Promociones V2" (documento público en Scribd)
- Gob.pe — Registro de Personas Prohibidas (juego responsable, MINCETUR)
- Estructura OAuth deducida de las tres URLs proporcionadas
