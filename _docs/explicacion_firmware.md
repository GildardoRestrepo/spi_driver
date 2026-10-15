---
type: referencia
created: 2026-10-14
---

# Explicación del firmware

> [!success] Resumen
> Recorrido por el código del proyecto para preparar la presentación: por qué está separado en `inc/` y `src/`, qué hace cada archivo, bloque a bloque, y por qué existe. Escrito para alguien que sabe programar pero no conoce C.

---

## 0. Mini glosario de C

Lo mínimo para leer el código del proyecto.

| Elemento | Ejemplo | Qué significa |
| -------- | ------- | ------------- |
| Comentario | `/* ... */` | Texto que el compilador ignora. `/** ... */` es el mismo comentario con formato de documentación (`@brief`, `@param`, `@return`) |
| `#include` | `#include "ll_spi.h"` | Pega el contenido de otro archivo en este punto antes de compilar. Con `< >` es de la biblioteca estándar; con `" "` es nuestro |
| `#define` | `#define CS_PIN 12U` | Constante con nombre: antes de compilar, cada `CS_PIN` se reemplaza por `12U`. No ocupa memoria |
| Sufijos `U`, `UL` | `12U`, `0x40003800UL` | El número es sin signo (`U`) y de 32 bits (`UL`). Evita sorpresas en operaciones de bits |
| Hexadecimal | `0xE5` | Número en base 16. Se usa porque cada dígito son 4 bits exactos |
| Tipos de tamaño fijo | `uint8_t`, `int16_t`, `uint32_t` | Entero sin signo de 8 bits, con signo de 16 bits, sin signo de 32 bits. Vienen de `<stdint.h>` |
| `void` | `void SPI_Init(void)` | "Nada": la función no devuelve valor o no recibe parámetros |
| Función | `uint8_t f(uint8_t x) { ... }` | Tipo de retorno, nombre, parámetros y cuerpo |
| Prototipo | `void LL_SPI_Init(void);` | Solo la "firma" de la función, sin cuerpo y terminada en `;`. Le dice al compilador que existe en otro lado |
| `static` (en función o variable global) | `static uint8_t spi_error;` | Privado: solo se ve dentro de ese archivo `.c` |
| `volatile` | `volatile uint32_t CR1;` | "No asumas nada, lee y escribe siempre en memoria". Obligatorio en registros de hardware y variables que mira el depurador |
| `struct` | `typedef struct { int16_t x_mg; ... } ADXL345_Accel_t;` | Agrupa varios campos bajo un nombre, como un objeto sin métodos |
| `enum` | `typedef enum { SPI_OK = 0, SPI_ERROR = 1 } SPI_Status_t;` | Lista de constantes con nombre; se usa para códigos de resultado |
| `typedef` | `typedef struct {...} Nombre;` | Le pone un nombre corto a un tipo |
| Puntero | `uint8_t *buffer` | Una dirección de memoria. Permite que una función escriba en una variable de quien la llamó |
| `&variable` | `ADXL345_ReadAccel(&muestra)` | "La dirección de esta variable": así se pasa un puntero |
| `->` y `.` | `SPI2->CR1`, `muestra.x_mg` | Acceso a un campo: `.` sobre una estructura, `->` sobre un puntero a estructura |
| `NULL` | `buffer == NULL` | Puntero que no apunta a nada |
| Conversión (cast) | `(uint8_t)x`, `(void)f()` | Cambia el tipo de un valor. `(void)` descarta a propósito el valor que devuelve una función |

### Operaciones con bits

Los registros del microcontrolador son números de 32 bits donde cada bit (o grupo de bits) controla algo. Por eso el código usa mucho estas operaciones:

| Operación | Ejemplo | Para qué |
| --------- | ------- | -------- |
| `1UL << n` | `1UL << 14` | Un número con solo el bit `n` en 1 (una "máscara") |
| `reg \|= m` | `RCC->APB1ENR \|= (1UL << 14)` | Poner en 1 los bits de `m` sin tocar los demás |
| `reg &= ~m` | `reg &= ~(3UL << 24)` | Poner en 0 los bits de `m` sin tocar los demás (`~` invierte todos los bits) |
| `(reg & m) != 0` | `(SPI2->SR & SPI_SR_TXE)` | Preguntar si un bit está en 1 |

El patrón **leer → limpiar → escribir** aparece en todo el bajo nivel: se copia el registro a una variable, se borran solo los bits del campo que interesa, se ponen los nuevos y se escribe de vuelta. Así no se dañan los campos de otros pines o funciones.

---

## 1. Estructura general del proyecto

```
Reto_3_SPI/
├── inc/            Cabeceras (.h): QUÉ ofrece cada módulo
│   ├── ll_spi.h
│   ├── drv_spi.h
│   ├── adxl345.h
│   └── swo.h
├── src/            Código fuente (.c): CÓMO lo hace
│   ├── ll_spi.c
│   ├── drv_spi.c
│   ├── adxl345.c
│   ├── swo.c
│   └── main.c
├── _docs/          Enunciado, teoría, datasheets y esta explicación
└── README.md       Presentación del repositorio
```

### ¿Por qué separar `.h` y `.c`?

Cada módulo tiene dos archivos con el mismo nombre:

- **`.h` (cabecera, en `inc/`)** es el **contrato**: qué funciones existen, qué reciben, qué devuelven y qué tipos de datos usan. No tiene lógica. Es como la carta de un restaurante.
- **`.c` (fuente, en `src/`)** es la **implementación**: el cuerpo de las funciones, las constantes internas y las variables privadas. Es la cocina.

Quien quiere usar un módulo solo incluye su `.h`. Nunca ve ni depende del `.c`. Esto da tres ventajas:

1. **Encapsulamiento.** Lo que está en el `.c` y no en el `.h` (por ejemplo, las direcciones de los registros) es invisible para el resto. Se puede cambiar sin romper a nadie mientras el contrato siga igual.
2. **Compilación separada.** El compilador traduce cada `.c` por su cuenta a un archivo objeto (`.o`); luego el **enlazador** (*linker*) los une en un solo programa. El `.h` es lo que le permite a `drv_spi.c` llamar a `LL_SPI_Init()` sin conocer su código.
3. **Lo pide el reto.** La rúbrica exige archivos `.c` y `.h` independientes y la estructura `/src` + `/inc`.

### Las capas

```
┌────────────────────────────────────────────┐
│ main.c        Aplicación (prueba de concepto)│
├────────────────────────────────────────────┤
│ adxl345.c/.h  Driver del sensor             │   Alto nivel
├────────────────────────────────────────────┤
│ drv_spi.c/.h  API pública SPI               │   Alto nivel
├────────────────────────────────────────────┤
│ ll_spi.c/.h   Registros RCC, GPIOB y SPI2   │   Bajo nivel
└────────────────────────────────────────────┘
  swo.c/.h      Salida de texto por SWO (registros de depuración)
```

Regla: **cada capa solo llama a la de abajo.** `main.c` no sabe que existe SPI2; `adxl345.c` no sabe qué es un registro; solo `ll_spi.c` y `swo.c` tocan hardware. Si mañana el sensor se conectara a otro SPI, solo cambiaría `ll_spi.c`.

### Convenciones de nombres

| Prefijo | Capa | Ejemplo |
| ------- | ---- | ------- |
| `LL_SPI_` | Bajo nivel (*Low Level*) | `LL_SPI_TransferByte` |
| `SPI_` | API del driver | `SPI_ReadRegisters` |
| `ADXL345_` | Driver del sensor | `ADXL345_ReadAccel` |
| `SWO_` | Salida de texto | `SWO_Init` |

El prefijo dice de qué capa viene cada función al leer el código.

### Lo que no está en el repositorio

Al crear un proyecto **Empty** en STM32CubeIDE, el IDE agrega archivos que no escribimos nosotros y que no son librerías del fabricante prohibidas:

| Archivo | Función |
| ------- | ------- |
| `startup_stm32f407vetx.s` | Código en ensamblador que corre al encender: prepara la memoria y llama a `main()` |
| `STM32F407VETX_FLASH.ld` | Script del enlazador: dónde va el código (Flash) y las variables (RAM) |
| `syscalls.c`, `sysmem.c` | Funciones mínimas que necesita la biblioteca de C (`printf`, memoria dinámica) |

**Del encendido a `main`:** el micro arranca leyendo la tabla de vectores → ejecuta `Reset_Handler` del archivo de arranque → copia las variables iniciales a RAM y pone en cero las demás → llama a `main()`. Desde ahí manda nuestro código.

---

## 2. Archivos en `inc/`

Todas las cabeceras siguen el mismo esqueleto:

```c
#ifndef LL_SPI_H        /* Si todavía no se definió LL_SPI_H... */
#define LL_SPI_H        /* ...defínelo y procesa el archivo     */
...
#endif /* LL_SPI_H */   /* Fin del bloque                       */
```

Se llama **guarda de inclusión** (*include guard*). Si un archivo termina incluido dos veces (por ejemplo, `main.c` incluye `adxl345.h` y algo más también lo incluye), la segunda vez se salta el contenido y no hay definiciones duplicadas.

### 2.1 `ll_spi.h` — contrato del bajo nivel

**Incluye:** `<stdint.h>` (tipos `uint8_t`, `uint32_t`).

**Qué hace:** declara las 5 funciones con las que el resto del programa puede usar el SPI2 sin conocer registros.

**Bloque a bloque:**

1. **Comentario de cabecera.** Resume la configuración fija: SPI2 maestro, 8 bits, MSB primero, modo 3, 1 MHz, y qué pin hace qué (PB12 CS, PB13 SCK, PB14 MISO, PB15 MOSI).
2. **Códigos de error.** `LL_SPI_ERR_NONE` (0) y `LL_SPI_ERR_TIMEOUT` (1): los valores que devuelve `LL_SPI_GetError()`.
3. **Prototipos:**

| Función | Qué hace |
| ------- | -------- |
| `void LL_SPI_Init(void)` | Enciende relojes, configura pines y SPI2, deja CS en alto y borra el error |
| `uint8_t LL_SPI_TransferByte(uint8_t tx)` | Envía un byte y devuelve el que llegó al mismo tiempo. Si vence el tiempo devuelve `0xFF` y marca el error |
| `void LL_SPI_CS_Low(void)` | Baja CS: empieza una trama |
| `void LL_SPI_CS_High(void)` | Espera que el bus termine y sube CS: fin de trama |
| `uint8_t LL_SPI_GetError(void)` | Devuelve el error guardado desde el último `Init` |

**Por qué existe:** es la frontera entre hardware y software. La "unidad" de esta capa es **un byte**; no sabe qué es una trama ni un sensor.

### 2.2 `drv_spi.h` — API pública del SPI

**Incluye:** `<stdint.h>`.

**Qué hace:** declara las funciones que usaría cualquier driver de un dispositivo SPI (no solo el ADXL345).

**Bloque a bloque:**

1. **Comentario de cabecera.** Explica la idea central: cada función arma una **trama completa** (baja CS, transfiere, sube CS) y quien llama entrega el primer byte ya armado.
2. **`SPI_Status_t`.** Enumeración con `SPI_OK` y `SPI_ERROR`: el resultado de cada operación.
3. **Prototipos:**

| Función | Qué hace |
| ------- | -------- |
| `void SPI_Init(void)` | Deja el bus listo (llama al bajo nivel) |
| `SPI_Status_t SPI_WriteRegister(uint8_t cmd, uint8_t value)` | Trama de 2 bytes: comando y valor |
| `SPI_Status_t SPI_ReadRegisters(uint8_t cmd, uint8_t *buffer, uint8_t length)` | Envía el comando y lee `length` bytes que guarda en `buffer` |

`buffer` es un puntero: la función escribe los bytes leídos directamente en el arreglo de quien la llamó.

**Por qué existe:** es la API que pide el enunciado ("funciones públicas con firmas claras"). Su "unidad" es **la trama**. No conoce el ADXL345: el bit de lectura y el de varios bytes los pone la capa del sensor, así que sirve para cualquier esclavo SPI.

### 2.3 `adxl345.h` — contrato del sensor

**Incluye:** `<stdint.h>`.

**Qué hace:** define cómo se le piden datos al acelerómetro y en qué formato llegan.

**Bloque a bloque:**

1. **Comentario de cabecera.** Configuración fija: ±16 g, resolución completa (3,9 mg por unidad), 100 Hz, SPI de 4 hilos en modo 3.
2. **`ADXL345_Status_t`.** Cuatro resultados posibles:

| Código | Significado |
| ------ | ----------- |
| `ADXL345_OK` | Todo bien |
| `ADXL345_ERR_SPI` | El bus no respondió a tiempo |
| `ADXL345_ERR_ID` | Al iniciar, el sensor no se identificó como ADXL345 (ausente o mal cableado) |
| `ADXL345_ERR_CONFIG` | Estaba funcionando y perdió su configuración (desconectado o reiniciado) |

3. **`ADXL345_Accel_t`.** Estructura con tres campos `int16_t`: `x_mg`, `y_mg`, `z_mg`, la aceleración en miligravedades (1000 mg = 1 g).
4. **Prototipos:**

| Función | Qué hace |
| ------- | -------- |
| `ADXL345_Status_t ADXL345_Init(void)` | Inicializa el bus, verifica el sensor, lo configura y lo pone a medir |
| `ADXL345_Status_t ADXL345_ReadAccel(ADXL345_Accel_t *accel)` | Lee los tres ejes y verifica que el sensor siga configurado |

**Por qué existe:** la aplicación habla en "aceleración en mg", no en "registros y bytes". Esta cabecera es lo único que `main.c` necesita saber del sensor.

### 2.4 `swo.h` — contrato de la salida de texto

**Incluye:** nada (no usa tipos especiales en sus prototipos).

**Qué hace:** declara cómo enviar texto al depurador por el pin SWO.

**Bloque a bloque:**

1. **Comentario de cabecera.** Explica que reemplaza a una USART, que después de `SWO_Init()` también funciona `printf`, y la configuración fija: núcleo a 16 MHz, SWO a 2 MHz, puerto 0 del ITM. Advierte que el depurador debe usar esos mismos valores.
2. **Prototipos:**

| Función | Qué hace |
| ------- | -------- |
| `void SWO_Init(void)` | Configura PB3 y los bloques de traza |
| `void SWO_PrintChar(char c)` | Envía un carácter; si no puede, lo descarta sin bloquear |
| `void SWO_PrintString(const char *str)` | Envía una cadena de texto |

`const char *` es un texto (una cadena de caracteres) que la función promete no modificar.

**Por qué existe:** separa "mostrar resultados" del driver SPI. Es un módulo independiente: no depende de ninguna otra capa.

---

## 3. Archivos en `src/`

### 3.1 `ll_spi.c` — bajo nivel (Marco)

**Incluye:** `"ll_spi.h"` (su propio contrato, que a su vez trae `<stdint.h>`).

**Qué hace:** es el **único** archivo que toca los registros del RCC, del GPIOB y del SPI2. No usa librerías del fabricante: las direcciones salen del manual de referencia RM0090.

**Bloque a bloque:**

1. **Mapa de registros (`typedef struct`).** Tres estructuras, `RCC_TypeDef`, `GPIO_TypeDef` y `SPI_TypeDef`, describen cómo están ordenados los registros en memoria. Cada campo es un `volatile uint32_t` en el orden exacto del manual, con su offset en un comentario (`/* 0x40 */`). Los huecos se rellenan con campos `RESERVED`.
2. **Direcciones base.**
   ```c
   #define SPI2 ((SPI_TypeDef *)0x40003800UL)
   ```
   Dice: "en la dirección `0x40003800` hay una estructura `SPI_TypeDef`". A partir de ahí, `SPI2->CR1` es literalmente el registro CR1 del SPI2. Lo mismo con `RCC` (`0x40023800`) y `GPIOB` (`0x40020400`).
3. **Bits de los registros.** `#define` con nombre para cada bit que se usa: `RCC_APB1ENR_SPI2EN (1UL << 14)`, `SPI_CR1_MSTR (1UL << 2)`, `SPI_SR_TXE (1UL << 1)`, etc. El código dice `SPI_CR1_CPOL` en lugar de un número mágico.
4. **Configuración de la aplicación.** Números de pin (`CS_PIN 12U`...), función alternativa (`SPI2_AF 5`), divisor del reloj (`SPI_BR_DIV16 3` → 1 MHz), tiempo de espera (`SPI_TIMEOUT 10000`) y valores de los campos GPIO. Las macros `FIELD2` y `FIELD_AFRH` ubican un valor en el campo de un pin (2 bits por pin en MODER, 4 bits por pin en AFRH).
5. **Estado interno.** `static uint8_t spi_error`: guarda el último error. Es `static`, así que nadie fuera de este archivo puede tocarlo; se consulta con `LL_SPI_GetError()`.
6. **`SPI_WaitSR(mask, expected)` (privada).** Espera a que unos bits de `SPI_SR` tengan el valor esperado, contando vueltas. Si llega a 10 000 vueltas, guarda `LL_SPI_ERR_TIMEOUT` y devuelve 0. Gracias a ella **ninguna espera puede colgar el programa**.
7. **`LL_SPI_Init()`.** En cuatro pasos:
   - **Relojes:** enciende GPIOB (bus AHB1) y SPI2 (bus APB1). Lee el registro una vez de relleno, porque el reloj tarda un par de ciclos en activarse. Después pulsa el **reset** de SPI2 para partir siempre de los valores de fábrica.
   - **Pines:** pone CS en alto **antes** de volverlo salida (si no, el sensor vería un pulso falso). Configura push-pull, AF5 en SCK/MISO/MOSI, velocidad media, pull-up en SCK (modo 3 reposa en alto) y en MISO (un sensor ausente se lee `0xFF`), y por último el modo de cada pin. Todo con el patrón leer → limpiar → escribir para no tocar PB3 (SWO).
   - **SPI2 con SPE = 0:** una sola escritura en `CR1` con SSM, SSI, MSTR, CPOL, CPHA y BR. El resto de bits queda en 0 (8 bits, MSB primero, full-duplex, sin CRC).
   - **Habilitar:** `SPE = 1` y borra el error.
8. **`LL_SPI_TransferByte(tx)`.** Espera TXE → escribe `DR` (el hardware genera los 8 pulsos) → espera RXNE → lee `DR`. Siempre lee `DR`, aunque el dato no sirva, para evitar el error de overrun.
9. **`LL_SPI_CS_Low()` / `LL_SPI_CS_High()`.** Escriben en `BSRR`: la mitad alta del registro baja el pin y la baja lo sube, en una sola escritura. `CS_High` espera antes `BSY = 0` para no cortar el último byte.
10. **`LL_SPI_GetError()`.** Devuelve `spi_error`.

**Por qué existe:** es la "capa física" que exige el enunciado. Concentra todo el conocimiento de registros en un solo lugar.

### 3.2 `drv_spi.c` — API SPI (Gilbert)

**Incluye:** `<stddef.h>` (para `NULL`), `"drv_spi.h"` (su contrato) y `"ll_spi.h"` (la capa de abajo).

**Qué hace:** convierte bytes sueltos en **tramas completas** y traduce el error del bajo nivel.

**Bloque a bloque:**

1. **`SPI_DUMMY_BYTE 0x00`.** El byte que se envía solo para generar reloj cuando se quiere leer (en SPI, leer siempre implica escribir).
2. **`SPI_Status()` (privada).** Pregunta a `LL_SPI_GetError()` y devuelve `SPI_OK` o `SPI_ERROR`. El operador `cond ? a : b` es un "si-entonces-sino" en una línea.
3. **`SPI_Init()`.** Llama a `LL_SPI_Init()`. Existe para que las capas de arriba no tengan que conocer la de abajo.
4. **`SPI_WriteRegister(cmd, value)`.** Baja CS → envía `cmd` → envía `value` → sube CS → devuelve el estado. Lo que llega por MISO se descarta con `(void)`.
5. **`SPI_ReadRegisters(cmd, buffer, length)`.** Primero valida los parámetros (puntero no nulo, al menos 1 byte). Luego baja CS → envía `cmd` → repite `length` veces "envía dummy y guarda lo recibido en `buffer[i]`" → sube CS → devuelve el estado.

**Por qué existe:** es la API del enunciado. No tiene ni un registro: se podría portar a otro microcontrolador cambiando solo `ll_spi.c`.

### 3.3 `adxl345.c` — driver del sensor (Gilbert)

**Incluye:** `<stddef.h>` (`NULL`), `"adxl345.h"` (su contrato) y `"drv_spi.h"` (la capa de abajo).

**Qué hace:** conoce el idioma del ADXL345: sus registros, el formato del byte de comando y la escala de los datos.

**Bloque a bloque:**

1. **Mapa de registros del sensor.** `DEVID` (`0x00`), `POWER_CTL` (`0x2D`), `DATA_FORMAT` (`0x31`, seguido de los 6 bytes de datos `0x32`–`0x37`) y el valor esperado de `DEVID`, `0xE5`.
2. **Bits del byte de comando.** `ADXL345_CMD_READ 0x80` (bit 7: lectura) y `ADXL345_CMD_MULTIBYTE 0x40` (bit 6: leer varios registros seguidos).
3. **Valores de configuración.** Modo medición (`0x08`), resolución completa (`0x08`), rango ±16 g (`0x03`) y `ADXL345_FORMAT_VALUE`, la combinación que se escribe y luego se verifica (`0x0B`). `ADXL345_LSB_PER_G 256` es la escala.
4. **`ADXL345_Write(reg, value)` (privada).** Llama a `SPI_WriteRegister` y traduce el resultado al estado del sensor.
5. **`ADXL345_Read(reg, buffer, length)` (privada).** Arma el comando: `0x80 | reg`, y si son varios bytes le suma el bit `0x40`. Luego llama a `SPI_ReadRegisters`.
6. **`ADXL345_ToMg(low, high)` (privada).** Une los dos bytes en un entero de 16 bits con signo (`(high << 8) | low`) y lo convierte: `raw × 1000 / 256`. Usa una variable de 32 bits en la multiplicación para que no se desborde.
7. **`ADXL345_Init()`.** Tres pasos:
   - Inicializa el SPI y lee `DEVID`. Si no es `0xE5`, devuelve `ERR_ID` (con MISO suelto se lee `0xFF`).
   - Escribe `DATA_FORMAT = 0x0B` (resolución completa, ±16 g).
   - Escribe `POWER_CTL = 0x08`: pasa de reposo a medición. Va al final, con el formato ya listo.
8. **`ADXL345_ReadAccel(accel)`.** Lee **7 bytes en una sola trama** desde `DATA_FORMAT`:
   - Byte 0: `DATA_FORMAT`. Si no es `0x0B`, el sensor se desconectó (`0xFF`) o se reinició (`0x00`) → `ERR_CONFIG`.
   - Bytes 1 a 6: X, Y, Z (byte bajo y alto de cada uno) → se convierten a mg y se guardan en `accel`.

   Leer todo en una trama garantiza que los tres ejes son de la **misma muestra**. Si hay error, `accel` no se modifica.

**Por qué existe:** separa "cómo se habla SPI" de "qué se le dice a este sensor en particular". Si se cambiara de sensor, solo se reemplaza este archivo.

### 3.4 `swo.c` — salida de texto (Marco)

**Incluye:** `<stddef.h>` (`NULL`), `<stdint.h>` y `"swo.h"`.

**Qué hace:** saca texto por el pin PB3 hacia el depurador usando los bloques de traza del núcleo, y redirige `printf` hacia ahí.

**Bloque a bloque:**

1. **Macros `REG32` y `REG8`.** Convierten una dirección en "el registro de 32 (u 8) bits que vive ahí". Es otra forma de acceder a registros, más simple que una estructura cuando son pocos y están dispersos.
2. **Registros.** Del micro: `RCC_AHB1ENR`, `GPIOB_MODER`, `GPIOB_AFRL`, `DBGMCU_CR`. Del núcleo Cortex-M4: `DEMCR`, los del TPIU (`CSPSR`, `ACPR`, `SPPR`, `FFCR`) y los del ITM (`STIM0`, `TER`, `TCR`, `LAR`).
3. **Bits y valores.** `TRCENA` (enciende la traza), `TRACE_IOEN` (asigna el pin), NRZ (formato tipo UART), la clave `0xC5ACCE55` del ITM, etc.
4. **Configuración.** PB3 con función alternativa 0 (TRACESWO), núcleo a 16 MHz, SWO a 2 MHz, tiempo de espera de 10 000 vueltas.
5. **`SWO_Init()`.** En cinco pasos:
   - PB3 como TRACESWO (solo su campo, leer → limpiar → escribir).
   - `DEMCR.TRCENA = 1`: sin esto los bloques de traza están apagados.
   - TPIU: puerto de 1 bit, NRZ, prescaler `16 MHz / 2 MHz − 1 = 7`, sin formateador.
   - `DBGMCU_CR`: pin de traza en modo asíncrono.
   - ITM: desbloquear con la clave, habilitarlo y habilitar el puerto 0.
6. **`SWO_PrintChar(c)`.** Si el ITM no está habilitado, sale. Si no, espera (con límite) a que haya espacio en la cola y escribe el carácter en el puerto 0. Nunca bloquea.
7. **`SWO_PrintString(str)`.** Recorre la cadena carácter por carácter hasta encontrar `'\0'`, el carácter que marca el final de todo texto en C.
8. **`_write(file, ptr, len)`.** `printf` arma el texto y llama a `_write` para "escribirlo en algún lado". El `syscalls.c` del IDE trae una versión *weak* (débil, de reserva); al definir la nuestra, el enlazador la prefiere y todo `printf` termina en `SWO_PrintChar`.

**Por qué existe:** el reto no permite verificar con una USART, y SWO solo necesita el depurador que ya está conectado. Es un módulo aparte porque no tiene nada que ver con SPI.

### 3.5 `main.c` — la aplicación (Gilbert y Marco)

**Incluye:** `<stdint.h>`, `<stdio.h>` (para `printf`), `"adxl345.h"` y `"swo.h"`. **No incluye nada de SPI**: la aplicación solo ve el sensor y la salida de texto.

**Qué hace:** lee el acelerómetro cada ~100 ms, imprime X, Y, Z y se recupera sola de los errores.

**Bloque a bloque:**

1. **`DELAY_LOOPS 400000`.** Cantidad de vueltas del retardo (≈ 100 ms a 16 MHz).
2. **Variables globales `volatile`.** `accel`, `lecturas`, `errores` y `estado`. Son globales y `volatile` para que el depurador las muestre en vivo con **Live Expressions**, incluso sin SWO.
3. **`Delay(loops)`.** Un ciclo vacío que solo gasta tiempo. El contador es `volatile` para que el compilador no lo elimine por "inútil".
4. **`EstadoTexto(s)`.** Traduce cada código de estado a un texto legible con un `switch` (un "según el valor, haz...").
5. **`main()`:**
   - `SWO_Init()` y un mensaje de bienvenida.
   - **Ciclo de inicio:** repite `ADXL345_Init()` hasta que devuelva OK. Cada fallo suma un error, lo imprime y espera.
   - **Ciclo infinito** (`for (;;)`, un programa embebido nunca termina): lee el sensor; si está bien, copia la muestra a `accel`, cuenta la lectura e imprime los tres ejes; si falla, cuenta el error, lo imprime y llama a `ADXL345_Init()` para recuperarse. Espera y repite.

**Por qué existe:** es la **prueba de concepto** que exige el enunciado. Demuestra que la librería funciona en un caso real, y es tan corta porque las capas de abajo esconden toda la complejidad.

---

## 4. Recorrido completo de una lectura

Qué pasa cuando `main` llama a `ADXL345_ReadAccel(&muestra)`:

```
main.c        ADXL345_ReadAccel(&muestra)
                │
adxl345.c       ├─ ADXL345_Read(0x31, data, 7)  → comando = 0x80 | 0x40 | 0x31 = 0xF1
                │     │
drv_spi.c       │     └─ SPI_ReadRegisters(0xF1, data, 7)
                │           ├─ LL_SPI_CS_Low()              CS = 0
ll_spi.c        │           ├─ LL_SPI_TransferByte(0xF1)    envía comando
                │           ├─ 7 × LL_SPI_TransferByte(0x00) recibe DATA_FORMAT, X0, X1, Y0, Y1, Z0, Z1
                │           ├─ LL_SPI_CS_High()             espera BSY = 0, CS = 1
                │           └─ devuelve SPI_OK / SPI_ERROR
                │
adxl345.c       ├─ ¿data[0] == 0x0B?  no → ERR_CONFIG
                └─ X, Y, Z → mg  →  ADXL345_OK
                │
main.c        printf("X=... Y=... Z=... mg")
                │
swo.c           └─ _write → SWO_PrintChar → ITM → TPIU → PB3 → J-Link → consola
```

En el bus: 8 bytes, 64 pulsos de reloj a 1 MHz ≈ 64 µs por lectura.

---

## 5. Reparto para la presentación

| Tema | Archivos | Autor |
| ---- | -------- | ----- |
| Estructura, capas y glosario | — | Ambos |
| Bajo nivel: registros RCC, GPIO y SPI | `ll_spi.h`, `ll_spi.c` | Marco |
| API SPI | `drv_spi.h`, `drv_spi.c` | Gilbert |
| Driver del sensor y detección de desconexión | `adxl345.h`, `adxl345.c` | Gilbert |
| Visualización por SWO | `swo.h`, `swo.c` | Marco |
| Aplicación y demo | `main.c` | Ambos |

La rúbrica pide que **ambos dominen todo el código por igual**: cada uno debería poder explicar también las partes del otro.

---

## Enlaces
[[reto_3_spi_driver|Reto 3 SPI]]
[[teoria_spi|Teoría del protocolo SPI]] · [[reto_3_spi|Enunciado del reto]]

