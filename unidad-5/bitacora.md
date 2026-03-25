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

### Paso 2: El problema de sincronización — ¿Por qué necesitamos framing?

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

#### Entonces que es framing?
Es en pocas palabras poner orden al desorden
* Tiene un inicio claro del paquete
* Un fin claro
* Validacion (checksum)

#### Diferencias entre ASCII y Framing

##### ASCII
Ejemplo:
```
"500,524,True,False\n"
```
* Tiene \n, pa saber donde termina
* Puede esperar una linea completa
* Si algo falla, descarta la linea

##### Binario (sin framing)
ejemplo: 
```
01 F4 02 0C 01 00 ...
```
* No hay separadores
* No sabe donde empieza
* No sabe donde terina

#### Preguntas bitacora

##### 1. ¿Por qué el protocolo ASCII de la unidad anterior no tenía este problema de sincronización? (Pista: piensa en qué rol cumplía el carácter \n.)
* Porque usaba usaba el caracter **\n** para determinar el fin de un paquete, lo que permitia acumular los datos de la linea antes de procesarlos

##### 2. ¿Por qué en binario no podemos usar \n como delimitador?
* Porque, por ejemplo, **\n** en Binario es **0x0A**, y el sistema lo puede confundir con datos reales


### Paso 3: El protocolo binario final con framing
<img width="680" height="451" alt="image" src="https://github.com/user-attachments/assets/fcb110db-c378-4dc5-a571-56f8387b7950" />

El paquete ahora tiene la siguiente estructura:
```
[HEADER][DATOS][CHECKSUM]
```

#### Que hace cada parte?

##### 1. Header (0xAA)
* Marca el inicio del paquete
* Si lees en el lugar incorrecto del paquete, descarta los bytes hasta que encuentre **0xAA** 

##### 2. Datos (6 bytes)
* X → 2 bytes
* Y → 2 bytes
* A → 1 byte
* B → 1 byte

##### 3. Checksum
```
checksum = sum(data) % 256
```
* La suma final de los datos
* Reduce el valor entre 0-225
* Marca el final del paquete

#### Codigo Explicado (Lo importante)

##### Qué hace realmente esta línea
```py
data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
```
* Son los datos puros, los 6 bytes
* No incluye ni el header ni el checksum

##### El Checksum
```py
checksum = sum(data) % 256
```
* Aqui suma los bytes reales, no numeros grandes
* Entonces suma cada byte individual

Ejemplo:
```
01 F4 02 0C 01 00
```
Pasa a ser esto
```
1 + 244 + 2 + 12 + 1 + 0 = 260
```
Lo que hace el checksum:
```
260 % 256 = 4
```
checksum = 4

##### Esta es la construcción final
```py
packet = b'\xAA' + data + bytes([checksum])
```
Aqui se arma el paquete completo de datos
* Inicio
* Datos
* Validacionm
```
[AA] [01 F4 02 0C 01 00] [04]
```

#### Codigo completo del Microbit
```py
from microbit import *
import struct

uart.init(115200)
display.set_pixel(0, 0, 9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = button_a.is_pressed()
    bState = button_b.is_pressed()
    data = struct.pack('>2h2B', xValue, yValue, int(aState), int(bState))
    checksum = sum(data) % 256
    packet = b'\xAA' + data + bytes([checksum])
    uart.write(packet)
    sleep(100)
```

#### Preguntas bitacora

##### ¿Cuántos bytes tiene el paquete completo con framing? ¿Cuántos más que sin framing?
* Con framing tiene 8 bytes, porque se le suma el del header y el checksum. Sin framing solo 6, porque serian los datos

##### ¿Qué pasa si un byte de datos tiene el valor 0xAA (170 en decimal)? ¿Podría el receptor confundirlo con un header? ¿Cómo ayuda el checksum en este caso?
* El receptor solo interpreta **0xAA** como header cuando esta en el inicio del paquete. Entonces si hay un falso header en los datos, el checksum no coincidiria y deja ver el error, descartando asi el paquete corrupto




## Bitácora de aplicación 

Tenemos que adaptar otra vez el sistema que ya tenemos a un nuevo hardware sin dañar la estructura base del sistema. Este nuevo hardware lo que hace es traducir los datos en binario  mandados por el microbit

### Que teniamos que hacer
* Copiamos y pegamos el adapter que ya teniamos en uno nuevo llamado "MicrobitBinaryAdapter"

### Codigo original del adapter
```js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

function parseCsvLine(line) {
  console.log("Data arrives");
  const values = line.trim().split(",");
  if (values.length !== 4) throw new ParseError(`Expected 4 values, got ${values.length}`);

  const x = Number(values[0]);
  const y = Number(values[1]);
  const btnA = String(values[2]).trim().toLowerCase();
  const btnB = String(values[3]).trim().toLowerCase();

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" };
}


class MicrobitAsciiAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    this.buf += chunk.toString("utf8");

    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {
        const parsed = parseCsvLine(line);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          if (this.verbose) console.log("Bad data:", e.message, "raw:", line);
        } else {
          this._fail(e);
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitAsciiAdapter;

```

### Codigo adaptado a binario
```js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = Buffer.alloc(0); // Buffer binario
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }

    this.port = null;
    this.buf = Buffer.alloc(0); // limpiar buffer correctamente
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    // 1. Acumular bytes
    this.buf = Buffer.concat([this.buf, chunk]);

    // 2. Procesar mientras haya suficiente información
    while (this.buf.length >= 8) {

      // 3. Buscar header (0xAA)
      if (this.buf[0] !== 0xAA) {
        this.buf = this.buf.slice(1);
        continue;
      }

      // 4. Verificar tamaño mínimo
      if (this.buf.length < 8) {
        return;
      }

      // 5. Extraer paquete
      const packet = this.buf.slice(0, 8);

      // 6. Calcular checksum
      const data = packet.slice(1, 7);
      let calcChk = 0;
      for (let i = 0; i < data.length; i++) {
        calcChk += data[i];
      }
      calcChk = calcChk % 256;

      const recvChk = packet[7];

      // 7. Validar checksum
      if (calcChk !== recvChk) {
        if (this.verbose) {
          console.warn("Checksum incorrecto, descartando paquete:", packet);
        }
        this.buf = this.buf.slice(1);
        continue;
      }

      // 8. Parsear datos
      const x = packet.readInt16BE(1);
      const y = packet.readInt16BE(3);
      const btnA = packet[5] === 1;
      const btnB = packet[6] === 1;

      // 9. Emitir datos
      this.onData?.({ x, y, btnA, btnB });

      // 10. Eliminar paquete procesado
      this.buf = this.buf.slice(8);
    }

    // 11. Evitar crecimiento infinito
    if (this.buf.length > 4096) {
      this.buf = Buffer.alloc(0);
    }
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitBinaryAdapter;
```

#### Que se quito en el codigo

##### 1. Eliminamos la siguiente funcion
```js
function parseCsvLine(line) {
  console.log("Data arrives");
  const values = line.trim().split(",");
  if (values.length !== 4) throw new ParseError(`Expected 4 values, got ${values.length}`);

  const x = Number(values[0]);
  const y = Number(values[1]);
  const btnA = String(values[2]).trim().toLowerCase();
  const btnB = String(values[3]).trim().toLowerCase();

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" };
}
```

* Esto lo que hacia era convertir el texto mandado por el microbit: "500,524,true,false" en { x, y, btnA, btnB }

* La eliminamos porque lo que hace es procesar datos en formato ASCII se parado por comas, en este nuevo adapter, los datos no estan en ese formato, por lo que ya no nos sirve esta funcion

##### 2. Eliminamos la siguiente linea en _onChunk
```js
this.buf += chunk.toString("utf8");
```
* Lo que hace es convertir bytes a texto
* Como binario no es texto, no lo necesitamos


##### 3. Eliminamos el siguiente bloque
```js
let idx;
while ((idx = this.buf.indexOf("\n")) >= 0) {
  const line = this.buf.slice(0, idx).trim();
  this.buf = this.buf.slice(idx + 1);

  if (!line) continue;

  try {
    const parsed = parseCsvLine(line);
    this.onData?.(parsed);
  } catch (e) {
    if (e instanceof ParseError) {
      if (this.verbose) console.log("Bad data:", e.message, "raw:", line);
    } else {
      this._fail(e);
    }
  }
}
```

* Lo que hace es esperar las lineas que tengan al final "\n" lo que significa que es donde la linea termina
* Lo eliminamos porque solo funciona en ASCII y en binario no existe "\n"
* Ahora usamos "**0xAA (header)**"


#### Que se cambio en el codigo

##### 1. Cambiamos el buffer
Antes:
```js
this.buf = "";
```
* El "buffer" es una **memoria temporal** donde se guardan los datos que llegan. Y tiene ese "**="";**" ya que se trabaja con texto (**ASCII**)
* Ejemplo: "500,524,true,false\n". Esto es un **string**
-

Despues:
```js
this.buf = Buffer.alloc(0);
```
* Se crea un "buffer" vacio, por eso el "(0)", porque aun no tiene datos
* Cuando llegan datos: "this.buf = Buffer.concat([this.buf, chunk]);". Ahora si el buffer crece
* Cada vez que se hace "this.buf = Buffer.alloc(0);". Se borran los datos viejos 


##### Ahora si agregamos la logica del framing en _onChunk
```js
_onChunk(chunk) {
    // 1. Acumular bytes
    this.buf = Buffer.concat([this.buf, chunk]);

    // 2. Procesar mientras haya suficiente información
    while (this.buf.length >= 8) {

      // 3. Buscar header (0xAA)
      if (this.buf[0] !== 0xAA) {
        this.buf = this.buf.slice(1);
        continue;
      }

      // 4. Verificar tamaño mínimo
      if (this.buf.length < 8) {
        return;
      }

      // 5. Extraer paquete
      const packet = this.buf.slice(0, 8);

      // 6. Calcular checksum
      const data = packet.slice(1, 7);
      let calcChk = 0;
      for (let i = 0; i < data.length; i++) {
        calcChk += data[i];
      }
      calcChk = calcChk % 256;

      const recvChk = packet[7];

      // 7. Validar checksum
      if (calcChk !== recvChk) {
        if (this.verbose) {
          console.warn("Checksum incorrecto, descartando paquete:", packet);
        }
        this.buf = this.buf.slice(1);
        continue;
      }

      // 8. Parsear datos
      const x = packet.readInt16BE(1);
      const y = packet.readInt16BE(3);
      const btnA = packet[5] === 1;
      const btnB = packet[6] === 1;

      // 9. Emitir datos
      this.onData?.({ x, y, btnA, btnB });

      // 10. Eliminar paquete procesado
      this.buf = this.buf.slice(8);
    }

    // 11. Evitar crecimiento infinito
    if (this.buf.length > 4096) {
      this.buf = Buffer.alloc(0);
    }
  }
```
Que hace el codigo

###### 1. Acumular los bytes
```js
this.buf = Buffer.concat([this.buf, chunk]);
```
* Si tienes ya una parte de los bytes y legan otros, los junta en un solo paquete
Ejemplo:

Antes:
```
buf = [AA 01]
chunk = [F4 02]
```

Despues:
```
buf = [AA 01 F4 02]
```

###### 2. Procesar si hay suficiente informacion
```js
while (this.buf.length >= 8)
```
* Solo funciona si tiene 8 bytes, o sea, un paquete completo


###### 3. Busca el header (0xAA)
```js
if (this.buf[0] !== 0xAA) {
  this.buf = this.buf.slice(1);
  continue;
}
```
* Verifica si el primer byte es el inicio del paquete
* Si NO es 0xAA, elimina ese byte:
```js
this.buf = this.buf.slice(1);
```
* Esto se hace por si los datos que llegan esta desalineado, por si el primer byte que se lee no es el header


###### 4. Verifica el tamaño
```js
if (this.buf.length < 8) {
  return;
}
```
* Si no tiene los 8 bytes, pues espera a que se completen
* Es una medida de seguridad


###### 5. Extraer paquete
```js
const packet = this.buf.slice(0, 8);
```
* Toma los primeros 8 bytes como un paquete


###### 6. Calcular el checksum
```js
const data = packet.slice(1, 7);
let calcChk = 0;
for (let i = 0; i < data.length; i++) {
  calcChk += data[i];
}
calcChk = calcChk % 256;
```
-
```js
const data = packet.slice(1, 7);
```
* Toma una parte del paquete, o sea, los datos que van al checksum , del x al B

-
```js
let calcChk = 0;
```
* Inicializa el checksum en cero

-
```js
for (let i = 0; i < data.length; i++) {
  calcChk += data[i];
}
```
* Recorre cada byte y lo suma
* Reduce el numero a un valor entre 0 y 255
* Ejemplo:
  - 260 % 256 = 4
  - Porque el checksum 1 byte

###### 7. validar el checksum
```js
if (calcChk !== recvChk) {
  if (this.verbose) {
    console.warn("Checksum incorrecto, descartando paquete:", packet);
  }
  this.buf = this.buf.slice(1);
  continue;
}
```
1. Compara el checksum que calculamos y el que venia en el paquete
```js
if (calcChk !== recvChk)
```

2. Muestra un mensaje en consola si hay error
```js
if (this.verbose) {
  console.warn("Checksum incorrecto, descartando paquete:", packet);
}
```

3. Elimina solo el primer byte, por si el header no esta al frente del paquete
```js
this.buf = this.buf.slice(1);
```

4. Salta al siguiente ciclo while. Lo que hace es ingnorar el paquete que este mal y revisa el siguiente
```js
continue;
```


###### 8. Parsear datos
```js
const x = packet.readInt16BE(1);
const y = packet.readInt16BE(3);
const btnA = packet[5] === 1;
const btnB = packet[6] === 1;
```

1. Lee el valor de X desde el paquete
   * Empieza en el byte 1
   * Lee 2 bytes
   * Los interpreta como numero con signo
   * Usamos "readInt16BE(1)" ya que X ocupa 2 bytes, no 1
```js
const x = packet.readInt16BE(1);
```

2. Lee el valor de Y desde el paquete
   * Empieza en 3 porque:
     ```
     [1-2] → X  
     [3-4] → Y  
     ```
     *  Usamos "readInt16BE(1)" ya que Y ocupa 2 bytes, no 1
```js
const y = packet.readInt16BE(3);
```

3. Lee el estado del boton A
```js
const btnA = packet[5] === 1;
```

4. Lee el estado del boton B
```js
const btnB = packet[6] === 1;
```


###### 9. Emitir datos
```js
this.onData?.({ x, y, btnA, btnB });
```
* Envia los datos al sistema

###### 10. Eliminar paquete procesado
```js
this.buf = this.buf.slice(8);
```
* Elimina el paquete que ya fue procesado. Una medida de seguridad pa que no se lea el mismo otra vez


###### 11. Evitar crecimiento infinito
```js
if (this.buf.length > 4096) {
  this.buf = Buffer.alloc(0);
}
```
* Limpia el buffer si crece demasiado

 
#### Codigo del bridgeServer
```js

//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

//   WS contract:
//    * bridge To client:
//        {type:"status", state:"ready|connected|disconnected|error", detail:"..."}
//        {type:"microbit", x:int, y:int, btnA:bool, btnB:bool, t:ms}
//    * client To bridge:
//        {cmd:"connect"} | {cmd:"disconnect"}
//        {cmd:"setSimHz", hz:30}
//        {cmd:"setLed", x:2, y:3, value:9}


const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitAscii2Adapter = require("./adapters/MicrobitASCIIAdapter2");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

// const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");

const log = {
  info: (...args) => console.log(`[${new Date().toISOString()}] [INFO]`, ...args),
  warn: (...args) => console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),
  error: (...args) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};


function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function hasFlag(name) {
  return process.argv.includes(`--${name}`);
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try {
    return JSON.parse(s);

  } catch (e) {
    log.warn("Failed to parse JSON: ", s, e);
    return null;
  }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const DEVICE = (getArg("device", "sim") || "sim").toLowerCase();
const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const SERIAL_PATH = getArg("serialPort", null);
const BAUD = parseInt(getArg("baud", "115200"), 10);
const SIM_HZ = parseInt(getArg("hz", "30"), 10);
const VERBOSE = hasFlag("verbose");

async function findMicrobitPort() {
  const ports = await SerialPort.list();
  const microbit = ports.find(p =>
    p.vendorId && parseInt(p.vendorId, 16) === 0x0D28
  );
  return microbit?.path ?? null;
}

async function createAdapter() {
  if (DEVICE === "microbit") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit found at ${path}`);
    return new MicrobitAsciiAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbit2") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit 2 found at ${path}`);
    return new MicrobitAscii2Adapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "microbitBinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit binary found at ${path}`);
    return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  // if (DEVICE === "microbit-bin") {
  //   const path = SERIAL_PATH ?? await findMicrobitPort();
  //   if (!path) {
  //     log.error("micro:bit not found. Use --serialPort to specify manually.");
  //     process.exit(1);
  //   }
  //   return new MicrobitBinaryAdapter({ path, baud: BAUD });
  // }

  return new SimAdapter({ hz: SIM_HZ });
}



async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  adapter.onConnected = (detail) => {
    log.info(`[ADAPTER] Device Connected: ${detail}`);
    status(wss, "connected", detail);
  };

  adapter.onDisconnected = (detail) => {
    log.warn(`[ADAPTER] Device Disconnected: ${detail}`);
    status(wss, "disconnected", detail);
  };

  adapter.onError = (detail) => {
    log.error(`[ADAPTER] Device Error: ${detail}`);
    status(wss, "error", detail);
  };

  adapter.onData = (d) => {
    broadcast(wss, {
      type: "microbit",
      x: d.x,
      y: d.y,
      btnA: !!d.btnA,
      btnB: !!d.btnB,
      t: nowMs(),
    });
  };

  status(wss, "ready", `bridge up (${DEVICE})`);

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Remote Client connected from ${req.socket.remoteAddress}. Total clients: ${wss.clients.size}`);

    const state = adapter.connected ? "connected" : "ready";

    const detail = adapter.connected
      ? adapter.getConnectionDetail()
      : `bridge (${DEVICE})`;

    ws.send(JSON.stringify({ type: "status", state, detail, t: nowMs() }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;

      if (msg.cmd === "connect") {
        log.info(`[NETWORK] Client requested adapter connect`);

        if (adapter.connected) {
          log.info(`[HW-POLICY] Adapter already open. Sending current status to incoming client.`);
          ws.send(JSON.stringify({ type: "status", state: "connected", detail: adapter.getConnectionDetail(), t: nowMs() }));
          return;
        }
        
        try {
          await adapter.connect();
        } catch (e) {
          const detail = `connect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "disconnect") {
        log.info(`[NETWORK] Client requested adapter disconnect`);
        if (wss.clients.size > 1) {
          log.info(`[HW-POLICY] Adapater kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
          ws.send(JSON.stringify({ type: "status", state: "disconnected", detail: "logical disconnect only", t: nowMs() }));
          return;
        }
        
        try {
          await adapter.disconnect();
        } catch (e) {
          const detail = `disconnect failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }

      if (msg.cmd === "setSimHz" && adapter instanceof SimAdapter) {
        log.info(`Setting Sim Hz to ${msg.hz}`);
        await adapter.handleCommand(msg);
        status(wss, "connected", `sim hz=${adapter.hz}`);
        return;
      }

      if (msg.cmd === "setLed") {
        try {
          await adapter.handleCommand?.(msg);
        } catch (e) {
          const detail = `command failed: ${e.message || e}`;
          log.error(`[ADAPTER] ` + detail);
          status(wss, "error", detail);
        }
        return;
      }
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Remote Client disconnected. Total clients left: ${wss.clients.size}`);
      if (wss.clients.size === 0) {
        log.info("[HW-POLICY] No more remote clients. Auto-disconnecting adapter device to free resources...");
        adapter.disconnect();
      }
    });
  });

  if (DEVICE === "sim") {
    await adapter.connect();
  }
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```

##### Que se le puso al codigo

###### 1. Importar el nuevo adapter
```js
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
```
* Le hace saber al Node que hay un nuevo adapter

###### 2. Agregamos el caso en "createAdapter()"
```js
if (DEVICE === "microbitbinary") {
  const path = SERIAL_PATH ?? await findMicrobitPort();
  if (!path) {
    log.error("micro:bit not found. Use --serialPort to specify manually.");
    process.exit(1);
  }
  log.info(`micro:bit binary found at ${path}`);
  return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
}
```
* Esta parte es para decidir que adapter usar, dependiendo de los datos que lleguen del microbit


#### Prueba de que siguen funcionando los otros Adapters

##### MicrobitAsciiAdapter (Parte Importante)
```js
function parseCsvLine(line) {
  console.log("Data arrives");
  const values = line.trim().split(",");
  if (values.length !== 4) throw new ParseError(`Expected 4 values, got ${values.length}`);

  const x = Number(values[0]);
  const y = Number(values[1]);
  const btnA = String(values[2]).trim().toLowerCase();
  const btnB = String(values[3]).trim().toLowerCase();

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" };
}
```

###### Codigo microbit
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
    sleep(100) # Envia datos a 10 Hz
```

###### Prueba de que se detecta el microbit
<img width="718" height="106" alt="image" src="https://github.com/user-attachments/assets/83327dad-e50a-4304-b041-1098b42f9109" />

###### Prueba de que se conecta al server y funciona
<img width="1911" height="1142" alt="image" src="https://github.com/user-attachments/assets/2543377d-d16b-4617-9ec5-46b7c762c096" />

##### MicrobitAscii2Adapter (Parte importante)
```js
function parseCsvLine(line) {

  const values = line.trim().split("|");
  if (values.length !== 6) throw new ParseError(`Expected 6 values, got ${values.length}`);

  const x = Number(values[1].split(":")[1]);
  const y = Number(values[2].split(":")[1]);
  const btnA = Number(values[3].split(":")[1]);
  const btnB = Number(values[4].split(":")[1]);
  const chk = Number(values[5].split(":")[1]);

  const calcChk = Math.abs(x) + Math.abs(y) + btnA + btnB;
  if(calcChk !== chk)
  {
    throw new ParseError("Checksum no coincide");
  }

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  

  return { x: x | 0, y: y | 0, btnA: btnA === 1, btnB: btnB === 1 };
}
```

##### Codigo Microbit
```py
from microbit import *

uart.init(115200)
display.set_pixel(0,0,9)

while True:
    xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0

    t = running_time()

    chk = abs(xValue) + abs(yValue) + aState + bState
    
    data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, xValue, yValue, aState, bState, chk)
    uart.write(data)
    sleep(100) # Envia datos a 10 Hz
```

###### Prueba de que se conecta al Adapter
<img width="932" height="101" alt="image" src="https://github.com/user-attachments/assets/8d585b67-a0f5-475a-88bf-13dd6608d8d3" />

###### Prueba de que se conecta al server y funciona
<img width="1919" height="1087" alt="image" src="https://github.com/user-attachments/assets/67102c72-35a2-451f-acf3-5fc6605941f8" />







## Bitácora de reflexión
