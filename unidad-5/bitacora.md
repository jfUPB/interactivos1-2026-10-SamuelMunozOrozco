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

* > --> el orden
  - "Big-endian" quiere decir, el byte más importante va primero

* 2h --> Los dos primeros datos
  - Esta parte va a guardar dos números enteros (x, y)
  - Cada uno ocupa dos bytes y pueden ser positivos o negativos
  - Recordemos que esto quiere decir un punto en el canvas

* 2B --> los botones
  - Va a guardar dos tados de un solo digito (A y B)
  - Cada uno ocupa un byte
  - Solo puede ser 0 o 1. 0 cuando no esta siendo presionado y 1 cuando lo está

En resumen, '>2h2B' **hola**










## Bitácora de aplicación 


## Bitácora de reflexión
