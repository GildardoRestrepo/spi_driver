---
title: Reto Unidad 3 - Protocolos de Comunicaciones Seriales
created: 2026-10-06
time: 06:50pm
creator: Gilbert
last update: 2026-10-06
update by: Gilbert
type: referencia
status: activo
fase: ""
area: microcontroladores
editor: Gilbert
order: 1
tags:
  - tipo/referencia
---

# Reto Unidad 3 - Protocolos de Comunicaciones Seriales

> [!success] Resumen
> Enunciado oficial del reto: driver Bare-Metal (sin HAL/LL del fabricante) separado en bajo nivel y API, prueba de concepto física, repositorio documentado y sustentación en clase.


**Curso:** Microprocesadores | **Institución:** Universidad Pontificia Bolivariana

El presente documento establece los lineamientos técnicos y académicos para el desarrollo, entrega y sustentación del proyecto de la Unidad 3. El objetivo fundamental es que los estudiantes dominen el hardware interno del microcontrolador mediante el diseño de controladores (_drivers_) desde el nivel de registros hasta la implementación de una interfaz de programación de aplicaciones (API) funcional.

---
## 1. Dinámica de Trabajo y Asignación de Temas

El proyecto se desarrollará estrictamente en **parejas**. Cada equipo asumirá el rol de ingenieros de software embebido para diseñar un _driver_ Bare-Metal enfocado en uno de los siguientes submódulos del microcontrolador:

- **SPI (Serial Peripheral Interface):** Enfoque en comunicación de alta velocidad full-duplex.
- **I2C (Inter-Integrated Circuit):** Enfoque en topología de bus direccionable y control de pines Open-Drain.
- **CAN (Controller Area Network):** Enfoque en tramas diferenciales orientadas a la industria automotriz y automatización robusta.
- **ADC con DMA (Analog-to-Digital Converter + Direct Memory Access):** Enfoque en la captura asíncrona de datos de alta frecuencia sin intervención de la CPU.

---
## 2. Arquitectura del Software (Metodología HAL)

Se prohíbe el uso de librerías de fabricantes (como STM32Cube HAL o LL). Los estudiantes construirán su propia **Capa de Abstracción de Hardware (HAL)**. El concepto de HAL consiste en separar la lógica de negocio de los detalles físicos del microcontrolador, permitiendo que el código de la aplicación principal sea portable y fácil de leer.

El proyecto debe dividirse en archivos independientes (`.c` y `.h`) respetando la siguiente jerarquía:

- **Funciones de Bajo Nivel (Capa Física):** Encargadas de la manipulación directa de los registros del microcontrolador (ej. configurar el _baud rate_, habilitar relojes en el bus AHB/APB, gestionar banderas de estado).
- **API de Alto Nivel (Capa Lógica):** Funciones públicas que el usuario final del _driver_ invocará. Deben tener firmas (prototipos) claras y autoexplicativas (ej. `I2C_WriteData(address, buffer, length)` o `SPI_TransmitReceive(data)`), aislando al programador de la aplicación principal de la complejidad de los registros.
- **Modularidad:** Los archivos deben tener nombres coherentes con el módulo (ej. `drv_spi.c`, `drv_spi.h`).

---
## 3. Prueba de Concepto (PoC)

El _driver_ no puede quedarse en un ejercicio teórico. Cada equipo debe implementar una Prueba de Concepto física que valide su funcionamiento en un escenario real de ingeniería. Ejemplos de implementación esperada:

- **SPI:** Leer o escribir datos en una memoria externa, leer datos de un sensor o controlar una pantalla matricial.
- **I2C:** Interfaz con un sensor ambiental (temperatura/humedad), acelerómetro o un _display_ OLED.
- **CAN:** Establecer un nodo de comunicación entre dos microcontroladores distintos transmitiendo telemetría.
- **ADC + DMA:** Muestreo continuo de una señal analógica (ej. un sensor de audio u otro sensor analógico) enviando los bloques de memoria directamente a un búfer para su posterior transmisión.

---
## 4. Entregables: Repositorio GitHub y Documentación

La gestión del código se evaluará con estándares profesionales. El equipo debe entregar el enlace a un repositorio público en GitHub que contenga:

- El código fuente estructurado en carpetas lógicas (`/src`, `/inc`, `/docs`).
- Un archivo **[README.md](http://README.md)** riguroso que incluya:
    - Descripción del módulo y protocolo desarrollado.
    - Diagrama de conexión de hardware (esquemático de la Prueba de Concepto).
    - Guía de uso de la API (explicación de las funciones de alto nivel disponibles).
    - Instrucciones de compilación.

---
## 5. Sustentación y Demostración en Clase

La evaluación final consistirá en una defensa técnica presencial. Durante la presentación, los estudiantes deberán demostrar:

1. **Dominio del Hardware:** Explicar el funcionamiento interno del módulo del microcontrolador. Esto incluye la arquitectura del bus, los modos de operación, y el análisis detallado de los registros críticos (control, estado y datos) involucrados en su _driver_.
2. **Análisis de Bajo Nivel:** Exponer cómo implementaron las secuencias de inicialización y las operaciones de lectura/escritura a nivel de bits.
3. **Demostración en Vivo (Live Demo):** Poner a funcionar la Prueba de Concepto frente a la clase, demostrando la estabilidad del código y el correcto intercambio de información.

---
## Evaluación

|**Criterio**|**Ponderación**|**Excelente (4.5 - 5.0)**|**Aceptable (3.0 - 4.4)**|**Insuficiente (0.0 - 2.9)**|
|---|---|---|---|---|
|**1. Arquitectura de Software y Capa HAL**|**25%**|El _driver_ es totalmente Bare-Metal (sin librerías de fabricante). Implementa una separación estricta entre bajo nivel (registros) y alto nivel (API). Código modular en archivos `.c` y `.h` independientes, limpio, documentado y reutilizable.|Cumple con la implementación Bare-Metal, pero la separación entre el bajo nivel y la API presenta acoplamientos puntuales. Estructura de archivos funcional pero mejorable en modulación o nombres de funciones.|Dependencia directa de librerías HAL/LL del fabricante, mezcla de lógica de aplicación con manipulación de registros en un solo archivo, o código no modular e ilegible.|
|**2. Prueba de Concepto (PoC) e Integración Física**|**25%**|La Prueba de Concepto funciona de manera impecable en hardware real (sensor, pantalla, bus CAN o DMA). El intercambio de datos es continuo, correcto y robusto ante condiciones límite de comunicación.|La Prueba de Concepto funciona en condiciones normales, pero presenta fallas de comunicación esporádicas, bloqueo de bus por no limpiar banderas de estado o falta de manejo de errores básicos.|La Prueba de Concepto no funciona, no logra transmitir/recibir datos en el bus físico, o el código se bloquea de forma indefinida durante la ejecución.|
|**3. Dominio del Hardware y Registros**|**20%**|La pareja explica con solidez el funcionamiento interno del módulo (buses AHB/APB, relojes, modos de operación) y detalla bit a bit el rol de los registros de control, estado y datos utilizados.|Demuestran comprensión general del módulo y los registros principales, pero dudan al explicar la configuración de bits específicos o las secuencias de inicialización del reloj/periférico.|Muestran desconocimiento sobre cómo funciona el hardware internamente; no pueden explicar qué hacen los registros modificados en su propio código.|
|**4. Sustentación y Demostración en Vivo (Live Demo)**|**20%**|La demostración en vivo es fluida y exitosa. Ambos integrantes dominan el código por igual y responden con precisión técnica a las preguntas de modificación de registros en tiempo real.|La demostración en vivo funciona pero requiere ajustes de último minuto. Participación desigual entre los integrantes o respuestas parciales a las preguntas técnicas del docente.|La demostración en vivo falla por errores de configuración no justificados. Uno o ambos integrantes son incapaces de responder sobre la autoría y funcionamiento del código.|
|**5. Repositorio GitHub y Documentación**|**10%**|Repositorio organizado en carpetas (`/src`, `/inc`, `/docs`). `README.md` impecable con diagramas esquemáticos, guía completa de uso de la API y un historial de _commits_ que evidencia trabajo en equipo equilibrado.|Repositorio funcional pero desorganizado. El `README.md` incluye la información básica pero carece de diagramas de conexión claros o detalles completos de la API. _Commits_ concentrados en una sola persona.|Repositorio incompleto, sin archivo `README.md`, sin diagramas de hardware o entregado fuera de la plataforma especificada.|

---
## Enlaces
[[reto_3_spi_driver|Reto 3 SPI]]
[[teoria_spi|Teoría del protocolo SPI]]
