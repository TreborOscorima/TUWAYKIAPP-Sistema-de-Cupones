# Plan Técnico — Sistema de Cupones

> **Objetivo:** construir un sistema de fidelización para salas de juego (slots/casino)
> **equivalente o superior** al de Wargos/WGT, cubriendo la **suite completa**: Cupones, Puntos,
> Club, Promociones, Sorteos y Canjes. Multi-operador, multi-sala, escalable y auditable.
>
> **Base de evidencia:** [`investigacion-wargos.md`](investigacion-wargos.md) (recorrido en vivo,
> verificado, de los 3 portales). Este plan define **contratos y modelo propios** — no copia los
> internos de Wargos.
>
> **Fecha:** 2026-08-16 · **Estado:** v1 (para revisión)

---

## 0. Índice

1. [Principios de diseño](#1-principios-de-diseño)
2. [Stack recomendado (justificado)](#2-stack-recomendado-justificado)
3. [Arquitectura de alto nivel](#3-arquitectura-de-alto-nivel)
4. [Bounded contexts (módulos del dominio)](#4-bounded-contexts-módulos-del-dominio)
5. [Modelo de datos](#5-modelo-de-datos)
6. [Máquinas de estado](#6-máquinas-de-estado)
7. [Contratos de API (propios)](#7-contratos-de-api-propios)
8. [Flujos clave (secuencias)](#8-flujos-clave-secuencias)
9. [Requisitos no funcionales](#9-requisitos-no-funcionales)
10. [Cumplimiento regulatorio (Perú)](#10-cumplimiento-regulatorio-perú)
11. [Backlog priorizado por fases](#11-backlog-priorizado-por-fases)
12. [Riesgos y decisiones abiertas](#12-riesgos-y-decisiones-abiertas)

---

## 1. Principios de diseño

Aprendidos de lo observado en Wargos + buenas prácticas:

1. **Multi-tenancy por sala desde el día 1.** Todo dato y endpoint se filtra por `salaId` (y por
   `operadorId` arriba). Wargos es multisucursal real; nosotros igual.
2. **El cupón NO es un código: es una plantilla + instancias.** Separar `CouponType` (config) de
   `Coupon` (instancia emitida).
3. **Impresión desacoplada y asíncrona.** La caja **encola** un `PrintJob`; un agente lo imprime y
   reporta. Nunca bloquear al cajero por hardware.
4. **Motor de reglas reutilizable.** Un mismo `PromoRule` (multiplicador × horario × máquinas)
   aplica a cupones **y** a puntos.
5. **Auditoría con diff nativa.** Cada cambio de configuración registra `antes → después`, autor y
   hora (append-only).
6. **RBAC por módulo y por sala.** Permisos finos; un usuario puede ver caja en una sala y no en
   otra.
7. **KPI-first.** La métrica que justifica todo es **"cobertura apostado"** (juego identificado /
   juego total). Se instrumenta desde el inicio (vínculo sesión-de-máquina ↔ cliente).
8. **Empezar como monolito modular, con costuras para extraer servicios.** Evitar el "zoo" de
   microservicios de Wargos hasta que la escala lo pida.

---

## 2. Stack recomendado (justificado)

| Capa | Recomendación | Por qué |
|---|---|---|
| **Lenguaje/Backend** | **Node.js + TypeScript con NestJS** | Tipado fuerte, DI y módulos alineados a bounded contexts; ecosistema maduro; un solo lenguaje back/front reduce fricción del equipo. |
| **Base de datos primaria** | **PostgreSQL 16** (con **JSONB** para config flexible) | Un solo motor transaccional y ACID. **Mejora sobre Wargos** (que usa Mongo+SQL): JSONB da la flexibilidad documental (config de cupón, reglas de promo) *sin* operar dos bases. FTS nativo para búsqueda inicial. |
| **Cache / contadores / locks** | **Redis** | Contadores de límites de cupón (`couponLimit`, `perDay`) atómicos; rate-limiting; sesiones; locks de correlativo. |
| **Cola de trabajos** | **BullMQ (sobre Redis)** | Cola de `PrintJob`, notificaciones, cálculos de puntos y cierre de sorteos, con reintentos y backoff. |
| **Búsqueda de clientes** | **Postgres FTS** → **Meilisearch/OpenSearch** al escalar | Wargos usa ElasticSearch; empezamos simple y migramos el índice cuando el volumen lo exija. |
| **Tiempo real** | **WebSocket (Socket.IO)** | Estado de máquinas/sesiones en vivo, estado de `PrintJob`, tablero operativo, animación de sorteos. |
| **Frontend Admin** | **React + TypeScript + Vite** | Panel de configuración (equivalente a `app.wargos.cloud`). |
| **Frontend Operativo (caja/POS)** | **React + Vite**, PWA offline-tolerante | Flujo rápido de emisión/canje; equivalente al runner. |
| **App de proyección de sorteos** | **React + three.js / react-three-fiber** | Tómbola 3D en pantalla grande (como `sorteos.wargos.cloud`). |
| **Agente de impresión** | **Servicio local en Node o Go** hablando **ESC/POS** | Corre en cada sala; hace *polling*/suscripción a la cola de `PrintJob` y maneja impresoras térmicas por zona + respaldo. |
| **Auth / Identidad** | **Keycloak** (OIDC/OAuth2, multi-tenant) | IdP central como Wargos (`accounts.wargos.cloud`), un `client_id` por app, roles y realms; evita construir auth desde cero. |
| **Conectividad de máquinas (SAS)** | Servicio conector dedicado (fuera del núcleo) | Equivalente a `xtgmconex`; lee medidores/sesiones vía SAS/protocolo del proveedor. Se integra por eventos. |
| **Infra** | **Docker + Kubernetes** (o ECS), **CDN** para estáticos | Escalado horizontal por servicio cuando se extraiga del monolito. |
| **Observabilidad** | **OpenTelemetry + Grafana/Loki/Tempo** (o Datadog) | Trazas, métricas y logs; Wargos usa Datadog. |

> **Alternativas válidas** si el equipo lo prefiere: **.NET 8 + EF Core + Postgres**, o
> **Java/Spring Boot + Postgres**. La arquitectura y el modelo de datos de este plan son
> agnósticos del lenguaje.

---

## 3. Arquitectura de alto nivel

```
                        ┌──────────────────────────────────────┐
                        │        Identidad (Keycloak/OIDC)      │
                        │   realms, clients por app, RBAC       │
                        └───────────────┬──────────────────────┘
                                        │ OAuth2 / JWT
   ┌───────────┐   ┌───────────┐   ┌────┴─────┐   ┌──────────────┐
   │  Admin    │   │ Operativo │   │  Sorteos │   │  App Cliente │  (frontends)
   │  (React)  │   │ (POS PWA) │   │ (3D R3F) │   │   (móvil)    │
   └─────┬─────┘   └─────┬─────┘   └────┬─────┘   └──────┬───────┘
         │               │              │                │
         └───────────────┴───────┬──────┴────────────────┘
                                  │ REST + WebSocket (API Gateway)
                    ┌─────────────┴───────────────────────────────┐
                    │        BACKEND (monolito modular NestJS)     │
                    │  ┌────────┬─────────┬──────────┬───────────┐ │
                    │  │Identity│Customer │ Loyalty   │Promotions │ │
                    │  │& Access│& Ludopat│(Cupón/Pto/│(Multiplic/│ │
                    │  │        │         │  Club)    │ Cashback) │ │
                    │  ├────────┼─────────┼──────────┼───────────┤ │
                    │  │Raffles │Redemption│ Printing │ Analytics │ │
                    │  │Sorteos │ Canjes   │(Jobs)    │ (KPIs)    │ │
                    │  └────────┴─────────┴──────────┴───────────┘ │
                    └───┬───────────┬───────────┬──────────────────┘
                        │           │           │
                 ┌──────┴──┐  ┌─────┴────┐  ┌───┴────────┐   ┌───────────────┐
                 │Postgres │  │  Redis   │  │ BullMQ cola│   │ Conector SAS  │
                 │ (JSONB) │  │cache/lock│  │(PrintJob…) │   │ (xmáquinas)   │
                 └─────────┘  └──────────┘  └─────┬──────┘   └──────┬────────┘
                                                  │                 │ eventos de juego
                                          ┌───────┴──────┐          │ (coin-in, sesión)
                                          │ Agente de    │◄─────────┘
                                          │ impresión    │ ESC/POS → impresoras por zona
                                          │ (por sala)   │
                                          └──────────────┘
```

**Por qué monolito modular:** cada bounded context es un módulo NestJS con su propio esquema
lógico y API interna. Cuando un módulo necesite escalar aparte (p. ej. Printing o Analytics), se
extrae a servicio sin reescribir el dominio. Evitamos la complejidad operativa de ~30
microservicios que tiene Wargos hasta tener la escala que lo justifique.

---

## 4. Bounded contexts (módulos del dominio)

| Contexto | Responsabilidad | Equivalente Wargos |
|---|---|---|
| **Identity & Access** | Auth OIDC, usuarios/empleados, roles, permisos por módulo+sala | `xuser` + accounts |
| **Customer & Ludopatía** | CRM de clientes, alta, búsqueda, ingreso a sala, blacklist/autoexclusión | `xpointcore/customers` + `xblacklist` |
| **Loyalty Core** | **Cupones** (tipos + instancias), **Puntos** (reglas + saldo), **Club** (tiers) | `xpointcore` |
| **Promotions** | Multiplicadores, misteriosos/pozos, recompensas, cashback | `xpointcore` promos + `xautomation` |
| **Redemption (Canjes)** | Catálogo de productos, canje de puntos/cupones, reportes | `xcanje` + catalog-products |
| **Printing** | Plantillas, impresoras por zona, cola de `PrintJob`, agente | `xprint` |
| **Raffles (Sorteos)** | Sorteos, participantes, ganadores, acta, proyección 3D | `xsorteos` |
| **Machine/SAS** (integración) | Sesiones de juego, medidores, coin-in, vínculo cliente↔máquina | `xtgmconex` + `xmachinehistory` |
| **Analytics** | KPIs (cobertura apostado, producción), reportes | `xstatistic` |
| **Notifications** (soporte) | Push/email/SMS, campañas por evento | `xnotif` |

---

## 5. Modelo de datos

Notación: `PK` clave primaria, `FK` foránea, `[]` colección/relación N. Todo con `salaId` salvo
catálogos globales. Timestamps `createdAt/updatedAt` implícitos en todas.

### 5.1 Tenencia e identidad

```
Operador        { id PK, nombre, ruc, config JSONB }
Sala            { id PK, operadorId FK, nombre, direccion, timezone, config JSONB }
Usuario         { id PK, operadorId FK, email, nombre, estado, authSubject }   # empleado
Rol             { id PK, operadorId FK, nombre }
Permiso         { id PK, codigo }                 # ej. CUPON_CONFIG, CAJA_EMITIR, ADMIN_ROLES
RolPermiso      { rolId FK, permisoId FK }
UsuarioRolSala  { usuarioId FK, rolId FK, salaId FK }   # RBAC por módulo Y por sala
```

### 5.2 Cliente y ludopatía

```
Cliente         { id PK, operadorId FK, tipoDoc, doc UNIQUE(operador,doc), nombres,
                  apPaterno, apMaterno, alias, fechaNac, genero(N|F|M), estadoCivil,
                  lugar, direccion, celular, email, preferencias, estado }
ClienteIngreso  { id PK, salaId FK, clienteId FK, fechaHora, canal(SCAN|MANUAL), documentoLeido }
Blacklist       { id PK, operadorId FK, tipoDoc, doc, motivo(LUDOPATIA|JUDICIAL|AUTOEXCL),
                  vigenteDesde, vigenteHasta, fuente }
```

> **Regla:** en `ClienteIngreso` y al emitir cualquier beneficio se valida contra `Blacklist`
> (juego responsable / Registro de Personas Prohibidas). Ver §10.

### 5.3 Cupones (el núcleo)

```
CouponType      { id PK, salaId FK, enabled, nombre, abrv,
                  correlativoEnabled, correlativoNext,
                  iniAt (fecha+hora), finAt (fecha+hora),
                  genero(N|F|M), limitePeriodo, limitePorDia,
                  montoMin, montoMax, maxPorJugada,
                  virtual, autoDeposit, printable,
                  templateId FK, autoPrint, printerDefaultId FK, autoPrintByRedeem,
                  showAlias, moduleShow, modulePrint,
                  expirationPolicyId FK, config JSONB }
CouponTypeMachine   { couponTypeId FK, machineId FK }         # elegibilidad por máquina
CouponTypeZonePrinter { couponTypeId FK, zona, printerId FK } # impresora por zona
ExpirationPolicy    { id PK, salaId FK, tipo(FECHA_FIJA|DIAS_DESDE_EMISION|FIN_DE_DIA), valor }

Coupon          { id PK, salaId FK, couponTypeId FK, clienteId FK,
                  codigo/folio UNIQUE, correlativo,
                  machineId FK NULL, sessionId FK NULL,   # nace opc. atado a máquina/sesión
                  monto, estado (ver §6.1),
                  promoId FK NULL, printJobId FK NULL,
                  emitidoAt, emitidoPor FK, expiraAt,
                  canjeadoAt, canjeadoPor FK, anuladoAt, anuladoMotivo }
```

> `Coupon.estado` es la máquina de estados de §6.1. Los reportes de Wargos (otorgados /
> acumulados / generados) se derivan de este estado, no son tablas separadas.

### 5.4 Impresión

```
PrintTemplate   { id PK, codigo(M1..M5,L3..L7,E1,D1,D3..D5,...), familia(M|L|E|D),
                  ancho, layout JSONB, version }        # catálogo cerrado, versionado
Printer         { id PK, salaId FK, nombre, zona, tipoCupon, esRespaldo,
                  conexion(IP|USB), estado(ONLINE|OFFLINE), ubicacion }
PrintJob        { id PK, salaId FK, couponId FK NULL, clienteId FK,
                  printerId FK, cantidad, estado (ver §6.2), mensaje,
                  intentos, createdAt, enviadoAt, completadoAt }
```

### 5.5 Puntos y Club

```
PointRule       { id PK, salaId FK, grupoMaquinaId FK, soles PorPunto, tope, modo, enabled }
PointAccount    { id PK, clienteId FK, salaId FK, saldo, saldoVigente }
PointLedger     { id PK, pointAccountId FK, tipo(ACUMULA|CANJE|AJUSTE|EXPIRA),
                  puntos, refTipo, refId, fecha, expiraAt }
Club            { id PK, salaId FK, nombre, valorCanjePuntoSol }
ClubTier        { id PK, clubId FK, nombre, orden, umbralSubida, umbralBajada,
                  beneficios JSONB }
ClubMembership  { id PK, clienteId FK, clubId FK, tierId FK, desde,
                  temporada, ultimaActividadAt }
```

### 5.6 Promociones

```
PromoRule       { id PK, salaId FK, nombre, target(CUPON|PUNTOS), targetRefId,
                  multiplicador, enabled }
PromoWindow     { id PK, promoRuleId FK, dias(bitmask LUN..DOM), horaDesde, horaHasta }
PromoMachine    { promoRuleId FK, machineId FK }         # máquinas participantes
MysteryPool     { id PK, salaId FK, nombre, tipo(PROGRESIVO|MISTERIOSO), pozoActual,
                  config JSONB, maquinasVinculadas [] }
Cashback        { id PK, salaId FK, porcentaje, ventana, condiciones JSONB }
Reward          { id PK, salaId FK, nombre, tipo, config JSONB }
```

### 5.7 Sorteos

```
Sorteo          { id PK, salaId FK, tipo(HOT_SEAT|RULETA|TOMBOLA|TORNEO),
                  nombre, premioTipo(CASH|PRODUCTO), premioMonto, moneda,
                  ventanaDesde, ventanaHasta, fechaHora,
                  promoId FK NULL, estado(PROGRAMADO|EN_CURSO|SORTEADO|ANULADO),
                  actaNumero, seed, ganadorClienteId FK NULL }
SorteoParticipante { sorteoId FK, clienteId FK, participaciones, origen(CUPON|PUNTOS) }
SorteoResultado    { id PK, sorteoId FK, clienteId FK, machineId FK NULL,
                     seedUsada, hashVerificacion, timestamp }
```

> **Mejora sobre Wargos (auditable):** guardamos `seed` + `hashVerificacion` del sorteo →
> resultado **verificable/reproducible**, además del `actaNumero` regulatorio.

### 5.8 Canjes (redención)

```
CatalogProducto { id PK, salaId FK, nombre, costoPuntos, stock, enabled, imagen }
Canje           { id PK, salaId FK, clienteId FK, tipo(PUNTOS_PRODUCTO|CUPON),
                  productoId FK NULL, couponId FK NULL, costoPuntos,
                  estado(SOLICITADO|ENTREGADO|ANULADO), fecha, atendidoPor FK }
```

### 5.9 Máquina / sesión (integración) y auditoría

```
Maquina         { id PK, salaId FK, numero, modelo, zona, grupoId FK, sasLink(bool) }
GrupoMaquina    { id PK, salaId FK, nombre }
MachineSession  { id PK, salaId FK, maquinaId FK, clienteId FK NULL,
                  inicioAt, finAt, coinIn, apostado, ganado }   # base de "cobertura apostado"
EntityModification { id PK, entidad, entidadId, campo, valorAnterior, valorNuevo,
                     usuarioId FK, fechaHora }                  # auditoría con diff (append-only)
```

---

## 6. Máquinas de estado

### 6.1 Cupón (`Coupon.estado`)

```
 ACUMULADO ──(generar instancia)──► GENERADO ──(entregar/imprimir)──► OTORGADO
                                                                        │
                                            ┌───────────────────────────┼───────────────┐
                                            ▼                           ▼               ▼
                                        CANJEADO                    EXPIRADO         ANULADO
```

- **ACUMULADO**: derecho ganado (por juego/puntos) aún no materializado en instancia.
- **GENERADO**: instancia creada (folio asignado), aún no entregada.
- **OTORGADO** (≈ impreso/entregado): en manos del cliente, vigente.
- **CANJEADO**: redimido (validado por código o auto-depósito en máquina).
- **EXPIRADO**: venció `expiraAt` sin canje (job programado).
- **ANULADO**: invalidado manualmente (con motivo + auditoría).

### 6.2 Trabajo de impresión (`PrintJob.estado`)

```
 NUEVO ──(agente toma)──► ENVIADO ──► COMPLETADO
   ▲                         │
   └──────(reintento)◄──── ERROR (mensaje: "Papel agotado", "Offline", …)
```

- Reintentos con backoff (máx configurable). `ERROR` persiste el `mensaje` para diagnóstico.
- Ruteo a `printerDefault` → si `OFFLINE`, cae a la impresora `esRespaldo` de la zona.

### 6.3 Sorteo (`Sorteo.estado`)

```
 PROGRAMADO ──(abrir ventana)──► EN_CURSO ──(asignar ganador)──► SORTEADO
        └────────────────────────(anular)────────────────────► ANULADO
```

- Al pasar a `SORTEADO`: se fija `seed`, se calcula ganador, se genera `hashVerificacion` y
  `actaNumero` correlativo.

---

## 7. Contratos de API (propios)

Diseño REST propio, más limpio que el de Wargos. Prefijo `/api/v1`. Auth: `Bearer JWT` (OIDC).
Todo scopeado por sala vía header `X-Sala-Id` o path. Respuestas con envelope
`{ data, meta, error }`.

### 7.1 Cupones — configuración (Admin)

```
GET    /coupon-types?enabled=&estado=vigente|expirado|proximo
POST   /coupon-types
GET    /coupon-types/{id}
PATCH  /coupon-types/{id}
GET    /coupon-types/{id}/audit                 # modificaciones (diff)
GET    /coupon-types/{id}/zone-printers
PUT    /coupon-types/{id}/zone-printers
GET    /expiration-policies
GET    /print-templates                          # catálogo cerrado
```

### 7.2 Cupones — operación (Caja/POS)

```
GET    /coupon-types/grouped                     # tipos vigentes para la caja
POST   /coupons/issue                            # EMISIÓN
        body: { clienteId, items:[{couponTypeId, cantidad}],
                machineId?, printerZone, promoId? }
        → crea Coupons (GENERADO→OTORGADO) + encola PrintJob(s)
POST   /coupons/validate                         # validación por código (lectura de estado)
        body: { codigo } → { estado, couponType, cliente, expiraAt }
POST   /coupons/{id}/redeem                       # canje (→ CANJEADO)
POST   /coupons/{id}/void                         # anular (→ ANULADO, con motivo)
GET    /coupons?estado=&clienteId=&couponTypeId=&desde=&hasta=
```

### 7.3 Impresión

```
GET    /print-jobs?estado=NUEVO|ENVIADO|COMPLETADO|ERROR&clienteId=&couponTypeId=
POST   /print-jobs/{id}/retry
GET    /printers?estado=                          # estado de impresoras por sala
# Agente (canal dedicado):
WS     /agent/print                               # suscripción a jobs NUEVO de su sala
POST   /agent/print-jobs/{id}/ack                 # ENVIADO
POST   /agent/print-jobs/{id}/result              # COMPLETADO | ERROR + mensaje
```

### 7.4 Clientes

```
GET    /customers/search?q=                       # FTS por doc/nombre
POST   /customers
GET    /customers/{id}
PATCH  /customers/{id}
POST   /customers/{id}/ingress                    # registro de ingreso (scan/manual)
GET    /customers/analytics/genders
GET    /customers/analytics/ages?desde=&hasta=
GET    /blacklist/check?tipoDoc=&doc=             # validación de prohibidos
```

### 7.5 Puntos y Club

```
GET    /point-rules
POST   /point-rules
GET    /customers/{id}/points                      # saldo + ledger
POST   /points/adjust                              # ajuste manual (auditado)
GET    /clubs/{id}/tiers
GET    /customers/{id}/membership
POST   /clubs/simulate                             # simulación de tiers
```

### 7.6 Promociones

```
GET    /promo-rules?target=CUPON|PUNTOS
POST   /promo-rules
PATCH  /promo-rules/{id}
GET    /mystery-pools                               # pozos progresivos
GET    /cashback-config
GET    /rewards
```

### 7.7 Sorteos

```
GET    /sorteos?desde=
POST   /sorteos
POST   /sorteos/{id}/participants                   # alta por cupón/puntos
POST   /sorteos/{id}/draw                           # asignar ganador (fija seed+hash+acta)
GET    /sorteos/{id}/acta                           # acta regulatoria (PDF)
GET    /sorteos/{id}/verify?seed=                   # verificación reproducible del resultado
# Proyección (app 3D):
GET    /sorteos/{id}/participants
GET    /sorteos/{id}/animation
GET    /sorteos/{id}/winner-view
```

### 7.8 Canjes y Analytics

```
GET    /catalog-products?enabled=true
POST   /redemptions                                 # canje puntos→producto
GET    /redemptions?desde=&hasta=
GET    /analytics/coverage                          # "cobertura apostado" (KPI)
GET    /analytics/production?desde=&hasta=
```

---

## 8. Flujos clave (secuencias)

### 8.1 Emisión de cupón en caja (con impresión asíncrona)

```
Cajero → POS: busca cliente (GET /customers/search)
POS → API:   POST /coupons/issue { clienteId, items, machineId?, printerZone }
API:         valida límites (Redis: couponLimit/perDay atómico), blacklist, vigencia
API:         genera Coupon(s) [GENERADO→OTORGADO], asigna folio correlativo (lock Redis)
API:         encola PrintJob(s) [NUEVO]  → responde 201 al POS (NO espera impresión)
Agente ⇄ API (WS): recibe job → imprime ESC/POS → POST result [COMPLETADO|ERROR]
POS (WS):    ve estado del PrintJob en vivo; si ERROR "papel agotado" → reintento/otra zona
```

### 8.2 Canje / validación de cupón

```
Cajero ingresa código → POST /coupons/validate → muestra estado (vigente/usado/expirado)
Si vigente y aplica → POST /coupons/{id}/redeem → CANJEADO (+ auto-depósito a máquina si aplica)
```

### 8.3 Acumulación de puntos (evento de juego)

```
Conector SAS → evento: MachineSession { clienteId, apostado }
Loyalty:      aplica PointRule (soles→puntos) × PromoRule activo (multiplicador, ventana, máquina)
              → PointLedger(ACUMULA); recalcula ClubTier (subida/bajada)
```

### 8.4 Sorteo

```
Admin crea Sorteo [PROGRAMADO]; participaciones entran por cupón/puntos [EN_CURSO]
Operador → POST /sorteos/{id}/draw → fija seed, calcula ganador, hash, actaNumero [SORTEADO]
App 3D proyecta animation/winner-view; Acta PDF disponible; verify reproducible
```

---

## 9. Requisitos no funcionales

- **Escalabilidad:** stateless en API (JWT), escalado horizontal; Redis para estado compartido;
  particionado temporal de tablas de alto volumen (`Coupon`, `PrintJob`, `PointLedger`,
  `MachineSession`) por mes/sala.
- **Concurrencia de folios:** correlativo por `CouponType` con lock (Redis `INCR` o secuencia
  Postgres) para evitar duplicados.
- **Idempotencia:** `POST /coupons/issue` acepta `Idempotency-Key` (evita doble emisión por
  reintento de red en caja).
- **Offline-tolerante (POS):** cola local si cae la red; sincroniza al reconectar.
- **Auditoría:** `EntityModification` append-only en toda config; retención según norma.
- **Seguridad:** OIDC, RBAC módulo+sala, cifrado en tránsito (TLS) y en reposo; PII minimizada;
  rate-limiting.
- **Observabilidad:** trazas OpenTelemetry por request; métricas de negocio (emisiones/hora,
  jobs en ERROR, cobertura apostado) en dashboards.
- **Disponibilidad:** la impresión y la caja no dependen de servicios no críticos (degradación
  elegante).

---

## 10. Cumplimiento regulatorio (Perú)

| Requisito | Implementación |
|---|---|
| **Juego responsable / Registro de Personas Prohibidas (MINCETUR)** | Validación contra `Blacklist` en ingreso a sala y antes de otorgar beneficios; escaneo de documento en `ClienteIngreso`. |
| **Acta de sorteo** | `Sorteo.actaNumero` correlativo + PDF + `seed/hash` verificable. |
| **Ley N.º 29733 (Protección de Datos)** | Consentimiento, minimización de PII, derecho de acceso/rectificación, cifrado, control de acceso por rol. |
| **SUNAT / tributario** | Reportes de premios y canjes exportables; trazabilidad de montos. |
| **Ludopatía** | Autoexclusión (`Blacklist` motivo AUTOEXCL) + límites configurables. |

---

## 11. Backlog priorizado por fases

### Fase 0 — Fundaciones (2–3 sprints)
- [ ] Tenencia (Operador/Sala) + multi-sala en toda capa.
- [ ] Identidad OIDC (Keycloak), RBAC por módulo+sala.
- [ ] Cliente: alta, búsqueda (FTS), ingreso a sala, Blacklist + validación.
- [ ] Auditoría (`EntityModification`) transversal.
- [ ] Observabilidad base + CI/CD + Docker.

### Fase 1 — MVP Cupones (3–4 sprints) 🎯 *el corazón*
- [ ] `CouponType` (config completa) + `ExpirationPolicy` + plantillas (catálogo).
- [ ] Emisión (`/coupons/issue`) con límites atómicos + folio correlativo + idempotencia.
- [ ] **Printing**: `PrintJob`, `Printer` por zona + respaldo, **agente ESC/POS**, estados.
- [ ] Validación y canje (`validate`/`redeem`/`void`) + máquina de estados.
- [ ] POS operativo (PWA) + Admin de cupones. Job de expiración.
- **Entregable:** emitir, imprimir y canjear cupones en producción, multi-sala.

### Fase 2 — Puntos y Club (2–3 sprints)
- [ ] `PointRule` (soles→puntos), `PointLedger`, saldo, ajustes auditados.
- [ ] `Club`/`ClubTier` dinámicos (subida/bajada, temporadas, expiración por inactividad, valor
      canje) + simulación.
- [ ] Integración con eventos de `MachineSession` (acumulación).

### Fase 3 — Promociones y Canjes (2–3 sprints)
- [ ] `PromoRule` (multiplicador × ventanas × máquinas) reutilizable cupón/puntos.
- [ ] `MysteryPool`/progresivos, `Cashback`, `Reward`.
- [ ] Catálogo de canje (`CatalogProducto`) + `Canje` (puntos→producto).

### Fase 4 — Sorteos (2–3 sprints)
- [ ] `Sorteo` (HOT_SEAT/ruleta/tómbola/torneo), participantes por cupón/puntos.
- [ ] Draw con `seed`+`hash` verificable + `Acta` PDF.
- [ ] App de proyección 3D (react-three-fiber).

### Fase 5 — Analítica y experiencia (continuo)
- [ ] KPI **cobertura apostado**, producción, dashboards en vivo (WebSocket).
- [ ] Notificaciones/campañas por evento.
- [ ] App cliente (saldo, cupones, sorteos) — opcional.

### Transversal (todas las fases)
- [ ] Conector SAS/máquinas (equipo/integración aparte).
- [ ] Hardening de seguridad y cumplimiento; pruebas de carga.

---

## 12. Riesgos y decisiones abiertas

| Tema | Decisión a tomar |
|---|---|
| **Integración SAS/máquinas** | ¿Qué protocolo/proveedor? Define el conector (coin-in, sesión, auto-depósito de cupón). Es el mayor riesgo técnico externo. |
| **Impresoras térmicas** | Modelos/marcas objetivo → set de plantillas ESC/POS y anchos (58/80mm). |
| **Mono vs. micro** | Confirmar: empezamos monolito modular (recomendado) y extraemos Printing/Analytics al escalar. |
| **Postgres solo vs. + Mongo** | Recomiendo **solo Postgres+JSONB**; revisar si algún dominio pide documentos puros. |
| **App cliente móvil** | ¿Entra en alcance ahora o fase posterior? |
| **Migración de datos** | ¿Hay que migrar clientes/puntos/cupones existentes desde otro sistema? |

---

### Referencias
- Investigación de base verificada: [`investigacion-wargos.md`](investigacion-wargos.md)
  (Partes II–V: admin, runner, sorteos).
- Contexto regulatorio: MINCETUR (juego responsable / personas prohibidas), Ley N.º 29733.
