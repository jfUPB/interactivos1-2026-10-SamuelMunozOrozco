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

#### Para que usamos el Checksum?
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

#### Cambios finales
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


#### Cambios en el bridgeServer.js
El servidor original solo tenia esta linea para usar un solo adaptador
```js
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
```
Como por el adapter nuevo, ya necesitabamos que el bridgeServer recibiera los datos en el nuevo formato. Entonces agregamos esta nueva linea pa que funcione el nuevo adapter
```js
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitASCIIAdapter2 = require("./adapters/MicrobitASCIIAdapter2");
```




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
### Codigo actualizado del bridge
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

### Codigo actualizado del Sketch.js
```js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

class PainterTask extends FSMTask {
    constructor() {
        super();

        this.c = color(181, 157, 0);
        this.lineSize = 100;
        this.angle = 0;
        this.clickPosX = 0;
        this.clickPosY = 0;

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            noCursor();
            strokeWeight(0.75);
            background(255);
            console.log("Microbit ready to draw");
            this.rxData = {
                x: 0,
                y: 0,
                btnA: false,
                btnB: false,
                prevA: false,
                prevB: false,
                ready: false
            };
        }

        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }

        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }

        else if (ev.type === EVENTS.KEY_PRESSED) {
            this.handleKeys(ev.keyCode, ev.key);
        }

        else if (ev.type === EVENTS.KEY_RELEASED) {
            this.handleKeyRelease(ev.keyCode, ev.key);
        }

        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        this.rxData.ready = true;
        this.rxData.x = map(data.x,-2048,2047,0,width);
        this.rxData.y = map(data.y,-2048,2047,0,height);
        this.rxData.btnA = data.btnA;
        this.rxData.btnB = data.btnB;

        if (!this.rxData.btnB && this.prevB) {
            this.c = color(random(255), random(255), random(255), random(80, 100));
            console.log("B released");
        }

        this.prevA = this.rxData.btnA;
        this.prevB = this.rxData.btnB;
    }
}

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(255);
    painter = new PainterTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        painter.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        painter.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {
        painter.postEvent({
            type: EVENTS.DATA, payload: {
                x: data.x,
                y: data.y,
                btnA: data.btnA,
                btnB: data.btnB
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(painter.estado_corriendo, drawRunning);
}

function draw() {
    painter.update();
    renderer.get(painter.state)?.();
}

function drawRunning() {

    let mb = painter.rxData;

    if (!mb.ready) return;

    if (mb.btnA) {

        push();
        translate(width/2, height/2);

        let circleResolution = int(map(mb.y, 0, height, 2, 10));

        let radius = mb.x - width/2;

        let angle = TAU / circleResolution;

        if (mb.btnB) {
            fill(34,45,122,50);
        } else {
            noFill();
        }

        beginShape();

        for (let i = 0; i <= circleResolution; i++) {

            let x = cos(angle*i)*radius;
            let y = sin(angle*i)*radius;

            vertex(x,y);
        }

        endShape();

        pop();
    }
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```


## Bitácora de reflexión
<img width="1392" height="724" alt="image" src="https://github.com/user-attachments/assets/c511dc1f-51f6-4abc-932f-8fae6f1463ba" />






