# Unidad 4

## Bitácora de proceso de aprendizaje

### Explicacion Diagrama

<img width="1280" height="960" alt="image" src="https://github.com/user-attachments/assets/157323a3-0d09-4716-a12d-0fc02d856a78" />


#### HTTP SERVER
* El servidor web que entrega los archivos al navegador, o sea, cuando se abre la pagina en el navegador, este servidor envia los archivos del proyecto

##### index.html
Es la pagina principal

Funciones
* Carga la pagina web
* Importa los scripts
* Inicia la aplicacion

##### fsm.js
Maquina de estados finitos
* Es un sistema donde el programa tiene los estados del proyecto, como "esperando", "conectado", "corriendo"

##### Sketch.js
* Este es el aechivo principal de p5.js
* Aqui se ejecuta la logica del arte generativo
* Se ejecuta el setup y el draw

#### Visuales p5.js (BROWSER)
* Lo del "BROWSER" significa que esta es la parte que corre en el navegador
* Aqui es donde se ejecuta el arte generativo
Aqui e hacen dos cosas importantes

##### updateLogic
Actualiza la logica del programa
Ejemplo:
* Leer sensores
* Cambiar variables
* Procesar datos

##### drawRunnig
* Se encarga de dibujar en pamntalla

##### bridgeClient.js
* Permite que el navegador se comunique con el servidor (Ya que el navegador no puede hablar directamente con el "hardware")

#### Server Bridge (bridgeServer.js)
* Actua como puente entre el navegador y el hardware
* Recibe los comandos del navegador, lee los datos del "hardware" y envia los datos al navegador

#### Adapters
* Es una capa que traduce la comunicacion del hardware, o sea, convierte los datos al formato que usa el sistema

##### MicrobitASCIIAdapter.js
* Lee los datos del puerto serial
* Interpreta el protocolo del microbit

##### SimAdapter.js
* Conecta con el simulador de microbit

##### BaseAdapter.js
* Es la clase base de los adapters, o sea, todos los adapters heredan de esta clase

##### Microbit
* Aqui es donde se encuentra el "hardware" real
* Envia los datos por "serial (UART)"

##### Simulador Microbit
* La version virtual del microbit


## Bitácora de aplicación 


### Que pedia el ejercicio
Nos pedian integrar un nuevo hardware con un sistema software ya hecho y no podiamos modificarlo. El nuevo hardware usa un protocolo diferente de comunicacion serial, por lo que teniamos que crear un adapter nuevo que interpretara correctmente los datos

### flujo del sistema
1. El microbit envia los datos del acelerometro y los botones por el puerto serial
2. El adapter nuevo recibe estos datos y los interpreta pa convertirlos en un objeto JSON
3. El bridgeServer envia el objeto al navegador mediante WebSocket
4. El bridgeClient recibe los datos en el navegador
5. Por ultimo el sketch.js usa esos datos para dibujar en el canvas y asi empezar el sistema interactivo

### Codigo Adapter original (Solo la parte importante)
```js
function parseCsvLine(line) {
  const values = line.trim().split(",");
  if (values.length !== 4) throw new ParseError(`Expected 6 values, got ${values.length}`);

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
#### Funcion del Adapter original
El adapter se encarga de interpretar los datos enviados por el microbit y los transforma al formato estandar que el sistema tiene, de esa forma no tener problemas de compatibilidad entre el servidor y la aplicacion web.

Este adapter recibia los datos del microbit que venian en un formato separado por comas. Ejemplo: -200,50,true,false

La siguiente linea era la encargada de separar los valores por medio de las comas
```js
const values = line.trim().split(",");
```
El "trim()" se encargaba de eliminar saltos de linea o espacios y "split(",")" de dividir la cadena usando la coma. Esto convertia la linea de valores en un arreglo

Ejemplo:

-200,50,true,false

values[0] = -200 --> x

values[1] = 50 --> y

values[2] = "true" --> btnA

values[3] = "false" --> btnB

Luego, con la siguiente parte, se convertian esos valores a los tipos de datos correspondientes
```js
const x = Number(values[0]);
const y = Number(values[1]);
const btnA = String(values[2]).trim().toLowerCase();
const btnB = String(values[3]).trim().toLowerCase();
```

Y finalmente se genera el objeto que el sistema utiliza. Lo que el bridgeServer envia al navegador
```js
{ x, y, btnA, btnB }
```

### Codigo Adapter copia (la parte importante)
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
#### Que le cambiamos al adapter copia 
En el ejercicio o contrato, se nos pidio que se cambiara la forma de separar los valores, ya no por ",", sino por "|". Ya que el microbit ahora envia un fomato diferente:

$T:tiempo|X:acel_x|Y:acel_y|A:estado_a|B:estado_b|CHK:checksum\n


Este es el diccionario de datos:

T: Timestamp en milisegundos desde el arranque del dispositivo (entero).

X, Y: Valores del acelerómetro (enteros entre -2048 y 2047).

A, B: Estado de los botones, 1 presionado, 0 liberado.

CHK: Checksum calculable. Es un número entero de 3 dígitos que representa la suma de los valores absolutos de X, Y, A y B.

Ahora tendriaos un nuevo valor que seria T, tambien se cambiaria el tipo de valor en btnA y btnB, ya no serian "String" sino "Number". Y por ultimo agregariamos un Checksum, cual funcion seria de sumar las variables y verificar si coincidian o no. Si no coincidian, significaba que la trama era corrupata y el sistema debia ignorarla pero sin actualizar la vista o el canvas, pero debia registrar un mensaje de advertencia en la consola

#### Cambios en el codigo
```js
const values = line.trim().split("|");
```
Esto separa los valores con la linea |, y nos quedaria asi:

values[0] = "$T:45020"

values[1] = "X:-245"

values[2] = "Y:12"

values[3] = "A:1"

values[4] = "B:0"

values[5] = "CHK:258"



##### Por que usamos values[1].split(":")[1]?
En el nuevo protocolo los valores vienen con una etiqueta

Ejemplo: x:-245

Y solo necesitamos el numero para asi realizar el Checksum. Si solo usamos "values[1]" solo tendriamos "x:-245"

Por eso al usar "values[1].split(":")[1]". Se separa la parte de "x:" de la parte "-245". Y por eso al final ponemos [1] pa quedarnos finalmente con ese valor

##### Para que usamos el Checksum?
Lo usamos como una medida de seguridad para verificar si los datos enviados por el microbit son correctos y no han sido corrompidos.

Pueden pasar como ruido en la comunicacion, datos incompletos o caracteres corruptos

Ejemplo:

Lo que se espera: X:-245|Y:12|A:1|B:0

Lo que llega: X:-245|Y:1Z|A:1|B:0

Si esto el sistema no lo detecta, pues el adapter no va a traducir los dats correctos y dañar el sistema

##### Como funciona el Checksum desde el microbit
Codigo completo del Microbit:
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
En esta parte el microbit lee los sensores:
```py
xValue = accelerometer.get_x()
    yValue = accelerometer.get_y()
    aState = 1 if button_a.is_pressed() else 0
    bState = 1 if button_b.is_pressed() else 0
```
Ejemplo de los valores:

xValue = -245

yValue = 12

aState = 1

bState = 0

Luego se hace el Checksum, en el microbit. ("abs()" es valor absoluto)
```py
chk = abs(xValue) + abs(yValue) + aState + bState
```
se veria asi el calculo:

chk = |x| + |y| + A + B

Y la suma da: 258

Luego se construye el mensaje y se envia por el puerto serial
```py
  data = "$T:{}|X:{}|Y:{}|A:{}|B:{}|CHK:{}\n".format(t, xValue, yValue, aState, bState, chk)
    uart.write(data)
    sleep(100) # Envia datos a 10 Hz
```
Llegan de esta forma: $T:45020|X:-245|Y:12|A:1|B:0|CHK:258

Ya luego, el adapter transforma los datos, extrae los valores y verifica el Checksum, haciendo otro Checksum, pero con los valores de los datos que llegaron 
```js
const calcChk = Math.abs(x) + Math.abs(y) + btnA + btnB;
```

Cunaod hace la operacion, lo compara con el Checksum que llego y si no es lo mismo, muestra el error en la consola
```js
if (calcChk !== chk) {
    throw new ParseError("Checksum mismatch");
}
```

###### Cambios finales
En el adapter original teniamos las siguientes lineas:
```js
 if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" };
```
La primer parte comprueba que los botones solo tienen dos valores validos, usando texto ya que ese era el protocolo original

En el nuevo adapter borramos la primera parte, porque como el nuevo protocolo ya no manda string, sino un Number, pues no la necesitamos, lo que solo nos deja cambiar, el return:
```js
return { x: x | 0, y: y | 0, btnA: btnA === 1, btnB: btnB === 1 };
```
Aqui convierte el formato al estandar del nuevo protocolo, que es usar 1 y 0, luego envia los datos al sistema


#### Codigo de Microbit
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

#### Codigo de Adapter Copia
```js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

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


class MicrobitAsciiAdapter2 extends BaseAdapter {
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

module.exports = MicrobitAsciiAdapter2;

```

#### Codigo Adapter original
```js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

function parseCsvLine(line) {
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


## Bitácora de reflexión

