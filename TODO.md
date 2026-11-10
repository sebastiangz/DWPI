# DWPI — Tablero de avance

**Actualizado:** 2026-11-10 · **Responsable:** S. González · **Operación:** ver [`README.md`](README.md)

Cómo usar este archivo: al terminar algo, se mueve a **Hecho** con su fecha; las decisiones
nuevas van a **Decisiones**; cada semana se agrega una fila a **Métricas**. Como no usamos
git, este archivo es el historial del proyecto.

---

## 🔴 Prioridad inmediata

- [ ] **Verificar los 337 "no existe" del sondeo (60%).** Si el expediente es consecutivo,
      casi todos los números entre 1 y 3.72 millones deberían existir; solo ~17 por encima
      de 3.72 M son inexistentes legítimos. Puede que el extractor esté marcando como
      inexistentes expedientes que sí existen (por ejemplo, si Marcanet tardó en responder).
  - [ ] Buscar a mano en Marcanet 3 números de `no_existe.txt` del rango medio (p. ej. ~2,000,000)
  - [ ] Si existen: corregir la detección en el extractor, vaciar `no_existe.txt` y repetir el sondeo
  - [ ] Si no existen: documentar cómo se numeran los expedientes (huecos, rangos por tipo)

---

## ✅ Hecho

- [x] **11-06** Análisis del reto; propuesta de capas raw / staging / core / mart / crm y orden de trabajo
- [x] **11-06** Diagnóstico de Browse.ai: 80% de corridas fallidas, valores con la etiqueta pegada y domicilio del apoderado asignado al titular → se decide reemplazarlo
- [x] **11-06** `normalizar_browseai.py` (transitorio, para el export existente)
- [x] **11-07** `parser_marcanet.py`: lee la ficha (generales, clases, titular, apoderado, avisos de Gaceta, trámites) en formato Ctrl+S y Playwright
- [x] **11-07** `extractor_marcanet.py` con Playwright: sesión propia, guarda raw, bitácora, capturas de error, valida la lista, se detiene si falla
- [x] **11-07** `reprocesar_raw.py`: regenera staging sin consultar al IMPI
- [x] **11-09** Diagnóstico del servidor (Docker 29, httpd, certbot, red hacia IMPI con IP MX)
- [x] **11-09** `/home2/DWPI` + usuario de servicio `dwpi` sin SSH + permisos por grupo
- [x] **11-09** Docker: PostgreSQL 17 + PostGIS (5433), Metabase LTS (3010), contenedor ETL
- [x] **11-09** Corrección de permisos para el usuario 999 de PostgreSQL
- [x] **11-09** Metabase configurado con usuario de solo lectura `metabase_lector`
- [x] **11-09** `cargar_staging.py` + `pipeline.py` + vista `mart.v_expedientes` (sin datos de contacto, estatus estimado, días para vencer)
- [x] **11-09** Extractor v3: los números inexistentes no cuentan como error ni se reintentan
- [x] **11-09** `generar_lista.py` para rangos y sondeos
- [x] **11-10** Sondeo 1 M – 3.8 M cada 5,000: 561 consultados, 224 encontrados, 337 inexistentes, 0 errores → **226 expedientes en base**

---

## 🔍 Por confirmar (infraestructura)

- [ ] Subdominio `dwpi.agentelegal.org` publicado con HTTPS (paso 6 de INSTALAR.md)
- [ ] Tarea semanal instalada y activa (`systemctl list-timers dwpi-etl.timer`)
- [ ] `docs/` copiado al servidor y una copia en Windows

---

## ⏭️ Sprint 2 — Rebanada vertical: Marcanet → Metabase

**Objetivo:** responder en Metabase *"¿qué marcas vencen en los próximos 12 meses, por estado y clase Niza?"* con datos reales.

- [ ] Con el sondeo, ubicar el rango de números de las marcas que vencen en 2027
- [ ] Decidir cómo bajar ese rango (ritmo actual ~8,000/día): acotar más, o 2-3 extractores en paralelo con pausas
- [ ] Validar con jurídico las reglas de vigencia, periodo de renovación y declaración de uso
- [ ] Modelo `core`: `expediente`, `evento_gaceta`, `titular` (con historial), `apoderado`, `expediente_clase`, `tramite`
- [ ] Cargador `staging → core` (idempotente)
- [ ] Vistas `mart` sobre `core` y la pregunta de vencimientos en Metabase
- [ ] Respaldo diario `pg_dump` a `backups/postgres`

**Hecho cuando:** la pregunta existe en Metabase, con un rango completo cargado y verificado contra 10 expedientes revisados a mano.

---

## 📋 Backlog

**Datos**
- [ ] Sprint 3: extractor SIGA (eventos semanales de la Gaceta) para saber qué expedientes cambiaron
- [ ] Unificar titulares escritos de distintas formas (`pg_trgm`, `unaccent`)
- [ ] Búsqueda "De Signos por Mes de Recepción" como fuente de listas
- [ ] Carga histórica (datos abiertos hasta 2017)
- [ ] Normalizar población → estado/municipio (y geografía con PostGIS)

**Negocio**
- [ ] Política de datos personales y aviso de privacidad antes de usar contactos
- [ ] Lead scoring (Fase 2)
- [ ] CRM: prospectos, seguimiento, embudo (Fase 3)
- [ ] Automatización de correo / WhatsApp (Fase 4)
- [ ] Agentes de IA (Fase 5)

**Servidor (fuera del DWPI, detectado en el diagnóstico)**
- [ ] PostgreSQL nativo (5432) y MariaDB (3306) abiertos a internet en el firewall
- [ ] Certificado vencido en `clothing4u.biz` (vhost activo) y ~11 archivos de renovación de certbot con error
- [ ] Módulos de httpd cargados por duplicado (solo avisos)

---

## 🧭 Decisiones

| Fecha | Decisión | Por qué |
|---|---|---|
| 11-06 | Capas raw / staging / core / mart / crm separadas | Trazabilidad; el CRM no se mezcla con datos de fuentes |
| 11-06 | El **evento** (publicación en Gaceta) es eje del modelo | Renovación, abandono, cambio de titular y oposición son eventos |
| 11-07 | Playwright en lugar de Browse.ai | Control total, guarda la página original, gratis, open source |
| 11-07 | Parser sobre el texto de la página, no sobre ids de JSF | Los ids de PrimeFaces cambian entre sesiones |
| 11-09 | Todo en el servidor Fedora; Windows solo para desarrollar | Corridas programadas, servicios 24/7 |
| 11-09 | SCP en lugar de git | Preferencia del responsable |
| 11-09 | PostgreSQL propio en contenedor (5433), no el nativo | Aislamiento del resto de servicios del servidor |
| 11-09 | Usuario `dwpi` sin SSH | Cuenta de servicio; menor superficie de ataque |
| 11-09 | ETL semanal, domingo 03:30; manual cuando se necesite | Balance entre frescura y carga al IMPI |
| 11-09 | Sin datos de contacto en `mart` | Protección de datos personales |
| 11-10 | Sprint 2 con Marcanet; SIGA pasa al Sprint 3 | Marcanet ya trae vigencia y titular; SIGA aporta actualización semanal |

---

## ⚠️ Riesgos

| Riesgo | Impacto | Mitigación |
|---|---|---|
| Ritmo de ~350 exp/hora | Rangos grandes tardan semanas | Sondeo para acotar; paralelismo moderado; SIGA para lo incremental |
| Marcanet cambia su página | El parser deja de funcionar | Se guarda raw y se reprocesa; alertas por 5 errores seguidos |
| Detección errónea de "no existe" | Expedientes perdidos sin aviso | Verificación en curso (prioridad inmediata) |
| Uso comercial de datos personales | Riesgo legal | Política antes de la Fase 3; datos de contacto fuera de mart |
| Términos de uso / carga al IMPI | Bloqueo de la IP | Pausas de 4–9 s; una sola sesión; horario nocturno |

---

## 📈 Métricas semanales

| Semana | Expedientes en base | Consultados | No existen | Errores | Corridas | Nota |
|---|---|---|---|---|---|---|
| 2026-W45 | 2 | 2 | 0 | 0 | 1 | Prueba inicial en servidor |
| 2026-W46 | 226 | 563 | 337 | 0 | 2 | Sondeo; "no existe" por verificar |
