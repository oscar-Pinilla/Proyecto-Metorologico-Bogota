# Proyecto-Metorologico-Bogota
# Sistema de Monitoreo Meteorológico 🌦️

> Sistemas Operativos · Pontificia Universidad Javeriana  
> Semestre 2026-1

---

## ¿De qué trata esto?

Este proyecto implementa un sistema de monitoreo de estaciones meteorológicas usando **IPC con Pipes Nominales (FIFOs)** y **concurrencia con hilos POSIX**. La idea es simular un sistema tipo SCADA donde múltiples agentes leen sensores CSV y le mandan los datos a un monitor central que los procesa en paralelo.

Al final, el monitor consolida todo en un archivo CSV y da un parte meteorológico de Bogotá según los promedios calculados.

---

## Integrantes

| Nombre | Usuario GitHub |
|---|---|
| Oscar Pinilla | `@opini` |
| David Pedraza | `@dpedraza` |
| Johan Barreto | `@jbarreto` |

---

## Arquitectura del sistema

```
  [agenteM - EK]  ──┐
  [agenteM - ET]  ──┼──→  FIFO (pipeNominal)  ──→  [monitor]
  [agenteM - EU]  ──┘                                   │
                                                         ├── Hilo Recolector (lee el pipe)
                                                         ├── Hilo Consumidor EK
                                                         ├── Hilo Consumidor ET
                                                         └── Hilo Consumidor EU
                                                                    │
                                                             consolidado.csv
                                                             + parte meteorológico
```

Cada `agenteM` es un proceso independiente (productor) que lee su CSV, filtra los datos inválidos y los manda por el pipe. El `monitor` tiene un hilo recolector que distribuye las mediciones a 3 buffers circulares (uno por estación), y 3 hilos consumidores que procesan y escriben el archivo final.

---

## Estructura del proyecto

```
proyecto-c/
├── src/
│   ├── monitor.c       # Monitor central: hilos, buffers circulares, semáforos
│   ├── agenteM.c       # Agente productor: lee CSV y escribe al pipe
│   └── validaciones.c  # Filtro de rangos físicos para los datos del sensor
├── include/
│   ├── comun.h         # Struct Medicion y constantes de rangos (Tabla 1)
│   └── validaciones.h  # Prototipo de validarRangos()
├── bin/                # Ejecutables generados por make
├── sensorken.csv       # Datos de ejemplo – Estación EK (Kennedy)
├── sensorteu.csv       # Datos de ejemplo – Estación ET (Teusaquillo)
├── sensorusa.csv       # Datos de ejemplo – Estación EU (Usaquén)
└── Makefile
```

---

## Rangos válidos de los sensores (Tabla 1)

Los agentes filtran mediciones fuera de estos rangos antes de enviarlas. Si una medición no pasa, el agente la descarta con un mensaje `[SCADA - ALERTA]`.

| Variable | Mínimo | Máximo |
|---|---|---|
| Humedad relativa (%) | 77 | 100 |
| Punto de rocío (°C) | 3 | 12 |
| Presión atmosférica (hPa) | 740 | 760 |

---

## Compilación

```bash
make
```

Genera `bin/monitor` y `bin/agenteM`. Para limpiar:

```bash
make clean
```

---

## Cómo correrlo

El sistema necesita que el monitor esté corriendo **antes** de lanzar los agentes, porque el pipe bloquea hasta que alguien abra el otro extremo.

**1. Terminal 1 – Arrancar el monitor:**
```bash
./bin/monitor -b 5 -p pipeNominal
```
- `-b` → tamaño del buffer circular por estación
- `-p` → nombre del pipe nominal (FIFO)

**2. Terminales 2, 3 y 4 – Lanzar los agentes (uno por terminal):**
```bash
./bin/agenteM -f sensorken.csv -t 1 -p pipeNominal
./bin/agenteM -f sensorteu.csv -t 1 -p pipeNominal
./bin/agenteM -f sensorusa.csv -t 1 -p pipeNominal
```
- `-f` → archivo CSV del sensor
- `-t` → segundos entre cada envío (simula frecuencia de muestreo)
- `-p` → mismo pipe que el monitor

---

## Formato del CSV de entrada

```
Estacion,Humedad,Rocio,Presion,Hora
EK,95,11,745,08:00:00
EK,92,10,748,08:30:00
.
```

La línea con `.` indica el fin del archivo para ese agente.

---

## Salida del sistema

**Consola del monitor** (ejemplo):
```
Parte Meteorológico Bogotá: "Lluvioso"
Monitor finalizado. Los datos consolidados están en 'consolidado.csv'.
```

**Consola de un agente** (ejemplo):
```
Agente [sensorken.csv] iniciado. Transmitiendo datos cada 1 seg...
[SCADA - ALERTA] Datos descartados en EK (Fuera de rango): H:50 R:2 P:800
Agente [sensorken.csv]: Transmisión finalizada con éxito.
```

**`consolidado.csv`:** contiene todas las mediciones válidas recibidas de las tres estaciones.

---

## Lógica del parte meteorológico

El monitor calcula los promedios globales de humedad, rocío y presión y clasifica así:

| Condición | Descripción |
|---|---|
| `avgH > 90`, `avgR > 9`, `avgP < 750` | ☔ **Lluvioso** |
| `80 ≤ avgH ≤ 95`, `avgR > 8`, `avgP = 751` | ⛅ **Nublado** |
| `avgH < 80`, `5 ≤ avgR ≤ 8`, `avgP > 754` | 🌤️ **Fresco** |
| Cualquier otro caso | 🌫️ **Variable** |

---

## Conceptos aplicados

- `mkfifo()` / `open()` — Creación y uso de Pipes Nominales (FIFO) para IPC
- `pthread_create()` / `pthread_join()` — Manejo de hilos POSIX
- `sem_init()` / `sem_wait()` / `sem_post()` — Semáforos para el patrón Productor/Consumidor
- `pthread_mutex_lock()` — Secciones críticas para el archivo de salida y estadísticas globales
- Buffers circulares sincronizados por estación
- `getopt()` — Parsing de argumentos por línea de comandos

---

## Dificultades que encontramos

- El monitor se **bloqueaba** al abrir el pipe si ningún agente había arrancado aún. Hay que lanzarlos casi al tiempo.
- Cerrar bien los semáforos y detectar el `FIN_<estacion>` correctamente para que los hilos consumidores no se queden esperando infinitamente.
- El orden de llegada de los datos al pipe no es determinístico cuando los agentes corren en paralelo.

---

## Entorno de desarrollo

- Ubuntu 22.04 / 24.04
- GCC 11+ con flags `-Wall -Wextra -g`
- Biblioteca `pthread` (`-lpthread`)

---

*Trabajo académico – Sistemas Operativos, Pontificia Universidad Javeriana, 2026.*
