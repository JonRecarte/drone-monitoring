# drone-monitoring

Guarda en InfluxDB y muestra en Grafana la telemetría del bridge de
[CF_renode_emulator](https://github.com/danielalaez/CF_renode_emulator).

## Descargar

En la máquina donde corre CF_renode_emulator (necesita Docker y Python 3.10+):

```bash
git clone https://github.com/JonRecarte/drone-monitoring.git
cd drone-monitoring
```

## Ejecutar

```bash
# Una vez: arranca InfluxDB y Grafana
docker compose up -d

# Terminal A, dentro de drone-monitoring/
python3 bridge_to_influx.py --file /tmp/cf_bridge.jsonl --follow

# Terminal B, dentro de CF_renode_emulator/ (misma ruta en TELEMETRY y --file)
TELEMETRY=/tmp/cf_bridge.jsonl FAKE=0 ALT=0.5 ./scripts/run_hover_demo.sh
```

Para parar: `Ctrl+C` en la terminal A y `docker compose down`.

## Acceso

| Servicio | URL | Usuario | Contraseña |
|---|---|---|---|
| Grafana | http://localhost:3000 | `admin` | `admin` |
| InfluxDB | http://localhost:8086 | `admin` | `adminadmin` |
