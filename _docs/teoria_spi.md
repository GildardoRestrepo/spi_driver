---
title: Teoría del protocolo SPI
created: 2026-10-06
time: 07:04pm
creator: Gilbert
last update: 2026-10-06
update by: Gilbert
type: referencia
status: borrador
fase: ""
area: microcontroladores
editor: Gilbert
order: 2
tags:
  - tipo/referencia
---

# Teoría del protocolo SPI

> [!success] Resumen
> Qué es SPI, cómo se intercambian los bits (modos CPOL/CPHA), cómo está construido el periférico SPI de la STM32F407, qué hace cada registro que usa el driver y cómo habla el ADXL345 por este bus. Fuente principal: manual de referencia RM0090, capítulo 28.

---

## 1. Qué es SPI

**SPI (Serial Peripheral Interface)** es un bus serial **síncrono** y **full-duplex** creado por `Motorola` para comunicar un microcontrolador con periféricos cercanos (sensores, memorias, pantallas).

- **Síncrono:** El maestro genera el reloj y ambos lados muestrean los datos con ese reloj. No hace falta acordar un baud rate como en UART.
- **Full-duplex:** En cada ciclo de reloj se envía un bit y se recibe otro al mismo tiempo.
- **Maestro–esclavo:** Solo el maestro inicia las transferencias. El esclavo nunca habla por iniciativa propia.

---

## 2. Señales del bus

| Señal | Nombre en STM32 | Dirección | Función |
|-------|-----------------|-----------|---------|
| SCK   | SCK  | Maestro → esclavo | Reloj de la transferencia |
| MOSI  | MOSI | Maestro → esclavo | Datos de salida del maestro |
| MISO  | MISO | Esclavo → maestro | Datos de salida del esclavo |
| CS / SS | NSS | Maestro → esclavo | Selección del esclavo, **activa en bajo** |

Mientras CS está en alto, el esclavo ignora el reloj y deja su MISO en alta impedancia. Por eso varios esclavos pueden compartir SCK, MOSI y MISO, cada uno con su propia línea CS.

---

## 3. Funcionamiento: dos registros de desplazamiento

El maestro y el esclavo tienen cada uno un registro de desplazamiento de 8 bits, conectados en anillo:
```
        MAESTRO                              ESCLAVO
  ┌───────────────────┐   MOSI     ┌───────────────────┐
  │ b7 b6 ... b1 b0   │ ─────────▶ │ b7 b6 ... b1 b0   │
  └───────────────────┘            └───────────────────┘
            ▲            MISO                │
            └─────────────────────────────────┘
                    SCK (lo genera el maestro)
```

En cada pulso de reloj, cada lado saca un bit por su salida y mete por su entrada el bit que le llega. Después de 8 pulsos, **los dos registros intercambiaron su contenido**.

Consecuencias prácticas:

1. **Leer siempre implica escribir.** Para recibir un byte, el maestro tiene que enviar uno (normalmente `0x00`, el *byte dummy*), porque solo así genera los 8 pulsos de reloj.
2. **Escribir siempre implica recibir.** Lo que llega mientras se envía un comando suele ser basura y se descarta, pero hay que leerlo igual para no dejar datos pendientes en el receptor (ver error OVR más abajo).

---

## 4. Modos de reloj: CPOL y CPHA

Dos bits definen la forma del reloj y en qué flanco se leen los datos:

- **CPOL (polaridad):** nivel de SCK en reposo. `0` = bajo, `1` = alto.
- **CPHA (fase):** en qué flanco se captura el dato. `0` = primer flanco, `1` = segundo flanco.

| Modo | CPOL | CPHA | SCK en reposo | Captura del dato |
|------|------|------|---------------|------------------|
| 0    | 0    | 0    | Bajo | Flanco de subida |
| 1    | 0    | 1    | Bajo | Flanco de bajada |
| 2    | 1    | 0    | Alto | Flanco de bajada |
| 3    | 1    | 1    | Alto | Flanco de subida |

Maestro y esclavo **deben usar el mismo modo**; si no, los bits se leen corridos. El ADXL345 usa el **modo 3**:

```
CS    ‾‾\_______________________________________________/‾‾‾
SCK   ‾‾‾‾‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾\_/‾‾‾‾‾‾‾
MOSI  ─────X b7 X b6 X b5 X b4 X b3 X b2 X b1 X b0 X──────
               ↑    ↑    ↑    ↑    ↑    ↑    ↑    ↑
               captura en cada flanco de subida
```

El reloj reposa en alto, el dato cambia en el flanco de bajada y se lee en el de subida.

---

## 5. Comparación con otros buses seriales

| Característica | SPI | I2C | UART |
|----------------|-----|-----|------|
| Líneas | 4 (+1 CS por esclavo) | 2 | 2 |
| Reloj | Sí, del maestro | Sí, del maestro | No (baud rate acordado) |
| Dúplex | Full | Half | Full |
| Selección de esclavo | Línea CS | Dirección en la trama | Punto a punto |
| Velocidad típica | Hasta decenas de MHz | 100 kHz – 1 MHz | Hasta ~1 Mbps |
| Confirmación (ACK) | No | Sí | No |

**Ventajas de SPI:** muy rápido, hardware simple, sin direcciones ni ACK, salidas push-pull (flancos limpios).
**Desventajas:** más pines, una línea CS por esclavo, sin confirmación de que el esclavo recibió el dato, pensado para distancias cortas (dentro de la misma tarjeta).

---

## 6. El periférico SPI en la STM32F407

La STM32F407 tiene tres SPI. Cada uno cuelga de un bus distinto, y de ese bus toma su reloj:

| Periférico | Bus | Dirección base | Bit de reloj en RCC |
|------------|-----|----------------|---------------------|
| SPI1 | APB2 | `0x4001 3000` | `RCC_APB2ENR.SPI1EN` (bit 12) |
| SPI2 | APB1 | `0x4000 3800` | `RCC_APB1ENR.SPI2EN` (bit 14) |
| SPI3 | APB1 | `0x4000 3C00` | `RCC_APB1ENR.SPI3EN` (bit 15) |

Usamos **SPI2**. Todo periférico arranca con el reloj apagado para ahorrar energía, así que el primer paso de cualquier driver es encender su bit en el RCC. Si no se hace, las escrituras a sus registros no tienen efecto.

### Diagrama interno simplificado

```
  bus APB1 ──▶ SPI_DR (escritura) ──▶ Buffer TX ──▶ Registro de ──▶ MOSI
                                                    desplazamiento
  bus APB1 ◀── SPI_DR (lectura)  ◀── Buffer RX ◀──               ◀── MISO
                                                         ▲
  fPCLK1 ──▶ Divisor BR[2:0] ──▶ Lógica de control ──────┴───▶ SCK
                                 (CR1, CR2, SR)
```

`SPI_DR` es una sola dirección con dos buffers detrás: escribir en ella llena el buffer de transmisión y leerla devuelve el buffer de recepción.

### Pines

Los pines del SPI son GPIO configurados en **función alternativa**. Para SPI2 en el puerto B:

| Pin | Función | Configuración GPIO |
|-----|---------|--------------------|
| PB12 | CS | Salida push-pull (la controla el software) |
| PB13 | SPI2_SCK  | Función alternativa AF5 |
| PB14 | SPI2_MISO | Función alternativa AF5 |
| PB15 | SPI2_MOSI | Función alternativa AF5 |

Registros GPIO involucrados: `GPIOx_MODER` (modo: `10` = función alternativa, `01` = salida), `GPIOx_AFRH` (número de función alternativa para los pines 8–15), `GPIOx_OSPEEDR` (velocidad del pin) y `GPIOx_BSRR` (subir o bajar CS de forma atómica).

### Velocidad del reloj SPI

El reloj SCK sale de dividir el reloj del bus (`fPCLK`) por una potencia de 2:

```
fSCK = fPCLK / 2^(BR + 1)
```

Sin configurar el PLL, la STM32F407 arranca con el oscilador interno HSI a **16 MHz**, y APB1 corre a la misma frecuencia. El ADXL345 acepta hasta 5 MHz:

| BR[2:0] | Divisor | fSCK con fPCLK1 = 16 MHz |
|---------|---------|--------------------------|
| `000` | /2   | 8 MHz (demasiado para el ADXL345) |
| `001` | /4   | 4 MHz |
| `010` | /8   | 2 MHz |
| `011` | /16  | 1 MHz |

---

## 7. Registros del SPI

### SPI_CR1 — Control 1 (offset `0x00`)

| Bit | Nombre | Función | Valor para el ADXL345 |
|-----|--------|---------|-----------------------|
| 15 | BIDIMODE | `0` = dos líneas de datos (MOSI y MISO) | 0 |
| 14 | BIDIOE | Dirección en modo bidireccional | 0 |
| 13 | CRCEN | Habilita el cálculo de CRC por hardware | 0 |
| 12 | CRCNEXT | La próxima transferencia es el CRC | 0 |
| 11 | DFF | Tamaño de trama: `0` = 8 bits, `1` = 16 bits | 0 |
| 10 | RXONLY | `0` = full-duplex | 0 |
| 9 | SSM | Manejo de NSS por software | **1** |
| 8 | SSI | Valor interno de NSS cuando SSM = 1 | **1** |
| 7 | LSBFIRST | `0` = primero el MSB | 0 |
| 6 | SPE | Habilita el periférico | **1** (al final) |
| 5:3 | BR[2:0] | Divisor del reloj | p. ej. `011` |
| 2 | MSTR | `1` = maestro | **1** |
| 1 | CPOL | Polaridad del reloj | **1** |
| 0 | CPHA | Fase del reloj | **1** |

> [!warning] SSM y SSI
> En modo maestro, si la entrada NSS interna se ve en bajo, el hardware asume que otro maestro tomó el bus: activa el error **MODF**, borra MSTR y SPE, y el SPI deja de funcionar. Como CS se maneja con un GPIO normal, se pone SSM = 1 y SSI = 1 para que NSS interno quede siempre en alto.

Los bits de configuración (BR, CPOL, CPHA, DFF, MSTR) deben escribirse **con SPE = 0**; SPE se pone en 1 al final.

### SPI_CR2 — Control 2 (offset `0x04`)

Habilita interrupciones y DMA. El driver trabaja por *polling*, así que se deja en `0x0000`.

| Bit | Nombre | Función |
|-----|--------|---------|
| 7 | TXEIE | Interrupción cuando TXE = 1 |
| 6 | RXNEIE | Interrupción cuando RXNE = 1 |
| 5 | ERRIE | Interrupción por errores (OVR, MODF, CRCERR) |
| 4 | FRF | `0` = formato Motorola, `1` = formato TI |
| 2 | SSOE | El hardware maneja el pin NSS como salida |
| 1 | TXDMAEN | Solicitud de DMA con TXE |
| 0 | RXDMAEN | Solicitud de DMA con RXNE |

### SPI_SR — Estado (offset `0x08`, valor de reset `0x0002`)

| Bit | Nombre | Significado | Cómo se limpia |
|-----|--------|-------------|----------------|
| 7 | BSY | El SPI está transfiriendo o el buffer TX no está vacío | Hardware |
| 6 | OVR | Llegó un byte nuevo sin haber leído el anterior | Leer `SPI_DR` y luego `SPI_SR` |
| 5 | MODF | Fallo de modo (NSS en bajo siendo maestro) | Leer `SPI_SR` y luego escribir `SPI_CR1` |
| 4 | CRCERR | El CRC recibido no coincide | Escribir 0 |
| 1 | TXE | Buffer de transmisión vacío: se puede escribir `SPI_DR` | Escribir `SPI_DR` |
| 0 | RXNE | Buffer de recepción con dato: se puede leer `SPI_DR` | Leer `SPI_DR` |

El valor de reset es `0x0002` porque al arrancar el buffer TX está vacío (TXE = 1).

### SPI_DR — Datos (offset `0x0C`)

Escribir aquí inicia la transmisión; leer aquí devuelve el último byte recibido y limpia RXNE.

---

## 8. Secuencias del driver

### Inicialización (modo maestro)

1. Encender el reloj del GPIOB (`RCC_AHB1ENR.GPIOBEN`) y del SPI2 (`RCC_APB1ENR.SPI2EN`).
2. Configurar PB13–PB15 como función alternativa AF5 y PB12 como salida; dejar CS en alto.
3. Con SPE = 0, escribir en `SPI_CR1`: BR, CPOL = 1, CPHA = 1, DFF = 0, LSBFIRST = 0, SSM = 1, SSI = 1, MSTR = 1.
4. Poner SPE = 1.

### Transferir un byte

1. Esperar **TXE = 1** (hay espacio en el buffer de transmisión).
2. Escribir el byte en `SPI_DR`: el hardware genera los 8 pulsos de reloj.
3. Esperar **RXNE = 1** (llegó el byte del esclavo).
4. Leer `SPI_DR`: devuelve el byte recibido y limpia RXNE.

Leer `SPI_DR` en cada transferencia, aunque el dato no sirva, evita el error de **overrun (OVR)**.

### Antes de subir CS o apagar el SPI

Según el RM0090 (sección 28.3.8), en full-duplex: esperar RXNE = 1, luego TXE = 1, luego **BSY = 0**. Si se sube CS con BSY = 1, se corta el último byte a la mitad.

---

## 9. SPI en el ADXL345 (GY-291)

### Parámetros del bus

| Parámetro | Valor |
|-----------|-------|
| Modo | 3 (CPOL = 1, CPHA = 1) |
| Reloj máximo | 5 MHz |
| Orden de bits | MSB primero |
| Variante | 4 hilos (bit SPI = 0 en `DATA_FORMAT`, valor por defecto) |

### Formato del primer byte (comando)

```
  bit:   7      6      5   4   3   2   1   0
       ┌─────┬──────┬───────────────────────┐
       │ R/W │  MB  │  dirección (6 bits)   │
       └─────┴──────┴───────────────────────┘
```

- **R/W:** `1` = lectura, `0` = escritura.
- **MB (multi-byte):** `1` = después del primero, el sensor sigue entregando los registros siguientes sin enviar otra dirección.

Ejemplos:

| Operación | Bytes en MOSI | Bytes útiles en MISO |
|-----------|---------------|----------------------|
| Leer `DEVID` (0x00) | `0x80`, `0x00` | 2.º byte = `0xE5` |
| Escribir `POWER_CTL` (0x2D) = 0x08 | `0x2D`, `0x08` | Ninguno |
| Leer los 6 bytes de datos (0x32–0x37) | `0xF2`, 6 × `0x00` | Bytes 2 a 7 |

`0xF2` = `1` (lectura) + `1` (multi-byte) + `110010` (0x32).

### Registros del sensor que usará el driver

| Dirección | Nombre | Uso |
|-----------|--------|-----|
| `0x00` | DEVID | Identificador fijo `0xE5`: confirma que la comunicación funciona |
| `0x2D` | POWER_CTL | Bit 3 (Measure) = 1 inicia las mediciones |
| `0x31` | DATA_FORMAT | Rango (±2/4/8/16 g) y resolución completa (FULL_RES) |
| `0x32`–`0x37` | DATAX0…DATAZ1 | Aceleración X, Y, Z; 16 bits en complemento a 2, primero el byte bajo |

Con FULL_RES = 1 la escala es de aproximadamente **3,9 mg por LSB** en cualquier rango. Leer los seis bytes en una sola ráfaga (MB = 1) garantiza que los tres ejes pertenecen a la misma muestra.

---

## Enlaces

[[reto_3_spi_driver|Reto 3 SPI]]
[[reto_3_spi|Enunciado del reto]] · Manual de referencia RM0090: `datasheets/dm00031020.pdf`, capítulo 28 (SPI)
