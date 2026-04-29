# Unidad 7

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Proposito
* En esta actividad, como actividades anteriores, debiamos integrar una nueva fuente de datos, con la unica diferencia que esta vez, tendriamos mas de una fuente enviando datos al mismo tiempo. Strudel, enviando datos constantes para activar las visuales y OpenStateControl para alterar o cambiar estas visuales en tiempo real, por medio de widgets

### Codigo

#### OpenStageControlAdapter.js
* Primero creamos un adapter nuevo que reciba los datos del OpenStage, que venian en el siguiente formato JSON
```
{
  "address": "/rgb_1",
  "args": [255, 0, 0]
}
```
* Luego ya el adapter lo pasa al siguiente formato que el sistema puede procesar. Esto se hace para que el sistema distinga entre tipos de mensajes
```
{
  "type": "osc",
  "payload": {
    "address": "/rgb_1",
    "args": [255, 0, 0]
  }
}
```

<br><br>

A continuacione ste es el codigo compelto del adapter nuevo
```
const osc = require("osc");
const BaseAdapter = require("./BaseAdapter");

class OpenStageControlAdapter extends BaseAdapter {

  constructor({ port = 9000 } = {}) {
    super();

    this.port = port;
    this.udpPort = null;
  }

  async connect() {

    if (this.connected) return;

    this.udpPort = new osc.UDPPort({
      localAddress: "0.0.0.0",
      localPort: this.port
    });

    this.udpPort.on("message", (msg) => {

      const normalized = {
        type: "osc",

        payload: {
          address: msg.address,
          args: msg.args || []
        }
      };

      this.onData?.(normalized);
    });

    this.udpPort.open();

    this.connected = true;

    this.onConnected?.(`OSC listening on ${this.port}`);
  }

  async disconnect() {

    if (!this.connected) return;

    this.udpPort?.close();

    this.connected = false;

    this.onDisconnected?.("OSC stopped");
  }

  getConnectionDetail() {
    return `OSC UDP ${this.port}`;
  }
}

module.exports = OpenStageControlAdapter;
```

##### Codigo Explicado (Lo importante)

Importamos las librerias: 
```
const osc = require("osc");
```
* Importamos las librerias encargadas de trabajar con mesajes OSC mediante UDP (UPD es un protocolo de comunicacion de red usado para enviar datos rapidamente entre aplicaciones,OpenStageControl, envia mensajes OSC usando UDP)
* Esta libreria permite abrir puertos
* Pemrite escuchar mensajes
* Permite recibir datos de OpenStageControl

<br><br>

```
const osc = require("osc");
const BaseAdapter = require("./BaseAdapter");
```
* osc permite escuchar mensajes OSC por UDP
* BaseAdapter integra este adapter a la arquitectura general del sistema

<br><br>

```
constructor({ port = 9000 } = {}) {
  super();

  this.port = port;
  this.udpPort = null;
}
```
* El constructor
  - Guarda el puerto UDP
  - Prepara la variable que maneja la conexion OSC
  - El puerto 9000 tiene que coincidir con la configuracion en OpenStageControl

<br><br>

```
this.udpPort = new osc.UDPPort({
  localAddress: "0.0.0.0",
  localPort: this.port
});
```
* Se crea el puerto UDP
  - Aui es donde OpgenStage se conecta al sistema y el adapter empieza a recibir los datos de OSC

<br><br>

```
this.udpPort.on("message", (msg) => {
```
* Recibidor de mensajes
  - Se ejecuta cada que llega un dato tipo OSC

<br><br>

```
const normalized = {
  type: "osc",

  payload: {
    address: msg.address,
    args: msg.args || []
  }
};
```
* Normalizacion de los datos
  - Convierte los mensajes OSC a el formato que lee el sistema

Ejemplo de Antes:
```
{
  "address": "/size",
  "args": [700]
}
```

Ejemplo de despues:
```
{
  "type": "osc",
  "payload": {
    "address": "/size",
    "args": [700]
  }
}
```

<br><br>

```
this.onData?.(normalized);
```
* Envia los datos al bridge
  - Entrega el mensaje normalizado al bridgeServer

<br><br>

```
this.udpPort.open();
```
* Abre la conexion
  - Aca el sistema ya puede recibir los datos OSC


#### Cambios en el bridgeServer

```
//   Uso:
//     node bridgeServer.js --device sim --wsPort 8081 --hz 30
//     node bridgeServer.js --device microbit --wsPort 8081 --serialPort COM5 --baud 115200

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");
const SimAdapter = require("./adapters/SimAdapter");
const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const MicrobitAscii2Adapter = require("./adapters/MicrobitASCIIAdapter2");
const MicrobitBinaryAdapter = require("./adapters/MicrobitBinaryAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");



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

  if (DEVICE === "microbitbinary") {
    const path = SERIAL_PATH ?? await findMicrobitPort();
    if (!path) {
      log.error("micro:bit not found. Use --serialPort to specify manually.");
      process.exit(1);
    }
    log.info(`micro:bit binary found at ${path}`);
    return new MicrobitBinaryAdapter({ path, baud: BAUD, verbose: VERBOSE });
  }

  if (DEVICE === "strudel") {
    log.info("Using Strudel adapter");
    return new StrudelAdapter({ verbose: VERBOSE });
  }

  return new SimAdapter({ hz: SIM_HZ });
}

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT} device=${DEVICE}`);

  const adapter = await createAdapter();

  // 🔵 NUEVO: múltiples adapters
const strudelAdapter = new StrudelAdapter({ verbose: VERBOSE });
const oscAdapter = new OpenStageControlAdapter({ port: 9000 });

// conectar ambos
await strudelAdapter.connect();
await oscAdapter.connect();

  //===============Nuevo servidor para Strudel================
const wssStrudel = new WebSocketServer({ port: 8080 });

log.info(`Strudel WS listening on ws://127.0.0.1:8080`);

wssStrudel.on("connection", (ws) => {
  log.info("[STRUDEL] Connected");

  ws.on("message", (raw) => {
    const msg = safeJsonParse(raw.toString("utf8"));
    if (!msg) return;

    // 🔥 pasar al adapter (NO lógica aquí)
    strudelAdapter.handleIncoming(msg);
  });

  ws.on("close", () => {
    log.info("[STRUDEL] Disconnected");
  });
});

  // =========================================================


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

  // 🔵 STRUDEL
strudelAdapter.onData = (data) => {
  broadcast(wss, data);
};

// 🎛 OSC
oscAdapter.onData = (data) => {
  broadcast(wss, data);
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

      // 🔥 STRUDEL (BIEN UBICADO)
      if (DEVICE === "strudel" && msg.address === "/dirt/play") {
        adapter.handleIncoming(msg);
        return;
      }

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
          log.info(`[HW-POLICY] Adapter kept open. Shared with ${wss.clients.size - 1} other active client(s).`);
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

<br><br>

```
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");
```
* Importamos el nuevo adapter

<br><br>

```
// 🔵 NUEVO: múltiples adapters
const strudelAdapter = new StrudelAdapter({ verbose: VERBOSE });
const oscAdapter = new OpenStageControlAdapter({ port: 9000 });

// conectar ambos
await strudelAdapter.connect();
await oscAdapter.connect();
```
* Crea mos multiples adapter
  - Asi el sistema ya no trabaja solo con una fuente de datos, ahora puede recibir el de Strudel y OSC

<br><br>

```
wssStrudel.on("connection", (ws) => {

  ws.on("message", (raw) => {

    const msg = safeJsonParse(raw.toString("utf8"));

    if (!msg) return;

    strudelAdapter.handleIncoming(msg);
  });
});
```
* Recepcion de mensajes de strudel
  - Recibe los mensajes enviados por Strudel
  - Lo convierte el texto a JSON
  - Los entrega al StrudelAdpater

<br><br>

```'
// 🔵 STRUDEL
strudelAdapter.onData = (data) => {
  broadcast(wss, data);
};

// 🎛 OSC
oscAdapter.onData = (data) => {
  broadcast(wss, data);
};
```
* Envio de datos al frontend
  - EL bridge recibe los mensajes ya normalizados y los pasa al frontend mediante EbSocket


#### bridgeClient.js

```
class BridgeClient {
  constructor(url = "ws://127.0.0.1:8081") {
    this._url = url;
    this._ws = null;
    this._isOpen = false;

    this._onData = null;
    this._onConnect = null;
    this._onDisconnect = null;
    this._onStatus = null;
  }

  get isOpen() {
    return this._isOpen;
  }

  onData(callback) { this._onData = callback; }
  onConnect(callback) { this._onConnect = callback; }
  onDisconnect(callback) { this._onDisconnect = callback; }
  onStatus(callback) { this._onStatus = callback; }

  open() {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) {
      if (!this._isOpen) this.send({ cmd: "connect" });
      return;
    }

    if (this._ws) {
      this.close();
    }

    this._ws = new WebSocket(this._url);

    this._ws.onopen = () => {
      this.send({ cmd: "connect" });
    };

    this._ws.onmessage = (event) => {
      // Esperamos JSON normalizado desde el bridge
      let msg;
      try {
        msg = JSON.parse(event.data);
      } catch (e) {
        console.warn("WS message is not JSON:", event.data);
        return;
      }

      // Convención mínima:
      // - {type:"status", state:"...", detail:"..."}
      // - {type:"microbit", x:..., y:..., btnA:..., btnB:...}
      if (msg.type === "status") {
        this._onStatus?.(msg);

        if (msg.state === "connected") {
          this._isOpen = true;
          this._onConnect?.();
        }

        if (msg.state === "disconnected" || msg.state === "error" || msg.state === "ready") {
          this._isOpen = false; 
          this._onDisconnect?.();
          if (msg.state === "error") {
            this._ws?.close();
            this._ws = null;
          }          
        }
        return;
       }

       // 🔵 microbit
       if (msg.type === "microbit") {
         this._onData?.(msg);
         return;
        }

        // 🟣 strudel
        if (msg.type === "strudel") {
         this._onData?.(msg);
         return;
        }

          // 🎛 OSC (NUEVO)
          if (msg.type === "osc") {
           this._onData?.(msg);
           return;
          }

      };

    this._ws.onerror = (err) => {
      console.warn("WS error:", err);
    };

    this._ws.onclose = () => {
      this._handleDisconnect();
    };
  }

  close() {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;

    try {
      this.send({ cmd: "disconnect" });
      this._isOpen = false;
    } catch (e) {
      console.warn("Failed to send disconnect command:", e);
    }
  }

  send(obj) {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._ws.send(JSON.stringify(obj));
  }

  _handleDisconnect() {
    this._isOpen = false;
    this._ws = null;
    this._onDisconnect?.();
  }
}

```

<br><br>

```
// 🔵 microbit
if (msg.type === "microbit") {
  this._onData?.(msg);
  return;
}

// 🟣 strudel
if (msg.type === "strudel") {
  this._onData?.(msg);
  return;
}

// 🎛 OSC (NUEVO)
if (msg.type === "osc") {
  this._onData?.(msg);
  return;
}
```
* Esta es la parte que permite que el frontend pueda distinguir entre diferentes tipos de datos
  - Por eso es tan importante el "type" ya que se encarga de detectar los eventos que mandan los datos

<br><br>

```
this._onData?.(msg);
```
* Se encraga de enviar los mensajes al FSM, udateLogic y a la capa de estados 


#### FSM y udateLogic (StrudelSketch.js)

```
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA"
};

class StrudelTask extends FSMTask {
   constructor() {
      super();

      this.eventQueue = [];
      this.activeAnimations = [];
      this.LATENCY_CORRECTION = 0;

       // 🎛 NUEVO: estado persistente
      this.controlState = {};

      this.transitionTo(this.estado_corriendo);
    }

    estado_corriendo = (ev) => {

        if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }
    };

    updateLogic(data) {

    // 🎵 EVENTOS STRUDEL (TIEMPO)
    if (data.type === "strudel") {
        this.eventQueue.push({
            timestamp: data.timestamp,
            sound: data.payload.s,
            delta: data.payload.delta
        });

        this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
    }

    // 🎛 CONTROLES OSC (ESTADO PERSISTENTE)
    if (data.type === "osc") {
        const addr = data.payload.address;
        const args = data.payload.args;

        // clave dinámica (sin slash)
        const key = addr.replace("/", "");

        // guarda automático (genérico)
        this.controlState[key] = args.length === 1 ? args[0] : args;
    }
}
}

let task;
let bridge;

function setup() {
    createCanvas(windowWidth, windowHeight);
    rectMode(CENTER);
    noStroke();

    task = new StrudelTask();
    bridge = new BridgeClient();

   bridge.onData((data) => {
  console.log("DATA RECIBIDA:", data);

  // 🎵 Strudel + 🎛 OSC
  if (data.type === "strudel" || data.type === "osc") {
    task.postEvent({
      type: EVENTS.DATA,
      payload: data
    });
  }
});

    bridge.open(); // auto conectar
}

function draw() {

    // 🔥 SIEMPRE actualizar FSM y eventos
    task.update();

    // 🎛 Toggle OFF
    if (task.controlState.toggle !== undefined && task.controlState.toggle === 0) {
        background(0);
        return;
    }

    background(0, 30);

    let now = Date.now() + task.LATENCY_CORRECTION;

    // scheduling (igual que original)
    while (task.eventQueue.length > 0 && now >= task.eventQueue[0].timestamp) {

        let ev = task.eventQueue.shift();

        task.activeAnimations.push({
            startTime: ev.timestamp,
            duration: ev.delta * 1000,
            type: ev.sound,
            x: random(width * 0.2, width * 0.8),
            y: random(height * 0.2, height * 0.8),
            color: getColorForSound(ev.sound)
        });
    }

    // render
    for (let i = task.activeAnimations.length - 1; i >= 0; i--) {

        let anim = task.activeAnimations[i];

        let elapsed = now - anim.startTime;
        let progress = elapsed / anim.duration;

        if (progress <= 1.0) {

            dibujarElemento(anim, progress);

        } else {

            task.activeAnimations.splice(i, 1);

        }
    }
}

// ---------------- VISUALES (IGUAL QUE TU CÓDIGO) ----------------

function dibujarElemento(anim, p) {
    push();
    const color = anim.color;

    switch (anim.type) {
        case 'tr909bd':
            dibujarBombo(p, color);
            break;

        case 'tr909sd':
            dibujarCaja(p, color);
            break;

        case 'tr909hh':
        case 'tr909oh':
            dibujarHat(anim, p, color);
            break;

        default:
            dibujarDefault(anim, p, color);
            break;
    }
    pop();
}

function dibujarBombo(p, c) {

    // 🎛 tamaño controlado por OSC
    let maxSize = task.controlState.size || 600;

    let d = lerp(100, maxSize, p);
    let alpha = lerp(255, 0, p);

    fill(c[0], c[1], c[2], alpha);

    circle(width / 2, height / 2, d);
}

function dibujarCaja(p, c) {
    let w = lerp(width, 0, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
    let sz = lerp(40, 0, p);
    fill(c[0], c[1], c[2]);
    rect(anim.x, anim.y, sz, sz);
}

function dibujarDefault(anim, p, c) {
    let size = lerp(100, 0, p);
    let angle = p * TWO_PI;

    translate(anim.x, anim.y);
    rotate(angle);

    stroke(c[0], c[1], c[2]);
    strokeWeight(2);
    noFill();

    rect(0, 0, size, size);
    line(-size, 0, size, 0);
    line(0, -size, 0, size);

    noStroke();
    fill(255, 150);
    textSize(20);
    text(anim.type, 10, 10);
}

function getColorForSound(s) {

    // 🎛 OSC controla el color del bombo
    if (s === "tr909bd" && task.controlState.rgb_1) {
        return task.controlState.rgb_1.map(v => Number(v));
    }

    const colors = {
        'tr909sd': [0, 200, 255],
        'tr909hh': [255, 255, 0],
        'tr909oh': [255, 150, 0]
    };

    if (colors[s]) return colors[s];

    let charCode = s.charCodeAt(0) || 0;
    let r = (charCode * 123) % 255;
    let g = (charCode * 456) % 255;
    let b = (charCode * 789) % 255;
    return [r, g, b];
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

<br><br>

```
constructor() {
    super();

    this.eventQueue = [];
    this.activeAnimations = [];
    this.LATENCY_CORRECTION = 0;

    // 🎛 estado persistente OSC
    this.controlState = {};

    this.transitionTo(this.estado_corriendo);
}
```
* Creamos el controlState
  - Antes el sistema solo manejaba eventos temporales
  - Ahora se agrega un espacio para guardar los controles de OSC
  - Se creo como controState, porque antes ya existia un state y si quedaban con el mismo nombre, generaba conflicto

<br><br>

```
updateLogic(data) {

    // 🎵 EVENTOS STRUDEL
    if (data.type === "strudel") {

        this.eventQueue.push({
            timestamp: data.timestamp,
            sound: data.payload.s,
            delta: data.payload.delta
        });

        this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
    }

    // 🎛 CONTROLES OSC
    if (data.type === "osc") {

        const addr = data.payload.address;
        const args = data.payload.args;

        const key = addr.replace("/", "");

        this.controlState[key] =
            args.length === 1 ? args[0] : args;
    }
}
```
* Modificamos el udateLogic

```
if (data.type === "osc")
```
* Procesa parametros enviados desde OpenStageControl, los cuales permanecen activos por toda la sesion (tambien depende de la configuracion, pero por lo general son asi)

```
const key = addr.replace("/", "");
```
* Extrae la direccion
  - Transforma /rgb_1 en rgb_1, para poder usarlo como clave dentro de controlSate

```
this.controlState[key] =
    args.length === 1 ? args[0] : args;
```
* Es un tipo de guardad
  - Permite que el sistema soporte varios tipos de widgets sin crear logica para cada uno de ellos

#### Integracion de bridge.onData()

```
bridge.onData((data) => {

    console.log("DATA RECIBIDA:", data);

    // 🎵 Strudel + 🎛 OSC
    if (data.type === "strudel" || data.type === "osc") {

        task.postEvent({
            type: EVENTS.DATA,
            payload: data
        });
    }
});
```
* Este bloque conecta el bridgeClient con el FSM del sistema

```
bridge.onData((data) => {
```
* Este callback se ejecuta automaticamente cuando llegan mensajes del bridgeServer
  - Los datos ya vienen normalizados
  - CLasificados por tipo y listos para ser procesados

```
console.log("DATA RECIBIDA:", data);
```
* Se usa para validar la conexion
  - Permite verificar si los mensajes llegan correctamente
  - Que Strudel y OSC funcionen juntos

```
if (data.type === "strudel" || data.type === "osc")
```
* Este condicional permite aceptar ambos tipos de datos

```
task.postEvent({
    type: EVENTS.DATA,
    payload: data
});
```
* Envia los mensajes al FSM


#### Integracion de los widgets de OSC

##### Cambio de color (RGB)
<img width="298" height="300" alt="image" src="https://github.com/user-attachments/assets/0fae2e84-7a12-4c62-a4ac-9da18de2a3fc" />


```
function getColorForSound(s) {

    // 🎛 OSC controla el color del bombo
    if (s === "tr909bd" && task.controlState.rgb_1) {
        return task.controlState.rgb_1.map(v => Number(v));
    }

    const colors = {
        'tr909sd': [0, 200, 255],
        'tr909hh': [255, 255, 0],
        'tr909oh': [255, 150, 0]
    };

    if (colors[s]) return colors[s];

    let charCode = s.charCodeAt(0) || 0;

    let r = (charCode * 123) % 255;
    let g = (charCode * 456) % 255;
    let b = (charCode * 789) % 255;

    return [r, g, b];
}
```
* Este bloque permite cambiar el color del bombo en tiempo real

```
if (s === "tr909bd" && task.controlState.rgb_1)
```
* Primero verifica que el sonido sea el bombo y si se estan enviando datos RGB desde OSC

```
.map(v => Number(v))
```
* Convierte los datos de RGB en numeros para evitar incompatibilidad con p5.js

##### Slider - Size

<img width="72" height="261" alt="image" src="https://github.com/user-attachments/assets/d51603f2-675a-4242-83f2-4c11b5fb2468" />


```
function dibujarBombo(p, c) {

    // 🎛 tamaño controlado por OSC
    let maxSize = task.controlState.size || 600;

    let d = lerp(100, maxSize, p);

    let alpha = lerp(255, 0, p);

    fill(c[0], c[1], c[2], alpha);

    circle(width / 2, height / 2, d);
}
```
* Ajusta el tamaño del bombo

```
let maxSize = task.controlState.size || 600;
```
* Lee el valor enviado por OSC y si no hay, usa 600 como valor por defecto

```
lerp(100, maxSize, p)
```
* Interpola suavemente l tamaño del circulo durante la animacion para que se vea fluido


##### Toggle o Switch

<img width="165" height="151" alt="image" src="https://github.com/user-attachments/assets/c72c269f-d6ce-455f-be73-bb6ca59459da" />


```function draw() {

    // 🔥 SIEMPRE actualizar FSM y eventos
    task.update();

    // 🎛 Toggle OFF
    if (task.controlState.toggle !== undefined &&
        task.controlState.toggle === 0) {

        background(0);
        return;
    }

    background(0, 30);

    let now = Date.now() + task.LATENCY_CORRECTION;*
```
* Permite activar y desactivar todas las visuales

```
task.update();
```
* Actualiza FSM
  - Se pone antes del toggle pa que asi el return ocurra correctamente depsues del update y asi se puedan volver a encender las visuales
 
```
if (task.controlState.toggle === 0)
```
* Se asegura de que si el switch esta apagado, se detiene el render, por ende no sale nada

##### Como se configuraron los widgets

En general, los address que se les asignan es para que se sepa que widget esta mandando los datos

###### RGB
* Solo se le puso el adres /rgb_1 para que se sepa que widget mando esos datos

###### Slider
* Se le asigno un address, /size
* Y se le puso un tamaño maximo y un minimo
<img width="306" height="106" alt="image" src="https://github.com/user-attachments/assets/2438eed1-6122-4406-9ad1-e1ad252ea4f8" />

###### Switch
* Se le asigno un address, /toggle
* Y se le pusieron los siguientes valores
<img width="334" height="111" alt="image" src="https://github.com/user-attachments/assets/209aaeee-eb1d-4e59-9306-7dcd9cac49a8" />





```
```

<br><br>

## Bitácora de reflexión
