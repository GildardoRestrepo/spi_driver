---
type: readme
created: 2026-10-06
---

# Reto 3 — Driver SPI Bare-Metal para STM32F407

> [!success] Resumen
> Driver SPI escrito a nivel de registros (sin STM32Cube HAL ni LL) para la **STM32F407VET6**. La prueba de concepto lee el acelerómetro **GY-291 (ADXL345)** y muestra la aceleración en X, Y y Z por la consola **SWO** del depurador.

**Curso:** Microprocesadores — Universidad Pontificia Bolivariana
**Integrantes:** Gildardo Estevan Restrepo · Marco Aurelio Guardia

---

## Estructura del repositorio

```
/inc     Cabeceras (.h)
/src     Código fuente (.c)
/_docs   Enunciado, teoría y datasheets
```

---

## Reparto de trabajo

| Etapa | Responsable | Estado |
|-------|-------------|--------|
| 1. Bajo nivel (registros) | Marco | Completada |
| 2. Alto nivel (API y sensor) | Gildardo | Completada |
| 3. Visualización por SWO | Marco | En revisión |
| 4. Fundamentos teóricos | Gildardo | Primera versión |

Flujo de trabajo: una rama por etapa, Pull Request a `main` y revisión del otro integrante antes del merge.

---

## Arquitectura y guía de la API

```
main.c        Aplicación: lee el sensor en un ciclo y lo imprime
adxl345.c/.h  Driver del sensor (registros del ADXL345, escala a mg)
drv_spi.c/.h  API pública SPI (tramas completas con CS)
ll_spi.c/.h   Bajo nivel: registros de RCC, GPIOB y SPI2

swo.c/.h      Salida de texto por SWO (ITM y TPIU); redirige printf
```

Cada capa solo llama a la inmediatamente inferior; solo `ll_spi.c` y `swo.c` tocan registros.

### drv_spi.h

| Función | Descripción |
|---------|-------------|
| `void SPI_Init(void)` | SPI2 maestro, modo 3, 1 MHz, CS en alto. Llamarla de nuevo reinicia el periférico |
| `SPI_Status_t SPI_WriteRegister(uint8_t cmd, uint8_t value)` | Trama de 2 bytes: comando y valor |
| `SPI_Status_t SPI_ReadRegisters(uint8_t cmd, uint8_t *buffer, uint8_t length)` | Envía el comando y lee `length` bytes en la misma trama |

Devuelven `SPI_OK` o `SPI_ERROR` (venció un tiempo de espera o los parámetros no son válidos).

### adxl345.h

| Función | Descripción |
|---------|-------------|
| `ADXL345_Status_t ADXL345_Init(void)` | Inicializa el SPI, verifica `DEVID = 0xE5`, configura ±16 g con resolución completa y activa la medición |
| `ADXL345_Status_t ADXL345_ReadAccel(ADXL345_Accel_t *accel)` | Lee X, Y y Z de una misma muestra, en mg, y verifica en la misma trama que el sensor siga configurado |

Devuelven `ADXL345_OK`, `ADXL345_ERR_SPI` (venció un tiempo de espera), `ADXL345_ERR_ID` (sensor ausente o mal cableado al iniciar) o `ADXL345_ERR_CONFIG` (el sensor se desconectó o se reinició; se recupera llamando de nuevo a `ADXL345_Init`).

### swo.h

| Función | Descripción |
|---------|-------------|
| `void SWO_Init(void)` | Configura PB3 como TRACESWO y habilita TPIU e ITM (puerto 0) a 2 MHz. Llamarla antes del primer `printf` |
| `void SWO_PrintChar(char c)` | Envía un carácter. Si la cola no se desocupa a tiempo lo descarta, sin bloquear |
| `void SWO_PrintString(const char *str)` | Envía una cadena |

`swo.c` define `_write`, así que `printf` sale por SWO sin modificar `syscalls.c`.

---

## Hardware y conexiones

| GY-291 | STM32F407VET6 | Función |
|--------|---------------|---------|
| VCC    | 3V3           | Alimentación |
| GND    | GND           | Tierra |
| CS     | PB12          | Chip Select (GPIO salida) |
| SCL    | PB13          | SPI2_SCK (AF5) |
| SDO    | PB14          | SPI2_MISO (AF5) |
| SDA    | PB15          | SPI2_MOSI (AF5) |

| Depurador | STM32F407VET6 |
|-----------|---------------|
| SWDIO / SWCLK / GND | PA13 / PA14 / GND |
| SWO       | PB3 |

Se usa **SPI2** para no chocar con los LEDs de la placa (PA6/PA7) ni con la memoria W25Q16 conectada a SPI1.

---

## Ver las lecturas por SWO (J-Link)

El firmware saca el texto por PB3 a **2 MHz** con el núcleo a **16 MHz** (HSI). El depurador debe usar los mismos valores; si no coinciden, la consola muestra caracteres extraños.

1. Conectar el J-Link al conector JTAG de 20 pines de la placa: SWO ya llega a PB3 por el pin 13 (TDO). Con cables sueltos, unir el pin 13 del J-Link a PB3.
2. En STM32CubeIDE: *Debug Configurations → Debugger*, sonda **SEGGER J-LINK**, interfaz **SWD**.
3. En la misma pestaña, *Serial Wire Viewer (SWV)*: **Enable**, *Core Clock* = `16.0` MHz, *SWO Clock* = `2000` kHz.
4. Iniciar la depuración y abrir *Window → Show View → SWV → SWV ITM Data Console*.
5. En la consola, *Configure trace*: activar el **puerto 0** del ITM. Luego *Start Trace* y *Resume*.

Ejemplo de salida con el sensor quieto sobre la mesa (los valores cambian en cada lectura; Z debe quedar cerca de 1000 mg):

```
Reto 3 - SPI: buscando el ADXL345...
ADXL345 listo: +/-16 g, 3.9 mg por LSB
X=    12  Y=   -27  Z=  1003  mg
```

---

## Enlaces

[[reto_3_spi_driver|Reto 3 SPI]]
- Enunciado del reto
- Teoría del protocolo SPI
- [Manual de referencia RM0090](_docs/datasheets/dm00031020.pdf)
- [Esquemático de la placa](_docs/datasheets/stm32f407vet6_schematics.pdf)
