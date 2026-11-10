# DWPI — Data Warehouse de Propiedad Industrial

Plataforma para reunir la información pública de marcas del IMPI, guardarla con
historial y convertirla en oportunidades comerciales: renovaciones, vigilancia,
oposiciones, cambios de titular y licenciamiento.

- **Servidor:** Fedora Server 44, `/home2/DWPI`
- **Tablero:** https://dwpi.agentelegal.org (Metabase)
- **Avance y pendientes:** ver [`TODO.md`](TODO.md)

---

## 1. Arquitectura

```
 FUENTES                 CAPAS DE DATOS                              CONSUMO
┌───────────┐   ┌──────────────┐  ┌───────────────┐  ┌──────┐  ┌──────┐  ┌──────────┐
│ Marcanet  │──▶│ RAW          │─▶│ STAGING       │─▶│ CORE │─▶│ MART │─▶│ Metabase │
│ SIGA*     │   │ página tal   │  │ JSON por      │  │ *    │  │vistas│  │ (web)    │
└───────────┘   │ cual (.txt/  │  │ expediente,   │  └──────┘  └──────┘  └──────────┘
                │ .html)       │  │ con historial │                         ▲
                └──────────────┘  └───────────────┘              ┌──────┐   │
                  data/raw/         PostgreSQL                   │ CRM* │───┘
                                                                 └──────┘
                                                        * = pendiente
```

| Capa | Dónde | Qué guarda | Regla |
|---|---|---|---|
| **raw** | `data/raw/marcanet/<exp>.txt/.html` | La página tal como se descargó | Nunca se modifica. Si el parser mejora, se reprocesa desde aquí sin volver a consultar al IMPI |
| **staging** | `staging.marcanet_detalle` | Expediente completo en JSON, una fila por **versión** | Incluye datos personales; solo la usa el ETL |
| **core** | esquema `core` | Modelo normalizado: expediente, eventos, titular, apoderado, clases | *Pendiente (Sprint 2)* |
| **mart** | `mart.v_expedientes` | Una fila por expediente, en columnas, **sin datos de contacto** | Es lo que ve Metabase |
| **crm** | esquema `crm` | Leads, contactos, seguimiento | *Pendiente (Fase 3)* |
| **etl** | esquema `etl` + `data/control/` | Bitácora, errores, números inexistentes | Trazabilidad |

---

## 2. Dónde está cada cosa

```
/home2/DWPI/
├── app/
│   ├── etl/
│   │   ├── extractor_marcanet.py   Navega Marcanet (Playwright) y guarda raw
│   │   ├── parser_marcanet.py      Convierte la página en datos estructurados
│   │   ├── cargar_staging.py       Carga a PostgreSQL y aplica migraciones
│   │   ├── pipeline.py             Corrida completa: extraer + cargar
│   │   ├── generar_lista.py        Genera rangos de números de expediente
│   │   └── reprocesar_raw.py       Regenera staging desde raw (sin internet)
│   └── database/
│       ├── init/                   Se ejecuta UNA vez al crear la base
│       └── ddl/                    Migraciones; se aplican en cada carga
├── docker/
│   ├── compose.yaml                Servicios db, metabase, etl
│   ├── .env                        UID/GID de dwpi
│   ├── etl/Dockerfile              Imagen Python + Chromium
│   ├── httpd/                      VirtualHost de dwpi.agentelegal.org
│   └── systemd/                    Tarea semanal (dwpi-etl.service/.timer)
├── data/
│   ├── listas/expedientes.txt      Números a procesar en la próxima corrida
│   ├── raw/marcanet/               Páginas descargadas
│   ├── staging/                    Salida del parser (.jsonl)
│   └── control/                    bitacora.jsonl, no_existe.txt, capturas de error
├── volumes/postgres/               Archivos de la base (dueño: uid 999)
├── backups/postgres/               Respaldos
├── logs/etl.log                    Salida de cada corrida
├── secrets/dwpi.env                Contraseñas (no compartir)
└── docs/                           Este README y TODO.md
```

---

## 3. Servicios y accesos

| Servicio | Contenedor | Acceso | Notas |
|---|---|---|---|
| PostgreSQL 17 + PostGIS | `dwpi-db` | `127.0.0.1:5433` | El PostgreSQL nativo del servidor (5432) es otro y no se toca |
| Metabase LTS | `dwpi-metabase` | `127.0.0.1:3010` → https://dwpi.agentelegal.org | Se conecta con el usuario de solo lectura `metabase_lector` |
| ETL | `dwpi-etl` | — | No queda encendido; se ejecuta por corrida |

**Usuarios**
- `dwpi`: cuenta de servicio, dueña de `/home2/DWPI`. **Sin SSH ni contraseña.**
- `sgonzalez`: administra; entra por SSH (puerto 2222) y copia archivos por SCP. Pertenece al grupo `dwpi`.
- En PostgreSQL: `dwpi` (dueño, lo usa el ETL), `metabase_lector` (solo lectura), `metabase` (base interna de Metabase).

---

## 4. Operación

Todos los comandos desde `/home2/DWPI` salvo que se indique otra carpeta.

| Quiero… | Comando |
|---|---|
| Levantar base y Metabase | `cd docker && docker compose up -d` |
| Ver estado | `cd docker && docker compose ps` |
| Armar lista por rango | `python3 app/etl/generar_lista.py 1700000 1700500 > data/listas/expedientes.txt` |
| Armar lista de sondeo | `python3 app/etl/generar_lista.py 1000000 3800000 --paso 5000 > data/listas/expedientes.txt` |
| Correr el ETL ahora (segundo plano) | `sudo systemctl start dwpi-etl.service` |
| Ver avance | `tail -f logs/etl.log` |
| Detener la corrida | `sudo systemctl stop dwpi-etl.service` |
| Solo recargar a PostgreSQL | `cd docker && docker compose run --rm etl python /app/pipeline.py --solo-carga` |
| Reprocesar raw tras mejorar el parser | `cd docker && docker compose run --rm etl python /app/reprocesar_raw.py` y después `--solo-carga` |
| Ver próxima corrida automática | `systemctl list-timers dwpi-etl.timer` |
| Entrar a la base | `docker exec -it dwpi-db psql -U dwpi -d dwpi` |

**Corrida automática:** cada domingo a las 03:30 procesa `data/listas/expedientes.txt`.

**Reglas que conviene recordar**
- Una corrida se puede interrumpir y repetir: salta lo ya descargado y lo marcado en `no_existe.txt`.
- Para volver a consultar un número marcado como inexistente, bórralo de `data/control/no_existe.txt`.
- Ritmo: ~350 expedientes por hora, con pausas para no saturar al IMPI. **No reducir las pausas.**
- Después de copiar código nuevo por SCP no hace falta reconstruir la imagen; solo si cambia `docker/etl/Dockerfile` (`docker compose build etl`).
- **Nunca** `docker compose down -v`: borraría datos.

---

## 5. Problemas conocidos y solución

| Síntoma | Causa | Solución |
|---|---|---|
| `dwpi-db` *unhealthy*; log: `cannot open directory '/docker-entrypoint-initdb.d/'` | PostgreSQL (uid 999) no puede leer los scripts | `sudo setfacl -R -m u:999:rX -m d:u:999:rX app/database/init` |
| Errores en la bitácora con texto en lugar de números | Se pasó un archivo que no es lista de expedientes | La lista debe tener un número por línea; el extractor v2+ lo valida |
| `TargetClosedError` repetido | Se cerró el navegador durante la corrida | El extractor v2+ se detiene solo; volver a correr |
| Se detiene con "5 errores seguidos" | Marcanet caído o cambió su página | Revisar capturas en `data/control/error_*.png` |
| La tabla no aparece en Metabase | Metabase no ha sincronizado | Admin → Bases de datos → DWPI → *Sincronizar ahora* |
| Archivo copiado por SCP no editable por `dwpi` | Faltan permisos de grupo | Verificar con `getfacl`; el grupo `dwpi` debe tener `rwX` |

---

## 6. Estado actual

Actualizar con:
```bash
docker exec dwpi-db psql -U dwpi -d dwpi -c "
  SELECT count(*)                          AS expedientes,
         count(*) FILTER (WHERE estatus_inferido = 'REGISTRADA') AS registradas,
         count(*) FILTER (WHERE dias_para_vencer BETWEEN 0 AND 365) AS vencen_12_meses,
         max(extraido_en)::date            AS ultima_extraccion
  FROM mart.v_expedientes;"
wc -l data/control/no_existe.txt
```

| Fecha | Expedientes en base | No existen | Errores | Última extracción |
|---|---|---|---|---|
| 2026-11-10 | 226 | 337 (por verificar) | 0 | 2026-11-10 |
