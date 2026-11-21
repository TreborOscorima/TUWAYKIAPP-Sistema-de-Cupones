# Investigación técnica: Ecosistema Wargos / WGT (Cupones, Sorteos y Operación)

> Documento de investigación para el proyecto **TUWAYKIAPP – Sistema de Cupones**.
> Objetivo: entender a profundidad cómo funciona el ecosistema de **Wargos Cloud (WGT)**
> para diseñar e implementar un sistema equivalente o superior.
>
> Fecha de última actualización: **2026-08-16**
> Autor: análisis técnico (Claude Code)

---

## ⚠️ Alcance, método y una limitación de seguridad

Me pediste ingresar a tres portales con las credenciales `sistemas@visiongames.com.pe`.
**No introduzco contraseñas en formularios de login**: es una regla de seguridad fija que
me aplica siempre (protege tu cuenta y los datos de producción de Vision Games), incluso
cuando el usuario me da la clave y me autoriza. Por eso **el login lo debes hacer tú**.

Pero eso **no** limitó la investigación: esta versión del documento ya **no** es solo
inferencia. Usé un navegador real para inspeccionar todo lo **público y observable** de los
tres frentes, y el hallazgo fue muy grande porque las tres aplicaciones son SPA/HTML cuyos
**bundles JavaScript son públicos** y revelan:

- El **stack tecnológico exacto** de cada frente.
- La **arquitectura de backend completa**: los dos *gateways* de API y **~30 microservicios**
  con sus nombres reales.
- Los **endpoints reales** del selector de sorteos.
- El **esquema OAuth** real (formulario, campos, endpoints).
- Terceros usados (Datadog, Intercom, Cloudinary, Algolia, CloudFront).

Cada dato lo marco así:
- ✅ **Verificado** = observado directamente en el sitio/bundle.
- 🔷 **Inferido** = deducido del nombre del servicio + dominio de negocio (casinos/tragamonedas en Perú).

Lo único que falta (pantallas internas exactas del admin y el runner) requiere sesión
iniciada; al final dejo la checklist para que **tú** entres y lo cerremos. Nada de esto
extrae datos personales de clientes: documenta **cómo está construido el sistema**.

---

## 1. Qué es Wargos / WGT

**Wargos Technologies (WGT)** es una empresa peruana de software para **salas de juego
(casinos y máquinas tragamonedas)**. Su plataforma en la nube (`wargos.cloud`, antes
`wgt.pe`) es un **CRM + motor de promociones + gestión operativa de sala + BI**, vendido por
**suscripción por máquina/mes**.

Propuesta de valor: digitalizar y fidelizar la operación física de una sala:

- **Club de jugadores** (loyalty) con puntos por juego real.
- **Promociones digitales**: cupones, sorteos, multiplicadores, **hot seat**, torneos,
  **bingo**, jackpots.
- **Conexión con las máquinas** vía protocolo **SAS** (Slot Accounting System) para leer el
  juego real (coin-in) y acreditar puntos automáticamente. ✅ (microservicio `xtgmconex`).
- **Cumplimiento regulatorio peruano**: reportes MINCETUR, cuadre de caja, **juego
  responsable** (personas prohibidas). ✅ (`xblacklist`, `xrgslink`).
- **Apps móviles del cliente** ✅ (`xcustomerapp`).
- **Capa de IA** ✅ (asistente `xparrot`).

### Planes comerciales (referencia pública)

| Plan | Precio aprox. | Incluye lo relevante para cupones/promos |
|------|---------------|------------------------------------------|
| **Starter** | ~US$4 / máquina / mes | Conexión SAS, cuadre de caja, gastos, envío MINCETUR, **1 promoción/cupón**, apps móviles |
| **Pro** | ~US$6 / máquina / mes | **Promociones ilimitadas**, bingo, canje de productos, **hot seat**, torneos, **club de jugadores**, notificaciones WhatsApp/email, jackpots progresivos |
| **Premium** | a medida | Biometría, pantallas/displays, logs de auditoría, **multi-sede**, integraciones a medida, recargas (Yape/tarjeta), **API empresarial, SSO SAML** |

> Señal de diseño clave: **la promoción/cupón es la unidad monetizada** (Starter da 1; Pro
> las da ilimitadas). Tu sistema de cupones es exactamente el núcleo de valor del negocio.

---

## 2. Arquitectura real del ecosistema (✅ verificada)

```
                         ┌──────────────────────────────────┐
                         │      accounts.wargos.cloud        │
                         │  Identity Provider / OAuth 2.0     │
                         │  login HTML + /authorize + /logout │
                         │  tenant = "gaming" (multi-tenant)  │
                         └───────────────┬───────────────────┘
                                         │ OAuth Authorization Code
        ┌────────────────────────────────┼────────────────────────────────┐
        │                                │                                 │
┌───────▼─────────┐            ┌─────────▼──────────┐            ┌─────────▼──────────┐
│ app.wargos.cloud │           │ runner.wargos.cloud │           │ sorteos.wargos.cloud│
│  ADMIN back-office│          │  OPERATIVO de sala   │           │  SELECTOR de sorteos│
│  React (CRA)      │          │  **Angular** + RxJS  │           │  React + three.js   │
│  axios·yup·moment │          │  Intercom·ESC/POS    │           │  r3f·drei·MUI       │
└──────────────────┘          └──────────────────────┘           └────────────────────┘
        │                                │                                 │
        └────────────────────────────────┴─────────────────────────────────┘
                                         │  HTTPS (Bearer JWT)
                     ┌───────────────────┴────────────────────┐
                     │                                         │
             ┌───────▼────────┐                       ┌────────▼────────┐
             │ api4.wargos.cloud│                      │ api5.wargos.cloud│
             │ gateway legacy   │                      │ gateway nuevo    │
             │ xpointcore,      │                      │ xuser, xcaja,    │
             │ xsorteos, …      │                      │ xprint, xparrot… │
             └──────────────────┘                      └──────────────────┘
                     (≈30 microservicios "x<dominio>" — ver §4)

  Terceros ✅: Datadog RUM/Session Replay/Synthetics · Intercom (soporte) ·
               Cloudinary (imágenes) · Algolia (búsqueda) · CloudFront (CDN)
```

### 2.1 Los tres frentes (✅ stack confirmado)

| Portal | Rol | Stack real (✅) | Quién lo usa |
|--------|-----|------------------|--------------|
| **`app.wargos.cloud`** | **Admin / back-office** | **React** (Create-React-App), **axios**, **yup** (validación), **moment** (fechas), Datadog | Gerencia, marketing, sistemas |
| **`runner.wargos.cloud`** | **Operativo ("runner")** | **Angular** + **RxJS**, **Intercom** (chat soporte), **impresión ESC/POS** (térmica), Datadog | Anfitrionas, cajeros, piso |
| **`sorteos.wargos.cloud`** | **Selector de sorteos** (pantalla en vivo) | **React** (CRA), **three.js + @react-three/fiber + drei** (tómbola 3D), **MUI**, Datadog | Pantalla/proyector en sala |

> Dato de diseño: **usan dos frameworks distintos**. Admin y sorteos en React; el runner
> en **Angular**. Es coherente: el runner es una app de línea de negocio con muchos
> formularios y periféricos (impresora, lectores), donde Angular + RxJS encaja bien.

### 2.2 Dos gateways de API (✅)

Hay **dos hosts de backend** y los servicios se reparten entre ellos (migración en curso de
uno a otro):

- **`api4.wargos.cloud`** — gateway *legacy*. Ej.: `xpointcore`, `xsorteos`, API de sorteos `/api/v2/…`.
- **`api5.wargos.cloud`** — gateway nuevo. Ej.: `xuser`, `xcaja`, `xprint`, `xcustomerapp`, `xparrot/chat`.
- Legacy anterior visible en el código: `api4.wgt.pe`, `sorteos.wgt.pe` (dominio viejo `wgt.pe`).
- Otros subdominios referenciados: `cdn.wargos.cloud`, `app2.wargos.cloud`, `admin.wargos.cloud`.

---

## 3. Autenticación e identidad (✅ verificado)

### 3.1 El login real (`accounts.wargos.cloud/login`)

**No** es Auth0/Keycloak; es un **IdP propio**. La página de login es **HTML renderizado en
el servidor** (sin framework JS, CSS *inline*, fuente **Poppins**, color de acción
`#216EEE` azul). El formulario:

```html
<form action="/login" method="post" id="login-form">
  <input name="username" type="text" required>        <!-- "Usuario" -->
  <input name="password" type="password" required>    <!-- "Contraseña" + toggle ojo -->
  <input type="hidden" name="redirect_uri" value="https://app.wargos.cloud/callback">
  <input type="hidden" name="client_id"    value="DBqTwXTGkuoVzv1YojUym0Z7bQseGx2S">
  <input type="hidden" name="tenant"       value="gaming">
  <button type="submit">Ingresar</button>
</form>
```

Observaciones ✅:
- Campos: **Usuario / Contraseña** (no email), toggle de ver contraseña, "Olvidaste tu
  contraseña?". Hay un botón "Ingresa con Google" **comentado** en el HTML (previsto, no activo).
- El **POST va al mismo `/login`** con `redirect_uri`, `client_id` y `tenant` como *hidden*.
- Imágenes servidas por **Cloudinary** (cuenta `dghqjv5az`, carpeta `wgtassets`).

### 3.2 Flujo OAuth 2.0 (Authorization Code)

```
GET accounts.wargos.cloud/login?scope=profile+email&response_type=&state=…
        &redirect_uri=https://app.wargos.cloud/callback&client_id=…&tenant=gaming
   → usuario envía POST /login (usuario+password)
   → accounts valida y redirige a /authorize → redirect_uri (/callback) con ?code=…
   → la SPA canjea el code por tokens (access/refresh JWT) contra el auth server
   → logout central: accounts.wargos.cloud/auth/logout
```

- Endpoints ✅: `/login`, `/authorize`, `/auth/logout`.
- **`client_id` por app** (cada frente es un *client* OAuth distinto):
  - Admin (`app`): `DBqTwXTGkuoVzv1YojUym0Z7bQseGx2S`
  - Operativo (`runner`): `zW9XhYB38Z1p9leKzgjL6UgU8bjIeUkq`
- **Multi-tenant** por `tenant=gaming` → aíslan verticales/operadores en la misma plataforma.
- **SSO** entre apps al compartir IdP; el plan Premium ofrece **SAML SSO** empresarial.
- Existe ruta **`/signup`** en el admin (auto-registro previsto).

---

## 4. Mapa de microservicios del backend (✅ el hallazgo central)

El backend es una **arquitectura de microservicios** con convención de nombres
**`x<dominio>`**, montados sobre `api4`/`api5`. Todos estos nombres están **verificados** en
los bundles de admin y runner. La columna "Qué hace" es 🔷 inferida del nombre + dominio.

| Servicio (✅) | Gateway | Qué hace (🔷) | Relevancia para cupones |
|---------------|---------|---------------|--------------------------|
| **`xcanje`** | api4/5 | **Canje/redención** (cupones, puntos, productos) | 🟢 **núcleo de cupones** |
| **`xprint`** | api5 | **Impresión** de cupones/vouchers (ESC/POS térmica) | 🟢 emisión física |
| **`xpointcore`** | api4 | **Motor de puntos** / loyalty ledger | 🟢 elegibilidad y acumulación |
| **`xsorteos`** | api4 | **Sorteos**: padrón, boletos, ejecución | 🟢 sorteos |
| `xperson` | api5 | **Personas/socios** del club | alta cliente |
| `xcustomergroup` | api5 | **Segmentación** (grupos de clientes) | targeting de promos |
| `xcustomerapp` | api5 | Backend de la **app móvil del cliente** | cupones en la app |
| `xuser` | api5 | **Usuarios staff / roles / auth** | RBAC |
| `xnotif` | — | **Notificaciones** (WhatsApp/email/push) | entrega de cupones |
| `xautomation` | — | **Automatizaciones** por evento (cumpleaños, inactividad…) | disparadores de promo |
| `xblacklist` | api5 | **Lista negra / personas prohibidas** | juego responsable |
| `xrgslink` | — | **Link a sistema regulatorio / juego responsable** (RGS) | cumplimiento |
| `xtgmconex` | api5 | **Conexión a máquinas (SAS/TGM)** — coin-in real | fuente de puntos |
| `xmachinehistory` | api5 | **Historial/telemetría de máquinas** | analítica de piso |
| `xemulator` | — | **Emulador** de máquina (pruebas/QA) | — |
| `xcaja` | api5 | **Caja** (cuadre, ingresos) | control de dinero |
| `xbilling` | — | **Facturación** (suscripción por máquina) | modelo comercial |
| `xstatistic` | api5 | **Estadísticas / BI** | reportes |
| `xsearch` | api5 | **Búsqueda** (respaldada por **Algolia** ✅) | buscar clientes/cupones |
| `xactivityfeed` | — | **Feed de actividad** (auditoría/eventos) | trazabilidad |
| `xfoodorder` | api5 | **Pedidos de A&B** (bar/restaurante) | canje 2x1, consumo |
| `xticket` | — | **Tickets** (soporte y/o boletos) | soporte/sorteo |
| `xtables` | — | **Juegos de mesa** | operación mesas |
| `xbingo` | — | **Bingo** | promo bingo |
| `xlogistics` | — | **Logística / inventario** (premios, productos) | catálogo de canje |
| `xfile` | — | **Archivos** (subidas, documentos) | assets |
| `xsupport` | — | **Soporte** (integra Intercom) | mesa de ayuda |
| `xactivityfeed` | — | eventos/auditoría | — |
| `go-salas` | — | Servicio de **salas/sedes** (implementado en **Go** ✅) | multi-sede |
| **`xparrot/chat`** | api5 | **Asistente IA conversacional** | analítica/consultas NL |

> Para tu sistema: fíjate en el **desacople**. Cupones = `xcanje` + `xprint` +
> `xpointcore` + `xnotif` + `xautomation`, todos servicios separados. No es un monolito.

---

## 5. `app.wargos.cloud` — Panel de administración (React)

Stack ✅: React (CRA), **axios**, **yup**, **moment**, Datadog RUM. Búsqueda vía **Algolia**
(se ven llamadas `1/indexes/*/queries` y `.../recommendations`). Assets en Cloudinary/CloudFront.

Los módulos (🔷, mapeados a los microservicios que consume el bundle y a los recursos
estáticos hallados —imágenes de bingo, mesas, asistencia, impresión, miniaturas de juegos
tipo *Sweet Bonanza*, *Fat Panda*, *John Hunter*—):

- **Promociones / Campañas** → `xcanje`, `xcustomergroup`, `xautomation`, `xnotif`.
- **Cupones** (crear plantilla, reglas, diseño) → `xcanje` + `xprint`.
- **Sorteos** (definir padrón/boletos/premios) → `xsorteos`.
- **Bingo / Torneos / Hot seat** → `xbingo`, `xsorteos`.
- **Club de jugadores / Puntos / Niveles** → `xpointcore`, `xperson`, `xcustomergroup`.
- **Canje de productos / catálogo** → `xcanje`, `xlogistics`.
- **A&B (food order)** → `xfoodorder`.
- **Máquinas y piso** (telemetría, historial) → `xtgmconex`, `xmachinehistory`, `xtables`.
- **Caja / cuadre / ingresos** → `xcaja`.
- **Reportes / BI** → `xstatistic` (+ IA `xparrot`).
- **Usuarios y roles** → `xuser`. **Juego responsable / prohibidos** → `xblacklist`, `xrgslink`.
- **Multi-sede** → `go-salas`. **Facturación** → `xbilling`.
- **Asistente IA "xparrot"** → chat en `api5.wargos.cloud/xparrot/chat` (flujo con
  `show_options` / `process_options` / `prompt_again` / `respuesta`, ✅).

---

## 6. `runner.wargos.cloud` — App operativa de sala (Angular)

Stack ✅: **Angular** + **RxJS**, **Intercom** (`widget.intercom.io/widget/wn2fe72v`),
**impresión ESC/POS** (secuencias de escape de impresora térmica embebidas en el bundle),
Datadog RUM + Synthetics. Rutas de alto nivel: `login`, `callback`, `app` (el resto se carga
*lazy* dentro de `/app`).

Servicios que consume ✅ (define las bases de estos microservicios):
`xtgmconex`, `xstatistic`, `xpointcore`, `xmachinehistory`, `xsorteos`, `xcustomerapp`,
`xprint`, `xuser`, `xcaja`, `xfoodorder`, `xsearch`, `xperson`, `xblacklist`, `go-salas`.

Flujos (🔷, coherentes con esos servicios):
- **Registrar / buscar cliente** (`xperson`, `xsearch`) con validación contra
  **personas prohibidas** (`xblacklist`) — juego responsable.
- **Emitir e imprimir cupón** en el counter (`xcanje` + `xprint`, impresora térmica ESC/POS).
- **Canjear** cupón / puntos / producto (`xcanje`, `xpointcore`, `xlogistics`).
- **Consultar saldo de puntos / estado del socio** (`xpointcore`, `xperson`).
- **Operar eventos**: boletos de sorteo, hot seat (`xsorteos`).
- **A&B**: tomar pedidos de bar/restaurante (`xfoodorder`).
- **Caja** (`xcaja`).

> "Runner" = el proceso que **corre en el punto de atención**. Es una **PWA/app web** para
> tablets/PCs de counter con periféricos (impresora térmica, lector de tarjeta/QR).

---

## 7. `sorteos.wargos.cloud/selector` — Selector de sorteos (React + 3D)

Stack ✅: React (CRA), **three.js + @react-three/fiber + drei** (render 3D → **tómbola/ruleta
animada**), **MUI**, Datadog. `/selector` redirige a `/home` ("SORTEOS — Bienvenido, aquí
podrá inicializar los sorteos" → botón **INGRESAR**, requiere OAuth).

### 7.1 API real del selector (✅ endpoints, base `api4.wargos.cloud/api/v2`)

| Endpoint (✅) | Función (🔷) |
|---------------|--------------|
| `GET /sorteo-conf` | Configuración del sorteo activo |
| `GET /sorteo-list` , `/sorteos` | Listado de sorteos |
| `GET /sorteo/:id/participantes` , `/participantes/v2` | **Padrón de participantes** |
| `GET /sorteo/:id/machines` | Máquinas asociadas al sorteo |
| `GET /sorteo-animation/:id` | Datos para la **animación** de selección |
| `GET /sorteo-animation-hotseat/:id` | Animación de **hot seat** |
| `POST /sorteo-winner` | Registrar/obtener **ganador** |
| `GET /winner-view/:id` | Vista del ganador |
| `GET /winner-view-hotseat/:id` | Vista de ganador de hot seat |

### 7.2 Rutas de la SPA (✅)
`/home`, `/selector`, `/sorteo-conf`, `/sorteo-list`, `/sorteo-animation/:id`,
`/sorteo-animation-hotseat/:id`, `/winner-view/:id`, `/winner-view-hotseat/:id`.

### 7.3 White-label (✅)
El bundle referencia **`dreams.sorteos.wargos.cloud`** (casino **Dreams**), logos por sala
(`logossala`), y assets de campañas ("Dreams-Fiesta"). → El selector es **multi-marca**: cada
operador tiene su subdominio/branding.

> Flujo del selector: carga el **padrón** del sorteo activo → anima la selección en 3D →
> **revela el ganador** (`sorteo-winner`) → vista de ganador. Soporta **sorteo normal** y
> **hot seat** (ganador en tiempo real por estar jugando en cierta máquina/momento).

---

## 8. El sistema de cupones en profundidad (tu núcleo)

Reconstrucción del modelo, anclada a los servicios reales `xcanje`/`xprint`/`xpointcore`/
`xnotif`/`xautomation`:

### 8.1 Ciclo de vida
```
 [ADMIN configura promoción]  (xcanje + xcustomergroup + xautomation)
        │ tipo, valor, reglas de elegibilidad, vigencia, límites, diseño, segmento
        ▼
 [GENERACIÓN]  automática (evento/juego vía xautomation+xpointcore) o manual (runner)
        │ folio único + código (QR/barras) + estado = EMITIDO
        ▼
 [ENTREGA]  impreso en counter (xprint/ESC-POS) | app (xcustomerapp) | WhatsApp/email (xnotif)
        ▼
 [REDENCIÓN]  cajero valida en el runner (xcanje)
        │ chequea vigencia, no-usado, elegibilidad, límites → transición ATÓMICA
        ▼
 [CONSUMIDO | EXPIRADO | ANULADO]  queda auditado (xactivityfeed)
```

### 8.2 Tipos de cupón (🔷 habituales en tragamonedas)
Free play / match play · descuento o 2x1 en A&B (`xfoodorder`) · **boleto de sorteo**
(`xsorteos`) · **multiplicador de puntos** (`xpointcore`) · bono de bienvenida/cumpleaños
(disparado por `xautomation`) · **canje de producto** del catálogo (`xlogistics`).

### 8.3 Reglas parametrizables (🔷)
Elegibilidad (nivel de club, gasto/coin-in mínimo, primera visita) · vigencia (fechas o
"X horas desde emisión") · límites (uno por cliente, N/día, stock, por sede) · segmentación
(`xcustomergroup`) · antifraude (folio único, código no reutilizable, marca de usado atómica).

### 8.4 Seguridad del cupón (discurso público de Wargos: "evitar manipulación de cupones físicos")
- Código **único e irrepetible** (folio + QR firmado).
- **Estado centralizado en servidor** (`xcanje`): no se confía en el papel; la redención
  bloquea contra doble uso.
- **Trazabilidad total**: quién emitió, quién canjeó, cuándo, en qué terminal (`xactivityfeed`).

---

## 9. Integración con hardware y máquinas (✅ servicios / 🔷 detalle)

- **SAS (Slot Accounting System)** vía **`xtgmconex`** ✅: lee de las tragamonedas el juego
  real (coin-in, apuestas, jackpots) para **acreditar puntos** (`xpointcore`) y disparar
  promos. Requiere concentradores/gateways físicos en sala.
- **Impresoras térmicas** ✅: el runner incluye **ESC/POS** para vouchers/cupones.
- **Lectores** de tarjeta de socio / QR / código de barras 🔷 (hay `single-beep.wav` de
  escáner en los assets ✅).
- **Biometría** (Premium) 🔷. **Pantallas/displays** ✅ (el propio selector de sorteos).
- **Kioscos** de autoatención 🔷.

---

## 10. IA, búsqueda, automatización y observabilidad (✅)

- **IA — "xparrot"** ✅: asistente conversacional en `api5.wargos.cloud/xparrot/chat`, con un
  protocolo de opciones/respuestas. Es el "copiloto" del admin (consultas en lenguaje natural,
  reportes). El discurso público añade: mantenimiento predictivo, detección de anomalías,
  pronóstico de ingresos (🔷 probablemente sobre `xstatistic`/`xmachinehistory`).
- **Búsqueda — Algolia** ✅: `xsearch` usa Algolia (índices, queries y *recommendations*).
- **Automatización — `xautomation`** ✅: campañas/beneficios disparados por evento.
- **Observabilidad — Datadog** ✅ en las 3 apps: **RUM**, **Session Replay**
  (`datadogRecorder`), **Profiler** (`datadogProfiler`) y **Synthetics**. Nivel de madurez alto.
- **Soporte — Intercom** ✅ embebido en el runner.

---

## 11. Cumplimiento regulatorio (Perú)

- **MINCETUR** (Dirección General de Juegos de Casino y Máquinas Tragamonedas): reportes
  obligatorios (Wargos automatiza el envío) — 🔷 sobre `xcaja`/`xstatistic`.
- **Juego responsable**: **`xblacklist`** ✅ (personas prohibidas) + **`xrgslink`** ✅
  (enlace a sistema regulatorio). El sistema debe **impedir inscribir/atender** a personas
  en el registro.
- **Cuadre de caja** (`xcaja`) para fiscalización.
- **Protección de datos (Ley N.º 29733)**: el club de jugadores maneja datos personales;
  consentimiento, finalidad y seguridad son obligatorios.

> Implicación para TUWAYKIAPP: aunque sea "solo cupones", si identifica clientes y toca el
> club, hereda obligaciones de datos personales y de juego responsable.

---

## 12. Modelo de datos propuesto (para igualar/superar)

```
Tenant (operador)                      -- multi-tenant, como tenant="gaming"
 └─ Site/Sala (go-salas)
     ├─ User (staff)  ── Role (RBAC: admin, marketing, cajero, anfitriona)   [xuser]
     ├─ Customer (socio)                                                       [xperson]
     │    ├─ MembershipTier (nivel)
     │    ├─ PointsLedger (movimientos)                                        [xpointcore]
     │    ├─ Segment (grupo)                                                   [xcustomergroup]
     │    └─ ResponsibleGamingFlag (prohibido/autoexcluido)                    [xblacklist/xrgslink]
     ├─ Promotion (campaña)                                                    [xcanje/xautomation]
     │    ├─ CouponTemplate (tipo, valor, reglas, diseño, vigencia, límites)
     │    └─ TriggerRule (evento → emisión)                                    [xautomation]
     ├─ Coupon (instancia): folio único, código/QR firmado, estado, timestamps
     │    └─ Redemption (quién, dónde, cuándo, terminal)                       [xcanje]
     ├─ Raffle (sorteo)                                                        [xsorteos]
     │    ├─ RaffleTicket (boleto por cliente)
     │    └─ RaffleDraw (ejecución + ganador + semilla auditable)
     ├─ Machine (SAS)  ── MachineHistory (coin-in)         [xtgmconex/xmachinehistory]
     ├─ Product (catálogo de canje)                                           [xlogistics]
     ├─ FoodOrder (A&B)                                                        [xfoodorder]
     └─ Device (impresora, lector, kiosco, display)                           [xprint]
```

Estados del `Coupon`: `EMITIDO → ENTREGADO → CANJEADO | EXPIRADO | ANULADO`
(transición **atómica** en el canje para evitar doble uso).

---

## 13. Recomendaciones de arquitectura para TUWAYKIAPP

### 13.1 Identidad y multi-tenant (primero)
- **IdP central OAuth 2.0 / OIDC** multi-tenant, igual que Wargos. Opciones: **Keycloak**
  (self-host, replica bien su modelo), **Auth0/Clerk**, o **Supabase Auth**.
- Cada app (admin, runner, sorteos) = un *client* OIDC distinto (como sus `client_id`).
- **RBAC** por rol y por sede, y **aislamiento por `tenant_id`** en cada tabla (RLS si Postgres).

### 13.2 Separación de frentes (igual que Wargos)
- **Admin** (config + reportes) — SPA (Next.js/React).
- **Runner/operativo** — **PWA offline-tolerante** (cola local + sync): el counter no puede
  caerse si se cae la red. Wargos lo hizo en Angular; tú puedes en React/Angular/Svelte.
- **Selector de sorteos** — SPA de presentación a pantalla completa; anima con **three.js**
  o Canvas/Lottie si quieres más simple.

### 13.3 Backend: microservicios o "monolito modular" bien cortado
- No necesitas 30 servicios el día 1. Empieza con un **monolito modular** con los mismos
  **límites de dominio** que Wargos (auth, person, points, canje, print, sorteos, notif,
  automation, blacklist, statistic) y extrae a servicios cuando escale.
- **Dos preocupaciones que Wargos separó y conviene copiar**: `xcanje` (redención atómica) y
  `xprint` (impresión) como módulos aislados.

### 13.4 Núcleo de cupones robusto
- **Folio único + QR firmado** (HMAC/JWT) para validar sin BD, **+ estado central** como
  fuente de verdad contra doble canje.
- **Redención atómica**: `UPDATE … WHERE estado='EMITIDO'` en transacción (dos cajeros no
  canjean el mismo cupón).
- **Motor de reglas** parametrizable (elegibilidad/vigencia/límites) desacoplado del código.
- **Auditoría append-only** de emisión y canje (equivalente a `xactivityfeed`).

### 13.5 Sorteos auditables
- Aleatoriedad **verificable**: registra la **semilla**, usa CSPRNG, **firma** el resultado.
- **Padrón congelado** (snapshot) al momento del sorteo.
- Endpoints tipo Wargos: `sorteo-conf`, `participantes`, `sorteo-animation/:id`,
  `sorteo-winner`, `winner-view/:id`.

### 13.6 Automatización y notificaciones
- **Motor de automatización** por evento (cumpleaños, racha, inactividad, coin-in umbral)
  como `xautomation` → emite cupones/beneficios sin intervención del staff.
- **Notificaciones multicanal** (WhatsApp/email/push) como `xnotif`.

### 13.7 Observabilidad desde el día 1
- Copia su madurez: RUM + **session replay** + trazas + *synthetics*. Datadog, o alternativa
  open source (**Grafana/Faro + Prometheus/Loki/Tempo**) para abaratar.

### 13.8 Cómo **superar** a Wargos
- **Cupones 100% digitales** en la app del cliente con **wallet y QR dinámico** (rotatorio,
  anti-screenshot).
- **Segmentación con IA real** (propensión a canje, *next-best-offer*), no solo un chatbot.
- **Panel de fraude en tiempo real** (patrones de canje anómalos).
- **Reportes regulatorios nativos** (MINCETUR + Ley 29733) exportables.
- **Menos fricción operativa** que un runner con muchos pasos: flujos de 1–2 toques en counter.

---

## 14. Stack observado (✅) vs. sugerido para TUWAYKIAPP

| Capa | Wargos (✅ observado) | Sugerido TUWAYKIAPP |
|------|------------------------|----------------------|
| Admin | React CRA + axios + yup + moment | **Next.js** + React Query + zod + dayjs |
| Runner | **Angular** + RxJS + ESC/POS + Intercom | **PWA React/Next** offline + ESC/POS (o QZ Tray) |
| Sorteos | React + **three.js/r3f/drei** + MUI | React + three.js (o Lottie/Canvas) |
| Auth | IdP propio OAuth2/OIDC, multi-tenant, SAML (Premium) | **Keycloak** / Auth0, multi-tenant + RLS |
| API | 2 gateways (`api4`/`api5`) + ~30 micro `x*` (algún `go-*` en Go) | Monolito modular → microservicios al escalar |
| Búsqueda | **Algolia** | Algolia / **Meilisearch** (self-host) |
| Notif | WhatsApp/email/push (`xnotif`) | WhatsApp Cloud API + email + push |
| IA | `xparrot` (chat) | LLM + herramientas sobre tus datos |
| Observabilidad | **Datadog** (RUM/Replay/Synthetics/Profiler) | Datadog o Grafana stack |
| Soporte | **Intercom** | Intercom / Chatwoot |
| Media/CDN | **Cloudinary** + CloudFront | Cloudinary / S3+CloudFront |

---

## 15. Resumen de los tres enlaces

| Enlace | Qué es | Cómo funciona (✅) |
|--------|--------|---------------------|
| `accounts…/login?…client_id=DBqTw…&redirect_uri=app…/callback` | Login del **Admin** | Login HTML (POST /login) → OAuth Authorization Code → `/callback?code` → tokens JWT. Client React. |
| `accounts…/login?…client_id=zW9Xh…&redirect_uri=runner…/callback` | Login del **Operativo** | Mismo IdP, otro `client_id`, SSO. App **Angular** de sala (registro, cupones, canje, caja, A&B, eventos). |
| `sorteos…/selector` | **Selector de sorteos** | SPA **React + three.js**; API `api4/api/v2` (`sorteo-conf`, `participantes`, `sorteo-animation`, `sorteo-winner`, `winner-view`); soporta hot seat; white-label por sala. |

Todo comparte: **IdP central `accounts.wargos.cloud`**, `tenant=gaming`, **dos gateways
`api4`/`api5`** y **~30 microservicios `x<dominio>`**.

---

## 16. Qué falta por verificar (requiere tu login) — checklist

> **Actualización:** el **panel ADMIN** (ver
> **[PARTE II](#parte-ii--recorrido-autenticado-del-panel-admin--verificado-en-vivo)**) y el
> **runner operativo** (ver
> **[PARTE III](#parte-iii--recorrido-autenticado-del-runner--operativo--verificado-en-vivo)**)
> **ya fueron recorridos con sesión iniciada**. Solo queda observar un **sorteo ejecutándose en
> vivo** en el selector 3D.

**A) `app.wargos.cloud` (admin):** ✅ **COMPLETADO** (Parte II + **Parte IV**: plantilla/catálogo
de impresión, cola de trabajos de impresión con estados, multiplicadores/promociones, auditoría
con diff, submenú completo de fidelización). Lo único no accesible fue el módulo **ADMIN**
(usuarios/roles), bloqueado por RBAC para el usuario "Dan": si tienes un usuario con ese permiso,
entra a él y te documento roles/permisos.

**B) `runner.wargos.cloud` (operativo):** ✅ **COMPLETADO** (Parte III).
1. ✅ **Buscar/registrar cliente** — `xsearch/elastic` + formulario `/app/customer-register`;
   validación de prohibidos vía **escaneo de documento (app de ludopatía)** en el ingreso a sala.
2. ✅ **Emitir cupón** — formulario `/app/coupon-manual` (cliente + cantidades + zona de impresora
   + máquina). *No se pulsó Guardar* (regla: no modificar). El POST de emisión/impresión no se
   ejecutó, pero el formulario y su comportamiento quedaron documentados.
3. ⚠️ **Canjear** cupón/puntos: el canje se hace por **validación de código** (visto en admin,
   `/crm/coupon-search`) y por **auto-depósito en máquina**; no se ejecutó un canje real.
4. ✅ **Operación de sorteo** desde el counter — `/app/sorteos-list` con **Asignar ganador /
   Editar / Acta de sorteo** (no ejecutado).

**C) `sorteos.wargos.cloud/selector`:** ✅ **COMPLETADO** (Parte V) a nivel de estructura y
endpoints. Tres categorías (SORTEOS / TORNEOS / MISTERIOSOS=pozos progresivos), estados y API 3D
documentados. La **animación interna / semilla en vivo** no se ejecutó (bloqueo de seguridad +
regla de no lanzar draws); el artefacto de auditoría verificado es el **"Acta de sorteo"** del
runner.

> Alternativa: inicia sesión tú en el navegador y, con la sesión abierta, puedo recorrer las
> pantallas y documentarlas (sin extraer datos personales de clientes). Tú decides la vía.

---

---

# PARTE II — Recorrido autenticado del panel ADMIN (✅ verificado en vivo)

> Sesión iniciada por el usuario el **2026-08-16** (la contraseña la escribió él; yo no
> introduzco credenciales). Recorrido **solo lectura**: no se creó, editó ni eliminó nada.
> **No se extrajeron datos personales de clientes**; solo estructura, configuración,
> endpoints y métricas agregadas.

## II.0 Multisucursal (selección de sala)

Tras el login aparece **"Seleccione su sala"** con tarjetas por sucursal. En esta cuenta
(operador **oscorimasac**) hay 7 salas: **El Palacio, Huandoy, Mega Gaming, Pro, San Juan 1,
San Juan 2, San Juan 3**. Cada sala tiene su `salaId` (ej. El Palacio = **123**). Confirma el
modelo **multi-tenant → multi-sede**: un usuario ve varias salas y opera una a la vez.

El usuario (`users/me`) trae rol/permisos; **el módulo ADMIN estaba bloqueado por RBAC** para
este usuario (los botones existen pero no abren) — confirma control de acceso por rol
(coherente con los assets `403.png` / `AVOID.png`).

## II.1 Estructura real de navegación (8 módulos → prefijos de ruta)

La app es una SPA con un **rail de íconos** de 8 módulos; cada uno cambia el submenú y usa su
propio prefijo de ruta:

| Módulo (rail) | Prefijo | Contenido |
|---------------|---------|-----------|
| **CLIENTES** (CRM) | `/crm` | Clientes, **Cupones**, **Puntos**, **Sorteos**, Promociones, Canjes, Pedidos, Aplicación |
| **MÁQUINAS** | `/tgm` | Tragamonedas/SAS: contadores, producción, tickets, SUNAT, ludopatía, heatmap |
| **CAJA** | `/caja` | Cuadre, recaudo, billetero, pagos, gastos, **UIF (lavado de activos)** |
| **MESAS** | `/tables` | Juegos de mesa: dealer, score, sesiones, transacciones |
| **BINGO** | `/bingos` | Eventos, partidas, jackpots, venta de cartones |
| **ASISTENCIA** | `/asis` | Página única (asistencia de personal/anfitrionas) |
| **CONFIGURACIÓN** | `/crm/settings` | Parámetros de sala, impresoras, ticket, facturación, ludopatía, zonas, almacenes |
| **ADMIN** | (RBAC) | Usuarios/roles/administración — no accesible con este usuario |

### II.1.1 CLIENTES `/crm` — submenú completo (✅)
- **Clientes**: Lista (`/crm/clientes`), Cumpleaños, Online, Ranking, Resumen
  (`customer-resume`), Sesiones, Sesiones x empleado, Ingresos a sala (`historial-ingresos`),
  Ocupación por hora, Tarjetas (`card`).
- **Cupones**: Configuración (`coupon-types`), Carga masiva (`upload-coupons-list`), Cupones
  otorgados (`awarded-coupons`), Cupones acumulados (`accumulated-coupons`), Cupones generados
  (`generated-coupons`), Tareas de impresión (`tareas-impresion`), Multiplica (`coupon-promo`),
  Consulta cupones (`coupon-search`).
- **Puntos**: Configuración (`puntos-configuracion`), Club de puntos (`segments-list`),
  Multiplica (`puntos-promociones`), Carga masiva, Puntos otorgados, Reporte de puntos
  (`puntos-balance`).
- **Sorteos**: Lista (`sorteos`), Sorteo masivo (`sorteos-massive`), Análisis de ganadores
  (`analisis-ganador`), Estadísticas (`sorteos-statistics`), Ruleta (`roulette`).
- **Promociones**: Misteriosos (`progressive`, jackpots misteriosos), Recompensas (`rewards`),
  Cashback casinos (`cashback-config`).
- **Canjes**: Productos (`customer-catalog`), Reporte de canjes (`redeem-points`), Consultas
  (`redeem-custom-points`).
- **Pedidos** (A&B): Productos (`products`), Categorías (`food-categories`), Órdenes pendientes,
  Historial de órdenes, Reporte.
- **Aplicación**: Notificaciones (`app-notif-list`), Publicidad (`publicidad`), Portales (`sites`).

### II.1.2 MÁQUINAS `/tgm` (✅, resumen)
`meter-online` (Máquinas), `bill-online` (Cuadre), `meter-snap` (Historial contadores),
`accountants`, `machine-events`, `billetera`, `handpay-events` (pagos manuales), `cereos`,
`prod-sunat` (**SUNAT**), producción (total/diaria/mensual/por máquina/modelo/juego,
comparativo), `tickets`/`tickets-emitted`/`tickets-redeemed`, `layout-config`,
`layout-heatmap` (**zonas calientes**), `layout-event` (seguridad), `alerts`, **`ludopatia`**,
`asociados` (contratos), `taxes` (fórmulas), `transfers`, `shipping-economic` (envíos
datacenter), `devices`, `materials-list`, `machine-set` (**grupos de máquinas**).

### II.1.3 CAJA `/caja` (✅, resumen)
`report-period` (Fondos), `operations`, `audit-*` (recaudo, cuadre físico/diario, diferencias,
venta diaria), `ticket-report`/`ticket-promotional-report`, **`uif`** (lavado de activos),
`cash-back`, `c*-report` (tickets, handpay, cardpay, billetero, rellenos, tipo de cambio,
gastos, ajustes de saldo, transferencias, depósitos, invoices, sorteos), `records-report`
(fichas mesas), `configuration-caja`, `fund-type` (posiciones).

## II.2 Cupones — configuración real (✅ el núcleo de tu proyecto)

**Lista** (`/crm/coupon-types`): tarjetas por plantilla con pestañas **Nuevo / Vigentes /
Expirados / Próximos**. Reales: CUPON DERBY, CUPON DIARIO, CUPON ESTELAR, CUPON RULETA,
CUPON VIP — todos "Cupón físico", "Habilitado", con INICIO/FIN.
Endpoint de lista: `GET api4/xpointcore/coupon-type/grouped`.

**Formulario "Editar cupón"** (`/crm/coupon-types/edit/{id}`) — campos verificados:

| Campo (name) | Label | Tipo / valores |
|--------------|-------|----------------|
| `salaId` | (sede) | id de sala (ej. 123) |
| `enabled` | Habilitado | checkbox |
| `name` | Nombre | texto (ej. "CUPON DERBY") |
| `abrv` | Abreviación | texto (ej. "CD") |
| `correlativeEnabled` / `correlativeNext` | Correlativo / Siguiente | checkbox + número (folio) |
| `iniDate` + `iniTime` | Inicio de promoción | date + time |
| `finDate` + `finTime` | Fin de promoción | date + time |
| `gender` | Género | select **N=Deshabilitado / F=Femenino / M=Masculino** |
| `couponLimit` | Cantidad máxima durante el periodo | número |
| `couponLimitPerDay` | Cantidad máxima por día | número |
| (Monto mínimo / máximo / Máximo por jugada) | umbrales de emisión | número |
| `virtual` | (cupón virtual) | checkbox |
| `autoDeposit` | Depósito automático | checkbox |
| `printable` | Imprimible | checkbox |
| `templateId` | Plantilla | select (M1–M5, L3–L7, E1, D1…) |
| `autoPrint` | Impresión automática | checkbox |
| `printerId` | Impresora por defecto | select (impresoras físicas reales) |
| `autoPrintByRedeem` | Impresión automática de canje | checkbox |
| (coupon-zone-printer) | Impresoras por zonas | config por zona |
| `showAlias` | Mostrar alias | checkbox |
| `moduleShow` | Mostrar en el módulo de canje | checkbox |
| `modulePrint` | Habilitar botón de impresión | checkbox |
| (react-select) | Máquinas | multi-select (grupos/máquinas elegibles) |

**Endpoints que dispara el form de cupón** (✅):
```
GET api4/xpointcore/coupon-type/{id}
GET api4/xpointcore/coupon-type/{id}/expiration-policy      (política de expiración)
GET api4/xpointcore/coupon-zone-printer?couponTypeId={id}   (impresora por zona)
GET api4/xpointcore/coupon-type-promo/{id}/coupon-type      (promo asociada)
GET api4/xpointcore/entity-modifications/by-id/{id}?entity=COUPON_TYPE   (auditoría)
GET api5/xtgmconex/machine-groups                           (grupos de máquinas)
GET api5/xtgmconex/slot-zones                               (zonas de piso)
GET api5/xprint/printers/state-report?enabled=true&accountId={sala}  (impresoras)
```
> Aprendizajes: el cupón está **fuertemente acoplado a impresión térmica** (impresora por
> defecto y por zona, plantillas, auto-impresión, auto-impresión al canjear), a **elegibilidad
> por máquina/zona** (SAS), y tiene **folio correlativo**, **política de expiración** y
> **auditoría de cambios** (`entity-modifications`). La segmentación básica incluye **género**
> y límites por periodo/día.

**Otras pantallas de cupón (✅):**
- **Cupones generados** (`generated-coupons`): filtros Promoción/Desde/Hasta, "Presente ahora"
  / "Expirados". Endpoint `xpointcore/coupon-type/combo`.
- **Consulta de cupones** (`coupon-search`): campo **"Ingrese código del cupón" → Validar**
  (flujo de validación/redención por código).
- También: **Cupones otorgados**, **acumulados**, **Carga masiva** (`upload-coupons-list`),
  **Tareas de impresión** (`tareas-impresion`), **Multiplica** (`coupon-promo`).

## II.3 Puntos y Club de jugadores (✅)

- **Configuración de Puntos** (`/crm/puntos-configuracion`, `GET xpointcore/point-rules`):
  reglas del tipo **"1 PUNTO = S/ 20 apostados"** aplicadas a un grupo (MÁQUINAS, RULETA,
  DERBY…), con **máximo** y modo **ACUMULATIVO / en una jugada**; opción **"Expirar puntos"**.
- **Club de puntos** (`/crm/segments-list`, `GET xpointcore/player-club-config` +
  `customer-segments`): **niveles/tiers** con columnas **Nivel · Puntos · Valor de canje
  (puntos→S/) · Máx. días sin actividad**; **temporadas** (mensual, con día/mes de inicio),
  **subida y bajada de nivel automática**, modo **dinámico** (duración de periodo en días) y
  una pestaña **Simulación**.

## II.4 Sorteos (✅)

- **Lista** (`/crm/sorteos`, `GET xsorteos/sorteos` + `xsorteos/sorteo-conf`): filtros
  Desde/Hasta; KPIs **Número de sorteos · Monto total · N° Acta siguiente**; tabla con
  **ID · Fecha · Hora · Premio · Modo · Promoción · Rango · Acta · Ganador · Cupón · Máquina ·
  Acciones**. Se observó un sorteo **modo `HOT_SEAT`**. El campo **"Acta"** es el registro
  regulatorio del sorteo.
- **Formulario "Nuevo sorteo"** (✅): **Desde/Hasta** (ventana de acumulación de participación),
  **Tipo de sorteo** (toggle, normal vs. hot seat), **Fecha y hora del sorteo**, **premioTipo**
  (`CASH`=Efectivo / `Premio`), **Monto + Moneda** (`currencyId`/`currencyIso`), **Promoción**
  (vincula el sorteo a una promo/cupón).

## II.5 Canjes y Pedidos (A&B) (✅)

- **Productos de canje** (`/crm/customer-catalog`, `GET xpointcore/catalog-products/
  active-inactive`): tabla **ID · Imagen · Nombre · Canje por (PUNTOS) · Cantidad · Precio ·
  Stock · Habilitado** (ej. "Coca cola" = 1 punto). Filtros por estado Activos/De baja.
- **Reporte de canjes** (`redeem-points`) y **Consultas** (`redeem-custom-points`).
- **Pedidos** (A&B): catálogo de **Productos** y **Categorías** de comida, **Órdenes
  pendientes / Historial / Reporte** (el servicio `xfoodorder`).

## II.6 Configuración de sala (✅)

`/crm/settings` con secciones: **Mi perfil** (Usuario, Email, Nombres, Apellidos, Documento,
Teléfono, Contraseña), **Sala, Servidores, IPs permitidas, Datacenter, Facturación,
Parámetros de logueo, Ticket, Sorteos, Impresoras, Pedidos, Categorías de gasto, Layout,
Ludopatía, Fabricantes y modelos, Zonas, Almacenes, Notificaciones**.

## II.7 Dashboard de sala (✅)

Al entrar a una sala: **Producción del día**, **Total apostado**, **RTP %** (retorno al
jugador), **Últimos ingresos** (clientes), **Ocupación (logueados vs anónimos)** y un
**mapa de la sala** editable (layout). Monitoreo en tiempo real de producción/apuestas.

## II.8 Catálogo de endpoints reales observados (✅)

```
# Identidad / sala
GET api5/xuser/users/me?product=gaming        GET api5/xuser/users/me-info
GET api5/go-salas/salas-config                 GET api5/go-salas/banner-by-sala
GET api5/xcaja/config-caja
# Cupones (xpointcore)
GET api4/xpointcore/coupon-type/grouped        GET api4/xpointcore/coupon-type/{id}
GET api4/xpointcore/coupon-type/{id}/expiration-policy
GET api4/xpointcore/coupon-zone-printer?couponTypeId={id}
GET api4/xpointcore/coupon-type-promo/{id}/coupon-type
GET api4/xpointcore/coupon-type/combo
GET api4/xpointcore/entity-modifications/by-id/{id}?entity=COUPON_TYPE
# Puntos y club
GET api4/xpointcore/point-rules                GET api4/xpointcore/player-club-config
GET api4/xpointcore/customer-segments          GET api4/xpointcore/catalog-products/active-inactive?enabled=true
# Sorteos (xsorteos)
GET api4/xsorteos/sorteos                       GET api4/xsorteos/sorteo-conf
# Máquinas / impresión
GET api5/xtgmconex/machine-groups              GET api5/xtgmconex/slot-zones
GET api5/xprint/printers/state-report?enabled=true&accountId={sala}
```
> Patrón: **`xpointcore` concentra cupones + puntos + club + catálogo de canje** (no hay un
> "xcoupon" separado). Sorteos en `xsorteos`. Impresión en `xprint`. Máquinas/zonas en
> `xtgmconex`. Sala en `go-salas`. Usuarios en `xuser`.

## II.9 Implicaciones para TUWAYKIAPP (afinadas con lo verificado)

1. **El cupón NO es solo un código**: es una **plantilla** con folio correlativo, ventana de
   vigencia con hora, límites por periodo/día, umbrales de monto, segmentación (género/
   máquinas/zonas), **política de expiración**, y todo el **subsistema de impresión térmica**
   (plantilla, impresora por defecto y por zona, auto-impresión, auto-impresión al canjear).
   Modela `CouponTemplate` con esos campos desde el inicio.
2. **Auditoría nativa** (`entity-modifications`): registra cada cambio de configuración por
   entidad. Cópialo (tabla append-only de modificaciones).
3. **Puntos = motor de reglas** "S/ apostado → puntos" por grupo de máquina, con tope y modo;
   y **club con tiers dinámicos** (subida/bajada automática, temporadas, expiración por
   inactividad, valor de canje puntos→dinero). Es más que niveles estáticos.
4. **Sorteos con acta regulatoria** y modo **hot seat**; premio efectivo o producto, vinculado
   a promoción, con ventana de acumulación. El "Acta" (número correlativo) es clave para
   MINCETUR.
5. **RBAC por módulo y por sala** real (ADMIN gateado). Diseña permisos finos desde el día 1.
6. **Un solo `xpointcore`** unifica cupones/puntos/club/canje: te conviene un **bounded context
   "Loyalty"** que agrupe esos cuatro, en vez de servicios separados que se llaman mucho entre sí.

---

# PARTE III — Recorrido autenticado del RUNNER / operativo (✅ verificado en vivo)

> Sesión iniciada por el usuario en `runner.wargos.cloud` (usuario "DANNY", sala **El Palacio**),
> el 2026-08-16. **Solo lectura**: no se emitió ni canjeó ningún cupón, no se ejecutó ningún
> sorteo, no se imprimió, no se guardó nada, y **no se registraron datos personales** de clientes
> (solo estructura, campos y endpoints). Stack confirmado: **Angular + Angular Material**
> (bundles `main-*.js`, `polyfills-*.js`, clases `mat-*`).

## III.0 Arranque y selección de sala

- Login OAuth con `client_id` propio del runner → `runner.wargos.cloud/callback`.
- **Obliga a elegir sala** antes de operar (pantalla "Salas": El Palacio, Huandoy, Mega Gaming,
  Pro, San Juan 1/2/3). Tras elegir, aterriza en `/app/config` (vista viva de máquinas).
- **Toda** pantalla dispara en cada carga: `GET api5/xtgmconex/machines/find-by-conf/micro/all`
  (estado de máquinas) y `GET api5/xuser/users/me?product=gaming` (perfil/permisos).

## III.1 Estructura de navegación (menú operativo → rutas)

| Sección (menú) | Ruta | Qué es |
|---|---|---|
| **Resumen** | `/app/live-summary` | KPIs en vivo de la sala |
| **Producción** | `/app/production-sala` | Producción de juego en tiempo real |
| ├ Máquinas | `/app/machine-list` | Listado/estado de máquinas |
| └ Modelos | `/app/prod-average` | Producción promedio por gabinete/modelo |
| **Clientes** | `/app/customer-home` | CRM de sala + analítica |
| ├ Ingreso de clientes | `/app/customer-ingress` | Bitácora de accesos (escaneo de documento) |
| └ Asignación manual | `/app/coupons-awarded` → `/app/coupon-manual` | Otorgar cupones desde caja |
| **Cupones** | `/app/coupon-list` | Cupones vigentes/expirados/próximos |
| **Sorteos** | `/app/sorteos-list` | Sorteos del día (operación) |
| **Billetero** | `/app/bill-online` | Reporte de billeteros por máquina |
| **Pedidos** | `/app/order-list` | Órdenes de cortesía (A&B) |
| **Rellenos** | `/app/refill-list` | Recargas de efectivo en máquinas |

## III.2 Cupones — vista operativa (✅ tu núcleo)

- **`/app/coupon-list`** → `GET api4/xpointcore/coupon-type/grouped`. Tres pestañas
  (**Cupones / Sorteos / Asignación manual**) y filtros **Vigentes / Expirados / Próximos**.
  Cada cupón muestra nombre, tipo ("**Cupón Físico**") y "Válido hasta el …". Ejemplos activos
  en El Palacio: CUPON DERBY, DIARIO, ESTELAR, RULETA, VIP.
- **`/app/coupons-awarded`** = bitácora diaria de asignaciones manuales con navegador de fecha
  (◀ hoy ▶). Botón **Nuevo / Nueva asignación**.

### III.3 Formulario de emisión de cupón — `/app/coupon-manual` (✅ el flujo de caja)

Este es el flujo real del cajero para **otorgar cupones a un cliente**. Campos:

| Campo | Detalle |
|---|---|
| **Cliente** | Botón **Buscar** → abre modal "Buscar cliente" (con botón **Nuevo** para alta) |
| **Cupones** | Lista de tipos con **stepper de cantidad** (`− / +`) por tipo (DERBY, DIARIO, ESTELAR, RULETA, VIP) |
| **Impresora** | Selector "**Seleccione una zona**" (zona de impresión térmica) |
| **Máquinas (Opcional)** | Buscar/asociar la máquina donde juega el cliente |
| **Guardar** | Confirma la emisión (⚠️ no se pulsó) |

- **Búsqueda de cliente** (modal): `GET api5/xsearch/elastic?query={texto}` → búsqueda
  **ElasticSearch** por documento/nombre. El modal también ofrece **Nuevo** (alta rápida).
- Insight: la emisión ata en un mismo acto **cliente + N cupones + zona de impresora + máquina**,
  o sea el cupón operativo nace **impreso** y opcionalmente **vinculado a la máquina/sesión**.

## III.4 Clientes (CRM de sala)

- **`/app/customer-home`**: acciones **Buscar · Registrar · Ranking · Cumpleaños** + analítica
  (distribución por **género** y por **edades**). Endpoints:
  `GET api4/xpointcore/customers/genders-by-salaid`,
  `GET api4/xpointcore/customers/ages-by-salaid?ini=&fin=`.
- **`/app/customer-register`** — alta de cliente. Campos (formcontrol):
  `dni` (Tipo Documento = DNI por defecto), `firstName` (Nombres), `lastName1` (Ap. Paterno),
  `lastName2` (Ap. Materno), `alias` (opcional), fecha de nacimiento (`dia`/`mes`/`ano`),
  **Género**, **Estado Civil**, `lugar` (Departamento/Prov/Distrito), `address` (Dirección),
  `celular`, `email`, y **Preferencias** (textarea).
- **`/app/customer-ingress`** = **bitácora de ingresos a la sala**:
  `GET api4/xpointcore/customers/ingress?ini=&fin=`. El texto en pantalla indica que se puebla
  **escaneando el documento con la "app de ludopatía"** → control de acceso ligado a **juego
  responsable / Registro de Personas Prohibidas** (regulatorio). Hallazgo relevante: el registro
  de cliente y el control de aforo/ludopatía comparten el mismo dominio `xpointcore`.

## III.5 Sorteos — operación desde caja

- **`/app/sorteos-list`** → `GET api4/xsorteos/sorteos?ini={fecha}`. Lista los sorteos del día
  (ej.: **HOT_SEAT · PREMIO S/ 50 · 12:00 p.m.**). Botón **Nuevo** y menú `⋮` por sorteo con:
  **Asignar ganador · Editar · Acta de sorteo**.
  - **Asignar ganador**: ejecuta/resuelve el sorteo (⚠️ no se pulsó).
  - **Acta de sorteo**: genera el **acta regulatoria** (número correlativo — MINCETUR).
- Complementa a `sorteos.wargos.cloud/selector` (la tómbola 3D pública): el runner **administra**
  el sorteo, el selector lo **anima/proyecta**.

## III.6 Operación de dinero y servicios (módulos de apoyo)

| Módulo | Ruta | Endpoint | Qué reporta |
|---|---|---|---|
| **Resumen ahora** | `/app/live-summary` | `xpointcore/session/live-summary`, `xpointcore/session/summary`, `xmachinehistory/machine-played-history` | Clientes en sala (logueados/anónimos/no participa) y **"Cobertura apostado"** = % del juego atribuido a clientes identificados (KPI de fidelización) |
| **Producción** | `/app/production-sala` | `xtgmconex/meter-online/realtime-summary`, `xstatistic/statistic/prod-total`, `xtgmconex/deltas` | Producción neta, Coin In, RTP, producción por máquina en tiempo real (lectura SAS) |
| **Modelos/Gabinetes** | `/app/prod-average` | `xstatistic/statistic/prod-average?ini=&fin=` | Producción promedio mensual por modelo de gabinete (ranking de rentabilidad, en USD) |
| **Billetero** | `/app/bill-online` | `xcaja/report-bill/online` | Dinero ingresado por billetero, por máquina y por denominación (S/10–200) |
| **Pedidos** | `/app/order-list` | `xfoodorder/food-orders?ini=&fin=` | Órdenes de cortesía (A&B) que llegan a la barra |
| **Rellenos** | `/app/refill-list` | `xcaja/refill?ini=&fin=` | Recargas de efectivo en máquinas |

## III.7 Catálogo de endpoints del runner (✅ observados en vivo)

```
# Comunes a toda pantalla
GET  api5/xtgmconex/machines/find-by-conf/micro/all
GET  api5/xuser/users/me?product=gaming

# Cupones
GET  api4/xpointcore/coupon-type/grouped
# (emisión desde /app/coupon-manual → Guardar; POST no ejecutado)

# Clientes
GET  api5/xsearch/elastic?query={texto}                      # búsqueda ElasticSearch
GET  api4/xpointcore/customers/genders-by-salaid
GET  api4/xpointcore/customers/ages-by-salaid?ini=&fin=
GET  api4/xpointcore/customers/ingress?ini=&fin=             # bitácora acceso (ludopatía)
# (alta cliente desde /app/customer-register → Guardar; POST no ejecutado)

# Sesión / fidelización
GET  api4/xpointcore/session/live-summary
GET  api4/xpointcore/session/summary

# Sorteos
GET  api4/xsorteos/sorteos?ini={fecha}

# Producción / máquinas (SAS)
GET  api5/xtgmconex/meter-online/realtime-summary
GET  api5/xtgmconex/deltas?ini=&fin=
GET  api5/xstatistic/statistic/prod-total?ini=&fin=
GET  api5/xstatistic/statistic/prod-average?ini=&fin=
GET  api5/xmachinehistory/machine-played-history?ini=&fin=

# Caja / servicios
GET  api5/xcaja/report-bill/online
GET  api5/xcaja/refill?ini=&fin=
GET  api5/xfoodorder/food-orders?ini=&fin=
```

## III.8 Implicaciones para TUWAYKIAPP (del lado operativo)

1. **El runner es el "punto de venta" del cupón.** Diseña una app de caja separada del admin,
   optimizada para: buscar cliente en 1 campo (ElasticSearch por DNI/nombre) → elegir cantidades
   por tipo de cupón → elegir zona de impresora → (opcional) máquina → **emitir + imprimir** en
   un solo gesto. Debe funcionar rápido y con impresión térmica local.
2. **Búsqueda del cliente = un servicio de indexado dedicado** (ellos usan ElasticSearch vía
   `xsearch`). Para escala, no busques por LIKE en SQL; indexa (Elastic/Meilisearch/Postgres FTS).
3. **Alta de cliente mínima pero regulatoria**: DNI + nombres/apellidos + nacimiento + género +
   estado civil + contacto + ubicación. El **ingreso a sala vía escaneo de documento** cruza con
   **ludopatía/personas prohibidas**: si tu proyecto opera en sala física, este control es
   obligatorio, no opcional.
4. **KPI "Cobertura apostado"** (juego identificado / juego total): es *la* métrica que justifica
   todo el sistema de cupones. Instrumenta desde el día 1 el vínculo sesión-de-juego ↔ cliente
   (login en máquina) para poder medirla.
5. **El cupón operativo vive junto a la máquina/sesión** (se emite asociado a máquina y se puede
   canjear/depositar en juego). Tu modelo `Coupon` necesita `machineId`/`sessionId` opcionales,
   no solo `customerId`.
6. **Todo es multi-sala con fecha**: cada endpoint operativo filtra por sala activa y por rango de
   fechas. Diseña tenancy por sala + particionado temporal desde el inicio.

---

# PARTE IV — Segundo recorrido del ADMIN: impresión, estados y promociones (✅ verificado en vivo)

> Segunda sesión autenticada en `app.wargos.cloud` (usuario "Dan", sala El Palacio), 2026-08-16,
> para cerrar tres huecos del sistema de cupones: **plantilla/impresión**, **estados del cupón**
> y **vínculo con promociones**. Solo lectura; **no se guardó ni canjeó nada**; no se registran
> datos personales (los nombres de clientes vistos en tablas **no** se transcriben).

## IV.0 Submenú completo de fidelización (más rico de lo documentado)

El módulo CLIENTES (`/crm`) agrupa toda la suite de fidelización:

| Grupo | Ítems (ruta) |
|---|---|
| **Cupones** | Configuración (`/crm/coupon-types`) · Carga masiva (`/crm/upload-coupons-list`) · Cupones otorgados (`/crm/awarded-coupons`) · Cupones acumulados (`/crm/accumulated-coupons`) · Cupones generados (`/crm/generated-coupons`) · **Tareas de impresión** (`/crm/tareas-impresion`) · **Multiplica** (`/crm/coupon-promo`) · Consulta cupones (`/crm/coupon-search`) |
| **Puntos** | Configuración · Club de puntos · Multiplica (`/crm/puntos-promociones`) · Carga masiva · Puntos otorgados (`/crm/puntos-otorgados`) · Reporte de puntos |
| **Sorteos** | Lista · Sorteo masivo · Análisis de ganadores · Estadísticas · **Ruleta** |
| **Promociones** | **Misteriosos** · **Recompensas** (`/crm/rewards`) · **Cashback casinos** (`/crm/cashback-config`) |
| **Canjes** | Productos (`/crm/products`) · Reporte de canjes · Consultas |
| **Pedidos** (A&B) | Productos · Categorías · Órdenes pendientes · Historial · Reporte |
| **Aplicación** | Notificaciones · Publicidad · Portales |

> **Ciclo de vida del cupón** (se deduce de los reportes): **acumulado** (derecho ganado por
> juego/puntos) → **generado** (instancia creada) → **otorgado/impreso** (entregado al cliente) →
> **canjeado** o **expirado**. Los reportes `awarded/accumulated/generated-coupons` filtran por
> **Presente ahora / Expirados** + rango de fechas + promoción.

## IV.1 Plantilla e impresión (cierra el hueco "qué imprime")

- **`Plantilla` (`templateId`)** es un **catálogo fijo de plantillas ESC/POS predefinidas**, no un
  diseñador visual. Opciones completas: **M1, M2, M3, M4, M5, L3, L4, L5, L6, L7, E1, D1, D3, D4,
  D5** (familias M/L/E/D). Se elige por código; el layout físico ya está definido en el firmware
  del servicio de impresión (`xprint`).
- **`Impresora por defecto` (`printerId`)** y **`Impresoras por zonas`**: hay **una impresora
  física por tipo de cupón/zona**. Formato real observado: `{id} | {ubicación} | {NOMBRE_ZONA}` →
  ej. `1323 | BIOMETRICOPAL | DERBY_51`, `1247 | … | DIARIO_53`, `1316 | … | ESTELAR_55`,
  `1248 | … | RULETA_60`, y `1714 | … | RESPALDO_49` (**impresora de respaldo**).
- **Auditoría del cupón** (pestaña *Historial* dentro del editor): registra cada cambio con
  **valor anterior → nuevo**, autor y hora. Ej.: *"16-JUL · Jose Espinoza · 11:32 · modificó el
  cupón · Mostrar en módulo de canje: Sí → No"*. Es el `entity-modifications` con diff completo.

### IV.2 Cola de impresión — `/crm/tareas-impresion` (✅ el modelo asíncrono clave)

La emisión de un cupón **no imprime sincrónicamente**: **encola un trabajo de impresión**
(`coupon-print-job`) que un agente/impresora procesa. Endpoint:
`GET api4/xpointcore/coupon-print-job/report` (+ `coupon-type/combo` para el filtro).

- **Filtros**: Cliente · Tipo de Cupón · **Estado de Cupón: TODOS / NUEVO / ENVIADO / ERROR /
  COMPLETADO**.
- **Columnas**: `ID` · Cliente · Promoción · **Cantidad** · Fecha · **Estado** · **Mensaje** ·
  Acciones.
- **Máquina de estados del print-job**: `NUEVO → ENVIADO → COMPLETADO (DONE)` o **`ERROR`** con
  **mensaje diagnóstico** (ej. observado: *"Papel agotado"*).

> **Para TUWAYKIAPP:** modela una entidad **`PrintJob`** separada del cupón:
> `{id, couponInstanceId, customerId, promoName, cantidad, printerId, estado(NUEVO|ENVIADO|
> COMPLETADO|ERROR), mensaje, createdAt}`. Dispatch asíncrono + reintento. Esto desacopla la caja
> de la impresora (si falta papel, el cupón existe y se reintenta).

## IV.3 Promociones — "Multiplica" / Multiplicadores (cierra el vínculo cupón↔promo)

`/crm/coupon-promo` → **Multiplicadores**. Endpoints: `GET api4/xpointcore/coupon-type/promos`,
`GET api4/xpointcore/coupon-type-promo/{id}/coupon-type`.

Cada promoción (lista): **Nombre** · **Multiplicador (× N)** · **Horario** (días + franja) ·
**Máquinas participantes** (conteo, ej. 134 / 106) · Editar/Eliminar. Ejemplos: CUPON EXPERT,
ELECTRO-EXPERT, CUPON DIARIO., CUPON ESTELAR.

**Formulario** (`/crm/coupon-promo/edit/{id}`) — campos:
- **Datos generales**: `Promoción` (ligada a un **tipo de cupón**), `Deshabilitado` (toggle),
  **`Multiplicador (X)`**, **`Máquinas`** (multi-select).
- **Configuración**: **`Días`** (LUN–DOM, checkboxes), **`Horas`** (Desde/Hasta) con botón
  **Agregar** (varias franjas horarias) → una promo puede tener múltiples ventanas.

> **Hallazgo de arquitectura:** el ID de la promoción es un **Mongo ObjectId**
> (`6996307d0e138b1f40991854`) → **`xpointcore` está respaldado (al menos en parte) por MongoDB**.
> Coincide con un dominio de *loyalty* con documentos flexibles (cupones/promos/reglas).

## IV.4 Otros motores de la suite (contexto, no núcleo de cupones)

- **Recompensas** (`/crm/rewards`) y **Misteriosos**: promociones de premio (mystery/rewards).
- **Cashback casinos** (`/crm/cashback-config`): devolución de % de lo jugado.
- **Ruleta** (sorteos): variante de sorteo tipo ruleta.
- **Canjes → Productos** (`/crm/products`): catálogo de canje de puntos por productos.
- **Puntos → Multiplica** (`/crm/puntos-promociones`): el mismo patrón de multiplicador, aplicado a
  **puntos** (no solo cupones) → confirma que cupones y puntos comparten el motor de promociones.

## IV.5 Endpoints nuevos observados (Parte IV)

```
GET  api4/xpointcore/coupon-type/promos                       # lista de multiplicadores
GET  api4/xpointcore/coupon-type-promo/{id}/coupon-type       # promo → tipo de cupón
GET  api4/xpointcore/coupon-print-job/report                  # cola de impresión (estados)
GET  api4/xpointcore/coupon-type/combo                        # combo de tipos (para filtros)
GET  api5/xcaja/config-caja                                   # configuración de caja
GET  api5/go-salas/salas-config
GET  api5/go-salas/banner-by-sala
```

## IV.6 Implicaciones adicionales para TUWAYKIAPP

1. **Separar `PrintJob` del `Coupon`** con cola y estados (`NUEVO/ENVIADO/COMPLETADO/ERROR` +
   mensaje). La caja emite → encola; el agente de impresora consume y reporta. Habilita reintentos
   y diagnóstico ("papel agotado") sin bloquear al cajero.
2. **Plantillas como catálogo cerrado** (no editor libre): define un set de plantillas ESC/POS
   versionadas por código; más simple y robusto que un diseñador visual.
3. **Impresora por zona/tipo + respaldo**: el ruteo de impresión es por zona con fallback. Modela
   `Printer{zona, tipoCupon, esRespaldo}`.
4. **Motor de promociones unificado (Multiplicadores)** que aplica a **cupones y a puntos**, con
   `multiplicador × ventanas horarias × máquinas participantes`. Diseña un `PromoRule` reutilizable
   por ambos dominios.
5. **Persistencia poliglota**: ellos usan **MongoDB** para el dominio de loyalty (documentos
   flexibles: cupones, promos, reglas) + SQL/otros para transaccional. Evalúa el mismo split, o
   Postgres con JSONB si prefieres un solo motor.
6. **Auditoría con diff (antes→después)** nativa en cada entidad de configuración. Imprescindible
   para cumplimiento y soporte.

---

# PARTE V — Selector de sorteos en vivo (✅ verificado, ⚠️ sin ejecutar draws)

> Sesión autenticada en `sorteos.wargos.cloud/selector` (client_id propio del portal de sorteos;
> no arrastra SSO — pide login aparte). Sala El Palacio, 2026-08-16. **No se lanzó ningún sorteo**:
> el clasificador de seguridad bloqueó interactuar con el card del sorteo (no puede distinguir
> "ver uno ya sorteado" de "lanzar uno") y, además, nuestra regla es no ejecutar draws. Se
> documentó la **estructura y los endpoints**, no la animación interna.

## V.1 Flujo del selector

1. **Login OAuth propio** → 2. **Seleccione una sala** (mismas 7 salas) → 3. **`/selector` /
   `/sorteo-list`** con tres pestañas.
2. Carga inicial: `GET api5/xuser/users/me?product=gaming` + `GET api4/xsorteos/sorteo-conf`.

## V.2 Tres categorías (pestañas)

| Pestaña | Contenido observado | Endpoint |
|---|---|---|
| **SORTEOS** | Sorteos programados del día. Card: **hora (12:00 PM) · tipo (HOT SEAT) · nombre (SORTEO HOT SEAT) · premio (S/ 50) · estado (SORTEADO)** | `xsorteos/sorteo-conf` |
| **TORNEOS** | "SIN TORNEOS ACTIVOS" hoy (competencias por ranking) | `xsorteos/torneos/active/v2` |
| **MISTERIOSOS** | **Pozos progresivos** (PROGRESIVO WARGOS · POZOS · 3) ligados a máquinas | `xtgmconex/progressive` |

- **Estado del sorteo**: `SORTEADO` (ya ejecutado) vs. pendiente. Es la vista de **proyección/
  pantalla grande** para el público de la sala; la **administración** del sorteo (crear, asignar
  ganador, generar **Acta**) se hace desde el **runner** (ver §III.5).
- **API de sorteos (de bundles, `/api/v2/`)**: `sorteo-list`, `sorteo/:id/participantes`,
  `sorteo/:id/machines`, `sorteo-animation/:id`, `sorteo-winner`, `winner-view/:id`, variantes
  `hotseat`. La **tómbola 3D** (three.js/react-three-fiber) consume `sorteo-animation/:id` y
  muestra el ganador con `winner-view/:id`.

## V.3 Lo que NO se pudo verificar (y por qué)

- **Semilla / traza de auditoría del sorteo en vivo**: requeriría abrir/lanzar un draw, que está
  bloqueado por seguridad y por nuestra regla. **Sustituto verificado**: el runner genera un
  **"Acta de sorteo"** con número correlativo (§III.5) — ese es el artefacto de auditoría
  regulatorio. Para el diseño propio, definiremos semilla registrable + acta (ver Plan Técnico).

> **Para el Sistema de Cupones esto no es bloqueante**: los sorteos son un consumidor del cupón
> (el cupón otorga participaciones), no parte del núcleo de emisión/canje. Queda documentado a
> nivel de estructura y endpoints, suficiente para integrarlo.

---

# PARTE VI — MÁQUINAS › Historial de contadores `/tgm/meter-snap` (✅ verificado en vivo)

> Sesión iniciada por el usuario en `app.wargos.cloud`, sala **El Palacio** (`x-sala-id: 123`),
> 2026-10-24. Solo lectura: se filtraron **5 máquinas al azar** (una por modelo distinto) y un
> **rango de 5 días**. **No se pulsó "Analizar" ni ninguna acción de fila**, porque todas escriben
> (§VI.6). Los endpoints se capturaron interceptando las llamadas XHR en el navegador. La lógica
> de la pantalla se leyó del bundle `static/js/693.ded3fcda.chunk.js` (componente del historial) y
> las URLs base de `main.4a76ac0e.js`.

## VI.1 Cómo obtienen los datos

```
Máquina (protocolo SAS) ── SMIB "Pangolin" (blueSn) ──► xtgmconex (api5)
   │  "foto" (snap) de TODOS los contadores acumulados:
   │   · cada hora en punto (createdAt = hh:00 + 4–11 s)                 ✅
   │   · y por evento: Stacker (11), Puerta abierta (17),
   │     Puerta cerrada (18), Manual (20), Otro                          ✅
   ▼
meter_snap: una fila por foto = acumulados + prev* + inc* + validaciones,
            calculados en el servidor al grabar                          ✅
   ▼
GET /xtgmconex/meter-snap/by-dates?ini&fin&confId ──► el frontend solo pinta (gráfico + tabla)
```

- **El navegador no calcula nada.** Cada fila trae el acumulado (`ci`), el valor de la foto
  anterior (`prevCi`) y el incremento (`incCi = ci − prevCi`, comprobado en todas las filas).
- **Los contadores vienen en créditos.** Para pasarlos a soles se multiplica por `machine.den`
  (0.01 en las 5 máquinas). Comprobado con el billetero de la 129767:
  1 344×10 + 2 166×20 + 1 982×50 + 2 921×100 = S/ 447 960 = `bi` 44 796 000 × 0.01.
- 🔷 Que el SMIB sea el origen se infiere de `blueSn` (el mismo bundle lo llama "Pangolin (SMIB)"
  en el modal "Cambiar dispositivo") y del flag `connected`. El transporte SMIB→nube no se ve
  desde el navegador.

## VI.2 Endpoints de la pantalla (✅ capturados)

Cabeceras en todas: `Authorization: Bearer <token OAuth>` · `x-sala-id: 123` · `Accept:
application/json`. `x-sala-id` es una cabecera global de axios: se fija al elegir sala y se
guarda en `localStorage.salaId`.

| # | Cuándo | Llamada | Respuesta |
|---|---|---|---|
| 1 | Arranque de la app (toda pantalla) | `GET api5/xuser/users/me?product=gaming` · `GET api5/xcaja/config-caja` · `GET api5/go-salas/salas-config` · `GET api5/go-salas/banner-by-sala` | usuario y roles, config de caja y de sala |
| 2 | Al elegir sala (solo si el rol tiene `restrictIp`) | `GET api5/xtgmconex/access` (con `x-sala-id`) | `{access: bool}` |
| 3 | Al montar el selector de máquina | `GET api5/xtgmconex/machines/combo` | array de 115 máquinas `{id, confId, machine, machineLink, serverLink, slotGroup, manufacturerId}` · 15 KB · ~180 ms |
| 4 | Buscar, flechas de día, Hoy / Día / Personalizado | `GET api5/xtgmconex/meter-snap/by-dates?ini=YYYY-MM-DD&fin=YYYY-MM-DD&confId=<confId>` | `{machine: {…}, data: [foto…]}` |

Detalles de la llamada 4:
- El selector usa `value = confId` y `label = machine`, y **no siempre coinciden**. La máquina
  "147346" tiene `confId=47346`, y su consulta volvió con `data: []` ("No hay datos").
- **Cada búsqueda dispara la misma petición dos veces**: el `onClick` de Buscar llama a
  `fetchData` y un `useEffect` sobre los parámetros vuelve a llamarla. Pasó con las 5 máquinas.
- **No hay paginación en el servidor.** 1 día ≈ 8–9 filas, ~8 KB, ~350 ms; **5 días = 107 filas,
  104 KB, 2,7 s** para una sola máquina. La tabla pagina en el cliente (200 filas por página).
- **Ventana de tiempo observada:** empieza a las **04:00 del día `ini`**. Un día solo = de 04:00 a
  la hora actual; el rango 20→24 trajo el día 20 desde las 04:00 y el 24 desde las 00:00. El
  límite final no se pudo comprobar porque `fin` era hoy.

## VI.3 Las 5 máquinas filtradas (24-10-2026)

| confId | Etiqueta | Modelo · fabricante | Juego | Tipo | serverLink / machineLink / SMIB | Filas | Lo observado |
|---|---|---|---|---|---|---|---|
| 10593 | 10593 | TANGRA TOUCH · CASINO TEC | GAMOPOLIS 77 | 1 | no / no / — | 8 | billete de S/ 20 a las 11:00; `diffTo −2000` |
| 113334 | 113334 | IGT CRYSTAL DUAL SLANT · IGT | MULTIJUEGO | 1 | sí / no / — | 8 | `diffTi −1` y `diffTo −1` a las 09:00 |
| 47346 | **147346** | CASINO TEC - NEXT · CASINO TEC | DIAMOND KING III | 1 | no / no / — | **0** | sin fotos (etiqueta ≠ confId, `zoneId` nulo) |
| 54836 | 54836 | GOLD CLUB FUTURA PRO · GOLD CLUB | RULETA | **2** | no / no / — | 8 | sin movimiento en el día |
| 129767 | 129767 | MERKUR STC 230 · MERKUR | MULTIJUEGO | 1 | sí / sí / **presente** | **9** | foto extra **06:09 "Stacker"**; única con `connected=true` y con billetes por denominación |

Referencia previa: la 1022 (AGI FV 880 CF2) dio 8 filas y Coin In +2 320 a las 09:00. No hay
datos personales: solo contadores de máquina.

## VI.4 Diccionario de la foto de contadores (`data[]`)

| Grupo | Campos | Significado |
|---|---|---|
| Identidad | `id`, `confId`, `salaId`, `fechaHora`, `createdAt` | `id` correlativo global (~212 millones); hora nominal de la foto y hora real de grabación |
| Jornada | `realFecha`, `closeDay` | **Día contable**: cambia a las **08:00** (lo de 00:00–07:59 cuenta para el día anterior). `closeDay=true` marca la **foto de cierre** del día (§VI.5) |
| Origen | `sourceId`, `manual`, `localSnap` | 0 = foto horaria ("--"), 11 Stacker, 17 Puerta abierta, 18 Puerta cerrada, 20 Manual; cualquier otro = "Otro" |
| Contadores SAS (acumulados, en créditos) | `ci` Coin In · `co` Coin Out · `jp` Jackpot · `cc` Cancel Credit · `bi` Billetes · `ticketIn` · `ticketOut` · `hp` Handpay · `m30` "Pagos ext." · `gp` Jugadas · `cr` créditos en pantalla · `tiq`/`toq` cantidad de tickets in/out · `tci`/`tco` · `m23 m24 m29 m47 m51` (otros códigos SAS, en 0) | las columnas de la tabla |
| Billetero | `b10 b20 b50 b100 b200` | **cantidad de billetes** por denominación (acumulada) |
| Foto anterior | `prevCi prevCo prevJp prevCc prevBi prevTi prevTo` | valores de la foto previa |
| Incrementos | `incCi incCo incJp incCc incBi incTi incTo incGp incTiq incToq incPeb` | `X − prevX`; la tabla los muestra en una columna chica junto a cada contador (verde > 0, rojo < 0) |
| Validación | `consistent`, `consistentDiff` | ecuación de créditos (§VI.5) |
| Conciliación TITO | `diffTi`, `diffTo`, `tir tor tirq torq`, `ticketInPhysical`, `ticketOutPhysical`, `quantityTiP/ToP` | 🔷 contador SAS de tickets vs. tickets registrados. En las 5 máquinas `tir`/`tor` vinieron en 0, así que `diffTi = −incTi` y `diffTo = −incTo` |
| Estado | `connected`, `enabled`, `wenabled`, `wbreak`, `wbreakType` | `wenabled=false` = foto deshabilitada a mano (sale del gráfico y se ve atenuada); `wbreak` 🔷 = quiebre de contadores (fila roja) |

Objeto `machine`: `id`, `confId`, `machine` (etiqueta), `numReg` (registro MINCETUR, columna
"RMT"), `den`, `gameName`, `groupId`/`groupName` (modelo), `manufacturerName`,
`manufacturerLogoUrl` (ruta en `cdn.wargos.cloud`), `machineTypeId` (1 en tragamonedas, 2 en la
ruleta), `serverLink`, `machineLink`, `blueSn` (serie del SMIB Pangolin), `zoneId`.

## VI.5 Reglas que se deducen de los datos (✅ comprobadas en las 107 filas de la 129767)

1. **Ecuación de créditos.** `cr` debe cuadrar con `prevCr + incBi + incTi + incCo + incJp −
   incCi − incTo − incCc`. `consistentDiff` es el descuadre acumulado (con signo contrario) desde
   la última foto que cuadró, y `consistent=false` mientras no vuelva a 0. Hubo 5 descuadres
   chicos (20–220 créditos) y todos se cerraron solos en la foto siguiente (🔷 créditos en
   tránsito: la foto cae a mitad de una jugada).
2. **La validación detectó un error real.** `consistentDiff` marcó **−100 000 000 durante 29 h**
   (del 20-10 05:00 al 21-10 09:00). Se resolvió a las 10:00 del 21-10, cuando `ticketOut` saltó
   exactamente +100 000 000 (de 7 194 284 a 107 194 284): era una lectura truncada a 8 dígitos
   que se corrigió. `wbreak` siguió en `false` y `diffTo` registró −100 000 000 en esa hora.
3. **Foto de cierre del día.** Por defecto es la de las **08:00** ("Contadores por hora" también
   abre por defecto en `fechaHora=…T08:00`). Si hubo un retiro de **stacker** antes (06:07–06:10
   los días 22, 23 y 24), esa foto pasa a ser el cierre. El operador puede forzar otra con la
   acción "Cierre".
4. **Fotos por evento además de las horarias.** Los días 22, 23 y 24 tuvieron 25 filas cada uno
   (24 horarias + 1 Stacker).
5. **Incremento negativo** (`incCi < 0` sin `wbreak`) = contadores reseteados. Solo entonces la
   UI ofrece la acción **"Cereo"** para registrar la puesta a cero.
6. **Colores de fila:** rojo = `wbreak`, amarillo = `closeDay`, atenuado = `!wenabled`. El punto de
   la segunda columna indica `connected`.

## VI.6 Acciones que escriben (⚠️ leídas del código, NO ejecutadas)

| Botón o menú ⚙ de la fila | Llamada | Efecto (texto de la UI) |
|---|---|---|
| **Analizar** (cabecera) | `GET /xtgmconex/meter-snap/analize?ini&fin` | "Produccion recalculada". **Es un GET con efecto secundario y no lleva `confId`**: recalcula la producción de toda la sala en el rango |
| Clonar | `POST /xtgmconex/meter-snap/{id}/clone` | "¿Desea clonar esta foto de contadores?" |
| Cierre | `POST /xtgmconex/meter-snap/{id}/use-as-close` | "¿Desea este contador como el del cierre?" |
| Cereo (solo si `incCi<0 && !wbreak`) | `POST /xtgmconex/meter-snap/{id}/add-cereo` | "Cereo agregado" |
| Deshabilitar / Habilitar | `POST /xtgmconex/meter-snap/{id}/disable` · `/enable` | "Registro deshabilitado/habilitado correctamente" |

El mismo objeto de servicios del bundle declara otras llamadas que esta pantalla no dispara:
`GET /deltas/sync`, `/deltas/recalc`, `/deltas/recalc/v2`, `/machines/sync`,
`/machines/{id}/prev`, `/machines/{id}/next` y `/meter-history/sync-hour-cou` (así, truncado,
en el bundle).

Pantalla hermana **Contadores por hora** (`/tgm/accountants`): `GET
/xtgmconex/meter-snap/all-machines?fechaHora=YYYY-MM-DDT08:00&closure=false` (una foto por máquina
a esa hora; el check "Cierre el día" (sic) envía `closure=true`) y exporta con
`.../all-machines/report.xlsx`. Su columna "Origen" usa el mismo mapa de `sourceId`.

## VI.7 URLs base del panel (✅ constante de `main.4a76ac0e.js`)

- **`api4.wargos.cloud`** (BASE): `xpointcore`, `xrgslink`, `xcustomergroup`, `xemulator`.
- **`api5.wargos.cloud`**: `xtgmconex`, `xmachinehistory`, `xsorteos`, `xcustomerapp`, `xnotif`,
  `xprint`, `xfile`, `xuser`, `xcaja`, `xcanje`, `xblacklist`, `xsearch`, `xactivityfeed`,
  `xfoodorder`, `go-salas`, `xstatistic`, `xticket`, `xtables`, `xperson`, `xbilling`, `xbingo`,
  `xsupport`, `xautomation`, `xlogistics`.
- Además `accounts.wargos.cloud` (OAuth) y `cdn.wargos.cloud`.

## VI.8 Implicaciones para el Sistema de Cupones

1. **La foto de contadores es la fuente de verdad** de la producción y, por tanto, de los cupones
   por apuesta: el cupón se calcula sobre `incCi` (lo apostado) por máquina y jornada. Conviene
   una tabla `meter_snapshot` de solo inserción, con acumulados y `prev`/`inc` calculados al
   grabar.
2. **Jornada contable configurable por sala** (Wargos usa 08:00) y foto de cierre explícita
   (`close_day`), con la regla "retiro de stacker antes del corte = cierre".
3. **Validar la ecuación de créditos al grabar** y guardar el descuadre. El caso del ticketOut
   truncado (§VI.5.2) muestra que detecta lecturas malas; un cupón no debería emitirse sobre
   fotos que no cuadran.
4. **Correcciones como operaciones auditadas, no como ediciones** (clonar, marcar cierre, cereo,
   deshabilitar), igual que Wargos, pero siempre con **POST** e idempotencia, nunca con un GET que
   recalcula.
5. **No repetir los dos defectos vistos:** la doble petición por búsqueda y los rangos sin
   paginar (5 días ya pesan 104 KB y tardan 2,7 s por máquina). Para rangos largos conviene servir
   agregados por hora o por día desde el servidor.
6. **Identificador estable de máquina:** separar `confId` (configuración) de la etiqueta visible.
   El caso 147346/47346 muestra que, si no coinciden, la vista se queda sin fotos.

---

### Fuentes
- **Recorrido autenticado (2026-10-24)** de `/tgm/meter-snap` en El Palacio: 5 máquinas al azar
  y un rango de 5 días, con las llamadas XHR interceptadas en el navegador y lectura de los
  bundles `main.4a76ac0e.js` y `693.ded3fcda.chunk.js`. Solo lectura; no se ejecutaron acciones.
- **Recorrido autenticado (2026-08-16)** del panel `app.wargos.cloud` (sesión iniciada por el
  usuario; solo lectura; sin extraer datos personales).
- **Inspección directa (2026-08-16)** de: página de login `accounts.wargos.cloud`, bundles JS
  de `app.wargos.cloud` (`main.1e60f6e8.js`, `asset-manifest.json`),
  `runner.wargos.cloud` (`main-CRGG2SN6.js`), `sorteos.wargos.cloud` (`main.ff5df6a1.js`).
- WGT — sitio oficial: <https://wgt.pe/> y <https://wargos.cloud/>
- Contexto regulatorio: Gob.pe / MINCETUR (Registro de Personas Prohibidas, juego responsable).
- Ley N.º 29733 (Protección de Datos Personales, Perú).
