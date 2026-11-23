# Práctica 3: Controlador de Máquina Expendedora

**Sistemas Empotrados y de Tiempo Real** · Sara Casado Cuesta · URJC


## 1. Introducción

Para realizar la práctica he desarrollado el controlador de una **máquina expendedora** con un **Arduino UNO**. El usuario podrá interactuar con ella a través de una **pantalla LCD**, un **joystick** y un **botón**. La máquina monitorizará la **temperatura y humedad**, gracias al sensor DHT11 y detectará la **presencia del cliente** con un sensor de ultrasonidos (HC-SR04). Además, dispone de dos **LEDs** que indicarán el estado del sistema.

El sistema se estructura principalmente con tres modos de operación:
- **Arranque**: el LED1 parpadea 3 veces mientras la pantalla muestra "CARGANDO...".
- **Servicio**: detecta al cliente, le muestra la temperatura y humedad, le deja elegir un producto y simula su preparación.
- **Admin**: permite al técnico consultar los sensores en tiempo real, ver el tiempo que lleva encendida la máquina y modificar los precios.

Todo el programa está hecho sin `delay()`, usando hilos (ArduinoThread), interrupciones y un watchdog, para que la máquina responda siempre al instante.


## 2. Montaje del circuito

fotooooooooooooooo circuito

foto circuitooooooooooo

*Figura 1. Montaje de la máquina expendedora sobre la protoboard.*

### 2.1. Componentes

Todos los componentes utilizados pertenecen al kit proporcionado por los laboratorios:
- Arduino UNO
- Pantalla LCD 16x2 (modo 4 bits)
- Módulo joystick (eje X + eje Y + pulsador)
- Sensor de temperatura y humedad DHT11 (3 pines)
- Sensor de ultrasonidos HC-SR04
- Pulsador externo con resistencia pull-up de 10 kΩ
- 2 LEDs (LED1 azul y LED2 verde) con resistencias de 220 Ω
- Resistencia de 2 kΩ (contraste del LCD) y de 220 Ω (retroiluminación)

### 2.2. Conexiones

| Componente | Pin del componente | Pin Arduino |
|---|---|---|
| LED1 (azul) | Ánodo (+) (220 Ω) · Cátodo (−) a GND | D6 |
| LED2 (verde) | Ánodo (+) (220 Ω) · Cátodo (−) a GND | D5 |
| Botón externo | Un lado (con pull-up de 10 kΩ a 5V) · Otro lado a GND | D2 |
| Joystick | VRx / VRy | A0 / A1 |
| Joystick | SW | D3 |
| HC-SR04 | Trig / Echo | A2 / A3 |
| DHT11 | Data | D4 |
| LCD | RS / E | D12 / D11 |
| LCD | D4 / D5 / D6 / D7 | D10 / D9 / D8 / D7 |
| LCD | RW | GND |
| LCD | V0 (contraste) | GND vía 2 kΩ |
| LCD | A / K (retroiluminación) | 5V vía 220 Ω / GND |

NOTAS: 
* El botón va en el pin 2 porque es uno de los dos pines con interrupción hardware del Arduino UNO.
* El LED2 va en el pin 5 porque necesita PWM para aumentar su brillo progresivamente durante la preparación. Este pin pertenece al Timer0, así que no entra en conflicto con TimerOne, que usa el Timer1 (PWM de los pines 9 y 10). 
* Los pines 9 y 10 se usan para el LCD, pero solo como salidas digitales, sin PWM, por lo que tampoco hay conflicto.
* El ultrasonidos usa A2 y A3 como pines digitales, porque los pines digitales libres estaban llenos.
* El LCD funciona en modo 4 bits: la pantalla tiene 8 pines de datos (D0–D7), pero puede recibir cada byte en dos mitades de 4 bits usando solo D4–D7. Así se ahorran 4 pines del Arduino. La librería `LiquidCrystal` activa este modo automáticamente al recibir solo 4 pines de datos. Es algo más lento, pero para mostrar texto no se aprecia.


## 3. Fritzing

El esquema se ha realizado con Fritzing, replicando el montaje real.

fotooooooooooo Fritzing

*Figura 2. Esquema de conexiones realizado en Fritzing.*

* En Fritzing se ha usado la pieza RHT03 para el sensor de temperatura y humedad porque no existe el módulo DHT11 de 3 pines, en el montaje real solo se conectan VCC, Data y GND.


## 4. Descripción de la solución

### 4.1. Máquina de estados

El comportamiento de la máquina se organiza como una máquina de estados (`enum Estado` + `switch`), que ejecuta el hilo `hiloMaquinaEstados` cada 10 ms. En cada ejecución entra al `case` del estado actual y comprueba si se cumple la condición para pasar al siguiente. El texto de cada pantalla se muestra solo en el momento del cambio de estado, no en cada ejecución, de esta manera el LCD no parpadeará.

#### Modo ARRANQUE y modo SERVICIO

| Estado | Qué muestra el LCD | Pasa a | Cuando |
|---|---|---|---|
| ARRANQUE | CARGANDO ... | ESPERANDO | El LED1 termina sus 3 parpadeos |
| ESPERANDO | ESPERANDO CLIENTE | INFO | La distancia es menor de 1 m |
| INFO | Temperatura y humedad | MENU | Pasan 5 s |
| MENU | Producto y precio | PREPARANDO | Se pulsa el botón del joystick |
| PREPARANDO | Preparando + nombre del producto | RETIRAR | Pasa el tiempo aleatorio (4–8 s) |
| RETIRAR | RETIRE BEBIDA | ESPERANDO | Pasan 3 s |

- Cancelación: desde INFO, MENU, PREPARANDO o RETIRAR, una pulsación de 2–3 s del botón externo vuelve a ESPERANDO, sin reiniciar la placa.

- Modo Admin: con una pulsación de ≥ 5 s se entra desde cualquier estado, y con otra igual se sale, volviendo a ESPERANDO.


#### Modo ADMIN

| Estado | Qué muestra el LCD | Pasa a | Cuando |
|---|---|---|---|
| ADMIN_MENU | MENU ADMIN + opción marcada | ADMIN_TEMP / DIST / CONTADOR / PRECIOS | Se pulsa el joystick sobre esa opción |
| ADMIN_TEMP | Temperatura y humedad en vivo | ADMIN_MENU | Joystick a la izquierda |
| ADMIN_DIST | Distancia en vivo | ADMIN_MENU | Joystick a la izquierda |
| ADMIN_CONTADOR | Segundos desde el encendido | ADMIN_MENU | Joystick a la izquierda |
| ADMIN_PRECIOS | Producto y precio | ADMIN_EDITAR_PRECIO / ADMIN_MENU | Pulsar el joystick / Joystick a la izquierda |
| ADMIN_EDITAR_PRECIO | Producto y precio nuevo | ADMIN_PRECIOS | Pulsar (guarda) / izquierda (descarta) |

* Las pantallas en vivo (temperatura, distancia y contador) se repintan cada 500 ms con `setCursor` en lugar de `clear()`, para que no parpadeen. Al modificar un precio se trabaja sobre una copia: si se confirma, se guarda en `precios[]` y si se descarta, el original no cambia. Los cambios se ven en el modo Servicio.


#### CONTROLES

| Acción | En el menú de productos | En el modo Admin |
|---|---|---|
| Joystick arriba / abajo | Cambiar de producto | Cambiar de opción / subir o bajar el precio 0.05 |
| Pulsar joystick | Seleccionar producto | Entrar en la opción / confirmar el precio |
| Joystick izquierda | — | Volver atrás / descartar el precio |
| Botón 2–3 s | Cancelar y volver a ESPERANDO | — |
| Botón ≥ 5 s | Entrar en Admin | Salir de Admin |


### 4.2. Threads (ArduinoThread)

Todas las tareas periódicas se organizan en hilos con la librería ArduinoThread y se gestionan por un `ThreadController` que se ejecuta en cada vuelta del `loop()`. El programa tiene cuatro hilos:

- `hiloLed` (clase propia `LedParpadeoThread`, cada 1 s): hace parpadear el LED1 en el arranque. Cuenta los parpadeos y, al llegar a 3, se desactiva solo con `enabled = false`.
- `hiloDistancia` (clase propia `DistanciaThread`, cada 200 ms): mide la distancia con el HC-SR04 y guarda la última medida.
- `hiloDHT` (`Thread` con `onRun()`, cada 2 s): lee la temperatura y la humedad del DHT11 (no admite lecturas más rápidas).
- `hiloMaquinaEstados` (`Thread` con `onRun()`, cada 10 ms): ejecuta la máquina de estados, lee el joystick, actualiza el LCD y los LEDs.

En cuanto al `ThreadController`, en el `setup()` se le añaden los hilos con `controller.add()` y en el `loop()` con `controller.run()` los revisa uno a uno ejecutando solo los que ya han cumplido su intervalo.

Además he utilizado dos formas de crear hilos. 

- Clases que heredan de `Thread` para el LED y la distancia, porque tienen datos propios (pines, número de parpadeos, última distancia). Los datos son atributos `private`, y desde fuera solo se consultan con los métodos públicos (`haTerminado()`, `getDistanciaCm()`). Además, siempre se llama a `Thread::run()` para que la librería registre la ejecución y respete el intervalo establecido.
- `onRun()` con una función, para los hilos más sencillos.

Los hilos de sensores solo miden y guardan el valor. Es la máquina de estados la que escribe en el LCD, por ello, nunca hay dos partes del programa escribiendo en la pantalla a la vez. Por ejemplo, la distancia se mide cada 200 ms, pero en la pantalla de Admin se muestra cada 500 ms, para que se lea bien.

El Arduino UNO tiene un único núcleo, así que el controlador ejecuta los hilos uno detrás de otro cuando les toca. Si un hilo tarda, retrasa a los demás y si se bloquea, se bloquearía todo el programa.


### 4.3. Interrupciones

He utilizado dos tipos de interrupciones, una hardware para el botón externo y otra software con TimerOne para el contador.

#### Botón externo (Hardware)

El botón está conectado al pin 2, que es uno de los pines con interrupción del Arduino UNO, y se configura con `attachInterrupt(digitalPinToInterrupt(PIN_BOTON), isrBoton, CHANGE)`. Uso el modo `CHANGE` para que la interrupción salte tanto al pulsar como al soltar el botón. Así, puedo medir cuánto ha durado la pulsación y distinguir entre:

- 2 - 3 s: cancelar y volver al inicio del modo Servicio.
- ≥ 5 s: entrar o salir del modo Admin.

La ISR (`isrBoton`) apunta el instante del cambio con `millis()` y activa un aviso (`hay_flanco_boton`). Todo lo demás se procesa fuera de la interrupción, en la función `procesarBoton()`, que se llama en cada vuelta del `loop()`.

Como el contacto del botón rebota al pulsarlo y genera varias interrupciones seguidas, solo acepto el cambio cuando el pin lleva 50 ms estable (antirrebote), en ese momento leo su valor real con `digitalRead`. Al pulsar guardo el instante, y al soltar calculo la duración.

#### Contador con TimerOne (Software)

Para el contador del modo Admin uso la librería TimerOne, que genera una interrupción cada segundo con el Timer1 del microcontrolador. Su ISR (`isrContador`) solo suma 1 a la variable `segundos_encendido`, así el contador avanza de forma exacta, segundo a segundo, independientemente de lo que esté haciendo el resto del programa.

#### Variables compartidas con las ISR

Las variables que modifican las ISR (`hay_flanco_boton`, `t_flanco_boton` y `segundos_encendido`) están declaradas como `volatile`, para que el compilador las lea siempre de memoria y no use una copia antigua.

El Arduino UNO lee la memoria de byte en byte, y un `unsigned long` ocupa 4 bytes, así que leerlo entero le lleva varios pasos. Si una interrupción cambiara la variable a mitad de la lectura, el valor leído sería incorrecto. Para evitarlo, cuando el programa principal lee estas variables, desactiva un momento las interrupciones con `noInterrupts()`, copia el valor y las vuelve a activar con `interrupts()` (sección crítica).

Dentro de las propias ISR no hace falta, porque mientras se ejecuta una interrupción el Arduino no deja que salte otra.


### 4.4. Watchdog

El watchdog es un temporizador interno del microcontrolador que reinicia la placa si el programa se queda bloqueado.

- En el `setup()`, primero lo desactivo con `wdt_disable()` (por si viene activo de un reinicio anterior) y después lo activo con `wdt_enable(WDTO_2S)`.
- En el `loop()`, llamo a `wdt_reset()` en cada vuelta para indicar que el programa sigue funcionando.

He elegido un tiempo de 2 segundos para dejar margen ya que la operación más lenta del programa (`pulseIn`) puede tardar hasta 1 s, y así el watchdog solo reinicia la placa si realmente se ha bloqueado.

Para comprobar que funciona, forcé un bloqueo con un bucle infinito (`while (true) { }`). En la pantalla del contador el programa dejó de llamar a `wdt_reset()` y la placa se reinició sola a los 2 segundos. He dejado esa línea comentada en el código para poder repetir la prueba.


## 5. Conclusiones

Como resultado he conseguido programar una máquina expendedora que cumple con todos los requisitos y responde al instante sin bloquearse. Organizarla con una máquina de estados ha hecho que el código fuera fácil de ampliar con nuevos modos o pantallas.