# drone-monitoring

Monitorización del bridge Webots ↔ Renode de
[CF_renode_emulator](https://github.com/danielalaez/CF_renode_emulator). Guarda en
InfluxDB la telemetría que escribe el bridge y la muestra en un dashboard de
Grafana. No modifica el bridge ni se conecta a Renode: solo lee el fichero JSONL
que el bridge genera con `--telemetry`.

## Guía rápida

Se ejecuta en la máquina donde funciona CF_renode_emulator. Necesita Docker con
`docker compose` y Python 3.10 o superior.

**Una sola vez:**

```bash
git clone https://github.com/JonRecarte/drone-monitoring.git
cd drone-monitoring
docker compose up -d
```

La primera vez descarga unos 1,1 GB de imágenes. Los contenedores quedan
arrancados y se levantan solos cuando arranca Docker.

**En cada vuelo, dos terminales:**

```bash
# Terminal A, dentro de drone-monitoring/: el ingestor sigue el fichero de telemetría
python3 bridge_to_influx.py --file /tmp/cf_bridge.jsonl --follow

# Terminal B, dentro de CF_renode_emulator/: el vuelo de siempre, con TELEMETRY
TELEMETRY=/tmp/cf_bridge.jsonl FAKE=0 ALT=0.5 SECONDS_RUN=45 ./scripts/run_hover_demo.sh
```

La ruta de `TELEMETRY` y la de `--file` tienen que ser la misma: ese fichero es el
punto de unión entre los dos proyectos. El ingestor puede arrancarse antes que el
vuelo; espera a que aparezca el fichero.

**Grafana:** `http://localhost:3000`, o `http://<ip-de-la-máquina>:3000` desde otro
equipo. Usuario `admin`, contraseña `admin`. El dashboard
**Crazyflie · bridge Webots↔Renode** se abre directamente y se refresca cada 5 s.

**Para parar:** `Ctrl+C` en la terminal A y `docker compose down` dentro de
`drone-monitoring/`. Los datos se conservan; `docker compose down -v` los borra.

## Cómo funciona

```text
Webots ◄──► bridge ◄──► Renode (cf2.elf)
              │ --telemetry /tmp/cf_bridge.jsonl   (un objeto JSON por ciclo)
              ▼
     bridge_to_influx.py  ── HTTP /api/v2/write ──►  InfluxDB 2.7  ◄── Grafana 11.6
     (lee el fichero, no toca el bridge)                 bucket "bridge"    dashboard "Crazyflie · bridge"
```

- **No se conecta nada a Renode.** El monitor de Renode (`:9999`) admite un solo
  cliente y lo ocupa el bridge; un segundo cliente rompería el vuelo.
- **El JSONL es el registro primario.** Cada punto lleva el `t_wall` original como
  timestamp, así que reingerir un fichero reescribe exactamente los mismos puntos
  (idempotente). Si InfluxDB estuvo caído, basta con volver a pasar el fichero.
- **InfluxDB y no Prometheus.** Los puntos llegan empujados con su propio
  timestamp y se pueden cargar a posteriori; Prometheus muestrea por scrape y no
  admite rellenar el pasado.
- **Sin dependencias.** El ingestor usa solo la biblioteca estándar de Python 3.

| Fichero | Qué es |
|---|---|
| `bridge_to_influx.py` | el ingestor: JSONL → InfluxDB |
| `docker-compose.yml` | InfluxDB y Grafana, con credenciales fijas de laboratorio |
| `grafana/provisioning/` | configuración automática de Grafana: crea la fuente de datos y le dice dónde está el dashboard |
| `grafana/dashboards/crazyflie_bridge.json` | el dashboard |
| `grafana/build_dashboard.py` | genera el JSON del dashboard |
| `tests/` | pruebas del ingestor |

## Credenciales y seguridad

| Servicio | URL | Usuario | Contraseña |
|---|---|---|---|
| Grafana | `http://<host>:3000` | `admin` | `admin` (en el primer acceso ofrece cambiarla; se puede omitir) |
| InfluxDB | `http://127.0.0.1:8086` | `admin` | `adminadmin` (InfluxDB exige 8 caracteres como mínimo) |

El ingestor usa por defecto el token del compose (`drones-token`), así que no hay
que pasarle credenciales.

Las credenciales son fijas y públicas para que el stack arranque sin configurar
nada. Es adecuado para un laboratorio:

- InfluxDB solo escucha en `127.0.0.1`, así que su token fijo solo sirve desde la
  propia máquina.
- Grafana escucha en todas las interfaces para poder abrirlo desde otro equipo.
  Cualquiera en la misma red podría entrar con `admin`/`admin`; en una red
  compartida, cambia la contraseña en el primer acceso.

Para usar otras credenciales, se definen las variables (`GRAFANA_ADMIN_PASSWORD`,
`INFLUX_TOKEN`, …) en el entorno antes del **primer** `docker compose up`, y el
mismo `INFLUX_TOKEN` al lanzar el ingestor. Después ya no surten efecto: InfluxDB
y Grafana guardan las credenciales al crear sus volúmenes.

## Más usos del ingestor

En modo `--follow`, el ingestor detecta si el fichero se trunca o se sustituye y
empieza de nuevo. Si InfluxDB no responde, reintenta indefinidamente sin perder
datos: el propio fichero hace de búfer. Con `Ctrl+C` vacía el último lote antes de
salir.

```bash
python3 bridge_to_influx.py --file vuelo.jsonl                        # cargar un fichero ya grabado
python3 bridge_to_influx.py --file vuelo.jsonl --stdout | head        # ver el line protocol sin enviar
python3 bridge_to_influx.py --file /tmp/cf_bridge.jsonl --follow --tag host=$(hostname)
python3 bridge_to_influx.py --help
```

Los datos de un fichero grabado aparecen en su hora original: hay que ajustar el
intervalo de tiempo de Grafana a cuando se voló.

Los `--tag clave=valor` fijos (p. ej. `host`, `run`, `firmware`) forman parte de la
identidad de la serie: reingerir el mismo fichero con otros tags crea series
duplicadas en lugar de sobrescribir.

## Esquema en InfluxDB

| Medida | Tags | Contenido |
|---|---|---|
| `bridge_cycle` | `session`, `mode`, tags de `--tag` | un punto por ciclo de control |
| `bridge_event` | `session`, `kind`, tags de `--tag` | `session_start`, `session_end` y cualquier otro `kind` |

`session` es la hora UTC del `session_start` (`2026-10-23T07:37:39Z`). Un mismo
JSONL puede contener varias sesiones, porque el bridge abre el fichero en modo
append; cada `session_start` abre una sesión nueva.

**Campos crudos de `bridge_cycle`**, tal como los escribe el bridge:

| Campo | Significado |
|---|---|
| `sim` | tiempo de Webots (s); 0 con FakePlant |
| `alt` | altitud de la planta, GPS de Webots (m) |
| `vz` | velocidad vertical **calculada por el bridge con reloj de pared** |
| `thrust`, `u` | consigna de empuje RPYT escrita en el firmware y salida del PD+I de altura del bridge |
| `m1`–`m4` | ratios de motor del mixer del firmware (0–1), leídos de Renode |
| `omega1`–`omega4` | velocidad de hélice aplicada en Webots (rad/s, con signo) |
| `gyro_x/y/z` | giróscopo enviado al BMI088 emulado (°/s) |
| `accel_x/y/z` | acelerómetro enviado al BMI088 (mg); sintético, el PROTO no tiene acelerómetro |
| `pressure` | presión enviada al BMP388 (Pa) |

Cualquier campo nuevo que el bridge añada se ingiere sin tocar el ingestor:
números → float, listas → `<clave>_1…n`, diccionarios → `<clave>_<subclave>`.

**Campos derivados** que calcula el ingestor (nunca sustituyen a uno del bridge):

| Campo | Definición | Para qué |
|---|---|---|
| `cycle` | número de ciclo dentro de la sesión | recuentos, huecos |
| `alt_target`, `alt_error` | consigna del `session_start`; `alt − alt_target` | error estacionario |
| `motor_mean`, `motor_spread` | media y `max − min` de `m1–m4` | actuación de actitud |
| `dt_wall`, `loop_hz` | Δt de pared entre ciclos y su inversa | ritmo real del bucle |
| `dt_sim`, `sim_speed` | Δ`sim` entre ciclos; `dt_sim / dt_wall` | ¿va la simulación en tiempo real? |
| `vz_sim` | `Δalt / Δsim` | velocidad vertical física |

`vz` y `vz_sim` difieren a propósito: el bridge deriva `vz` con reloj de pared, y
cuando la simulación no va en tiempo real esa `vz` no es la velocidad física. En
el log de ejemplo de CF_renode_emulator la subida alcanza 1,16 m/s físicos y el
bridge registra 0,40.

Un `NaN` o infinito no cabe en InfluxDB: se descarta y se cuenta en el resumen
final (`valores_no_finitos`), porque en el bridge es una anomalía.

## Dashboard

Secciones: resumen del intervalo, vuelo, actuadores (firmware → planta), sensores
(planta → firmware), fidelidad de la co-simulación y tabla de eventos. La variable
**Sesión** filtra por sesión; cada panel explica su señal en el icono ⓘ.

El JSON aprovisionado se genera con `grafana/build_dashboard.py`. Para añadir un
panel, se edita el generador y se ejecuta `python3 grafana/build_dashboard.py`;
Grafana recarga el fichero en unos 10 s. Los cambios hechos en la interfaz se
pierden al regenerar.

## Pruebas

```bash
python3 -m unittest discover -s tests -v
```

No necesitan InfluxDB. Cubren el escapado del line protocol, el aplanado de
campos, las sesiones y las derivadas, la lectura de líneas a medio escribir y el
truncado del fichero. Si el log de ejemplo de CF_renode_emulator
(`cf_bridge_smoke_jsonl_example.txt`) está en la carpeta que contiene a
`drone-monitoring/`, se prueba también con él; si no, esa prueba se omite.

## Lo que la telemetría actual todavía no contiene

El ingestor solo ve lo que el bridge escribe. Faltan, y conviene añadirlos en el
bridge (`CycleSample.extra` y `write_raw` ya lo permiten sin romper el formato):

- **Parámetros de la sesión**: `hover_thrust`, ganancias del PD+I, `imu_hz`,
  `loop_hz` pedido, hash de `cf2.elf`. Sin ellos no se puede reconstruir qué se
  ejecutó; en el log de ejemplo, `HOVER_THRUST=31000` solo se deduce
  algebraicamente.
- **Tiempo de Renode**: `cpu_stats` (tiempo virtual, `sim_speed` del firmware).
  Hoy `sim_speed` mide solo Webots.
- **Latencia de cada consulta al monitor** (`imu_stream_set`, `setpoint_rpyt`,
  `motor_stats`). El log de ejemplo sugiere que es el cuello de botella: uno de
  cada tres ciclos tarda 0,20 s más, justo cuando el bridge hace una consulta
  extra (`setpoint_stats`) para su línea de estado cada 2 s.
- **Intervenciones del supervisor**: el teletransporte `unstick_for_takeoff` solo
  se imprime por pantalla, así que en los datos no se distingue de la física.
- **Estado del firmware**: `calib_status`, `preflight_status`, `setpoint_stats`.

Cada consulta extra a Renode alarga el bucle (~0,2 s según el log de ejemplo), y
eso cambia el experimento que se está midiendo. Medir tiempos con
`time.perf_counter()` es gratis; consultar `cpu_stats` debe hacerse con poca
frecuencia y dejando constancia de ello.
