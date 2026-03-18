# Unidad 5
## Bitácora de proceso de aprendizaje
### Paso 1: Del ASCII al binario — ¿Qué cambia?
Así lo teniamos en el microbit en la unidad anterior:
```py
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = "{},{},{},{}\n".format(xValue, yValue, aState,bState)
    uart.write(data)
    sleep(100)
```

Ahora reemplazamos la linea de empaquetado por:
```py
import struct
data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
```

Y nos quedaría algo así:
```py
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    uart.write(data)
    sleep(100)
```



Antes (ASCII), el micro:bit enviaba esto:
```py
"500,524,True,False\n"
```
Ahora (binario), envía esto:
```py
struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
```
* El ASCII envía texto legible
* Ahira envía bytes puros (binarios)

### ¿Qué hace struct.pack?
```py
'>2h2B'
```
Esto es un "molde" que define como se organizan los datos. Esto va a guardar 4 datos en un orden en especifico

* **> --> el orden**
  - "Big-endian" quiere decir, el byte más importante va primero

* **2h --> Los dos primeros datos**
  - Esta parte va a guardar dos números enteros (x, y)
  - Cada uno ocupa dos bytes y pueden ser positivos o negativos
  - Recordemos que esto quiere decir un punto en el canvas

* **2B --> los botones**
  - Va a guardar dos tados de un solo digito (A y B)
  - Cada uno ocupa un byte
  - Solo puede ser 0 o 1. 0 cuando no esta siendo presionado y 1 cuando lo está

En resumen, **'>2h2B'** guarda los datos en un orden determinado
1. x (2 bytes)

2. y (2 bytes)

3. botón A (1 byte)

4. botón B (1 byte)

#### Tamaño variable
* El ASCII puede variar, ejemplo:
  ```
  "500,524,True,False\n" → ~19 bytes
  ```
* El Binario es SIEMPRE **6 bytes**
  ```
  x (2) + y (2) + A (1) + B (1) = 6 bytes SIEMPRE
  ```

  #### Notas importantes
  * El binario siempre ocupa el mismo tamaño porque el "**struct.pack**" define exactamente cuantos bytes usa cada dato. 2 bytes para X y 2 para Y, y 1 byte para el botón A y 1 para el botón B
  * La forma Binaria es más eficiente, porque al usar menos bytes es más rapido

#### Preguntas bitácora

##### 1. Ventajas y desventajas del binario
Ventajas
* Usa menos bytes
* Transmite los datos más rapido
* Tiene un tamaño fijo
-
Desventajas
* No es legible por una persona
* Más difícil de depurar o hacer debug
* Necesita conocer el formato exacto

##### 2. Representación en hexadecimal
```
 xValue=500, yValue=524, aState=True, bState=False
```
```
01 F4 02 0C 01 00
```

* 01 F4 = 500
* 02 0C = 524
* 01 = A (true)
* 00 = B (false)

### El problema de sincronización — ¿Por qué necesitamos framing?
* La comunicación serial es un **flujo de bytes**, no es como si se mandaran paquetes de informacion por separado:
```
[PAQUETE 1] [PAQUETE 2] [PAQUETE 3]
```

Se manda TODO de seguido:
```
01 F4 02 0C 01 00 03 E8 01 F0 00 01 ...
```

* Si no llega un dato, o llega uno de más o erroneo, la lectura del sistema se daña por completo
* Lo que puede pasar es que se interpreten valores de dos paquetes distintos como si fueran uno solo y dar valores absurdo como:

Lo que se lee mal
```
F4 02 0C 01 00 03
```
resulta en:
* Valores absurdos:
  - 3073
  - 513

#### ¿Por qué pasa esto?
* No hay inicio de paqete
* No hay fin de paquete
* No hay separadores



  











## Bitácora de aplicación 


## Bitácora de reflexión
