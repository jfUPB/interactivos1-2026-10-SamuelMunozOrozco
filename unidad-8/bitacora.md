# Unidad 8

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Que se tenia que hacer

* Teniamos que integrar todo lo aprendido desde la unidad 5 hasta esta ultima unidad, debiamos ser capaces de integrar tres fuentes de informacion diferentes, que manden datos al mismo tiempo y que estos puedan ser leidos correctamente y mandados todos juntos por el bridgeserver al frontend

### Arquitectura general del sistema

#### Microbit

```
MICROBIT
↓
MicrobitAsciiAdapter
↓
                 ┌───────────────────────┐
STRUDEL ───────→ │                       │
↓                │     bridgeServer      │
StrudelAdapter   │                       │
↓                └───────────────────────┘
                                   ↓
OPEN STAGE CONTROL                WebSocket
↓                                  ↓
OpenStageControlAdapter     bridgeClient.js
                                   ↓
                                FSMTask
                                   ↓
                             updateLogic()
                                   ↓
                             drawRunning()
                                   ↓
                         VISUALES EN p5.js
```

### Rol de cada fuente

| Fuente | Qué controla | Cómo entra al sistema | Por qué se usa |
|---|---|---|---|
| micro:bit | Cambio de formas visuales y partículas | Serial → MicrobitAsciiAdapter | Permite interacción física y gestual |
| Strudel | Generación de eventos musicales y temporales | WebSocket → StrudelAdapter | Controla sincronización audiovisual |
| Open Stage Control | Parámetros globales visuales | OSC → OpenStageControlAdapter | Permite manipulación performática en tiempo real |


### Explicacion del recorrido

```
Adapter -> bridgeServer -> bridgeClient -> FSMTask -> updateLogic -> drawRunning
```

#### Adapter

* Son los encargados de traducir la informacion que llega externamente al sistema
* Cada adpater recibe datos distintos y los normaliza para que el sistema pueda trabajar con ellos

#### bridgeServer

* Es por asi decirlo, el comunicador del sistema. Lo que hace es:
  - Tiene activos al mismo tiempo todos los adapters
  - Recibe la informacion de cada fuente
  - Normaliza los eventos
  - Reenvia los datos al naved=gador mediante WebSocket
* El servidor permite que las tres fuentes de informacion funcionen al mismo tiempo con un solo sketch


#### bridgeClient
* Se conecta al WebSocket del bridgeServer y recibe todos los eventos del sistema
* Su funcion es tranportar los datos al sketch


#### FSMTask

* Organiza la logica del sistema mediante estados y eventos
* Se encarga de separar las funciones y que ninguna interfiera con una parte de la arquitectura que no le corresponde. Separa:
  - Conexion
  - Recepcion de datos
  - Logica
  - Render visual
 

#### updateLogic

* Procesa y organiza todos los datos que le llegan
  - Interpreta datos del microbit
  - Guarda estados OSC
  - Almacena eventos de strudel
  - Cambia las formas viuales
  - Controla las particulas y parametros globales


 #### drawRunning

 * Renderiza visualmente toda la informacion procesada
   - Dibuja las animaciones
   - Cambia los colores, Aplica todos los efectos

 
### Evidencias
<img width="1912" height="762" alt="image" src="https://github.com/user-attachments/assets/5d359244-9fed-4c01-82a2-869f39241f99" />

<br><br>

<img width="1918" height="695" alt="image" src="https://github.com/user-attachments/assets/acafc05b-565d-43b7-a04f-2011876640e6" />

<br><br>

<img width="1913" height="692" alt="image" src="https://github.com/user-attachments/assets/3a2186f1-8f16-4fea-92b0-16d64fbd5db4" />


### Codigo

#### Sketch.js

```
// ========================================
// EVENTS
// ========================================

const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};

// ========================================
// TASK
// ========================================

class PainterTask extends FSMTask {

    constructor() {
        super();

        // =========================
        // MICROBIT
        // =========================

        this.rxData = {
            x: 0,
            y: 0,
            btnA: false,
            btnB: false,
            prevA: false,
            prevB: false,
            ready: false
        };

        // =========================
        // STRUDEL
        // =========================

        this.eventQueue = [];
        this.activeAnimations = [];
        this.LATENCY_CORRECTION = 0;

        // =========================
        // OSC
        // =========================

        this.controlState = {
            globalScale: 1,
            rotationForce: 0,
            globalColor: [255,255,255],
            burst: 0
        };

        // =========================
        // FORMAS MICROBIT
        // =========================

        this.kickShapes = [
            "circle",
            "square",
            "triangle",
            "rectangle",
            "star",
            "hexagon",
            "octagon",
            "cross"
        ];

        this.snareShapes = [
            "tripleHorizontal",
            "vertical",
            "tripleVertical",
            "diagRight",
            "diagLeft",
            "crossLines"
        ];

        this.particleShapes = [
            "square",
            "circle",
            "triangle",
            "diamond",
            "x"
        ];

        this.currentKickShape = "circle";
        this.currentSnareShape = "tripleHorizontal";
        this.currentParticleShape = "square";

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
            background(0);

            console.log("System ready");

        }

        else if (ev.type === EVENTS.DISCONNECT) {

            this.transitionTo(this.estado_esperando);

        }

        else if (ev.type === EVENTS.DATA) {

            this.updateLogic(ev.payload);

        }

        else if (ev.type === "EXIT") {

            cursor();

        }
    };

    updateLogic(data) {

        // =========================
        // MICROBIT
        // =========================

        if (data.type === "microbit") {

            this.rxData.ready = true;

            this.rxData.x = map(data.x, -2048, 2047, 0, width);
            this.rxData.y = map(data.y, -2048, 2047, 0, height);

            this.rxData.btnA = data.btnA;
            this.rxData.btnB = data.btnB;

            // BOTON A
            if (this.rxData.btnA && !this.rxData.prevA) {

                this.currentKickShape =
                    random(this.kickShapes);

                console.log("Kick Shape:", this.currentKickShape);
            }

            // BOTON B
            if (this.rxData.btnB && !this.rxData.prevB) {

                this.currentSnareShape =
                    random(this.snareShapes);

                console.log("Snare Shape:", this.currentSnareShape);
            }

            // SHAKE
            let shakeAmount =
                abs(data.x) + abs(data.y);

            if (shakeAmount > 2500) {

                this.currentParticleShape =
                    random(this.particleShapes);

                console.log("Particle Shape:", this.currentParticleShape);
            }

            this.rxData.prevA = this.rxData.btnA;
            this.rxData.prevB = this.rxData.btnB;
        }

        // =========================
        // STRUDEL
        // =========================

        if (data.type === "strudel") {

            this.eventQueue.push({
                timestamp: data.timestamp,
                sound: data.payload.s,
                delta: data.payload.delta
            });

            this.eventQueue.sort((a, b) =>
                a.timestamp - b.timestamp
            );
        }

        // =========================
        // OSC
        // =========================

        if (data.type === "osc") {

            const addr = data.payload.address;
            const args = data.payload.args;

            const key = addr.replace("/", "");

            let value =
                args.length === 1 ? args[0] : args;

            this.controlState[key] = value;

            console.log("OSC:", key, this.controlState[key]);
        }
    }
}

// ========================================
// VARIABLES
// ========================================

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

// ========================================
// SETUP
// ========================================

function setup() {

    createCanvas(windowWidth, windowHeight);

    rectMode(CENTER);
    noStroke();

    background(0);

    painter = new PainterTask();

    bridge = new BridgeClient();

    bridge.onConnect(() => {

        connectBtn.html("Disconnect");

        painter.postEvent({
            type: EVENTS.CONNECT
        });
    });

    bridge.onDisconnect(() => {

        connectBtn.html("Connect");

        painter.postEvent({
            type: EVENTS.DISCONNECT
        });
    });

    bridge.onStatus((s) => {

        console.log(
            "BRIDGE STATUS:",
            s.state,
            s.detail ?? ""
        );
    });

    bridge.onData((data) => {

        console.log("DATA:", data);

        painter.postEvent({
            type: EVENTS.DATA,
            payload: data
        });
    });

    connectBtn = createButton("Connect");

    connectBtn.position(10, 10);

    connectBtn.style("background-color", "white");
    connectBtn.style("color", "black");
    connectBtn.style("border", "none");
    connectBtn.style("padding", "12px");
    connectBtn.style("font-size", "16px");

    connectBtn.mousePressed(() => {

        if (bridge.isOpen) {

            bridge.close();

        } else {

            bridge.open();
        }
    });

    renderer.set(
        painter.estado_corriendo,
        drawRunning
    );
}

// ========================================
// DRAW
// ========================================

function draw() {

    painter.update();

    renderer.get(painter.state)?.();
}

// ========================================
// MAIN RENDER
// ========================================

function drawRunning() {

    background(0, 30);

    let now =
        Date.now() +
        painter.LATENCY_CORRECTION;

    while (
        painter.eventQueue.length > 0 &&
        now >= painter.eventQueue[0].timestamp
    ) {

        let ev = painter.eventQueue.shift();

        // ========================================
        // NUEVO BURST SWITCH
        // ========================================

        let burstMultiplier =
            painter.controlState.burst ? 4 : 1;

        for (let i = 0; i < burstMultiplier; i++) {

            painter.activeAnimations.push({

                startTime: ev.timestamp,

                duration: ev.delta * 1000,

                type: ev.sound,

                x: random(width * 0.2, width * 0.8),

                y: random(height * 0.2, height * 0.8),

                color: getColorForSound(ev.sound)
            });
        }
    }

    for (
        let i = painter.activeAnimations.length - 1;
        i >= 0;
        i--
    ) {

        let anim =
            painter.activeAnimations[i];

        let elapsed =
            now - anim.startTime;

        let progress =
            elapsed / anim.duration;

        if (progress <= 1.0) {

            dibujarElemento(anim, progress);

        } else {

            painter.activeAnimations.splice(i, 1);
        }
    }
}

// ========================================
// DIBUJAR ELEMENTOS
// ========================================

function dibujarElemento(anim, p) {

    push();

    const color = anim.color;

    switch (anim.type) {

        case "tr909bd":
            dibujarKick(p, color);
            break;

        case "tr909sd":
            dibujarSnare(p, color);
            break;

        case "tr909hh":
        case "tr909oh":
            dibujarHat(anim, p, color);
            break;

        default:
            dibujarDefault(anim, p, color);
            break;
    }

    pop();
}

// ========================================
// KICK
// ========================================

function dibujarKick(p, c) {

    let scale =
        painter.controlState.globalScale || 1;

    let rotation =
        painter.controlState.rotationForce || 0;

    let d =
        lerp(100, 600, p) * scale;

    let alpha =
        lerp(255, 0, p);

    fill(c[0], c[1], c[2], alpha);

    translate(width/2, height/2);

    rotate(p * rotation);

    switch (painter.currentKickShape) {

        case "circle":
            circle(0,0,d);
            break;

        case "square":
            rect(0,0,d,d);
            break;

        case "rectangle":
            rect(0,0,d*1.5,d*0.6);
            break;

        case "triangle":
            triangle(
                0,-d/2,
                -d/2,d/2,
                d/2,d/2
            );
            break;

        case "hexagon":
            polygon(0,0,d/2,6);
            break;

        case "octagon":
            polygon(0,0,d/2,8);
            break;

        case "cross":

            rect(0,0,d*0.25,d);
            rect(0,0,d,d*0.25);

            break;

        case "star":

            estrella(0,0,d/2,d/4,5);

            break;
    }
}

// ========================================
// SNARE
// ========================================

function dibujarSnare(p, c) {

    let scale =
        painter.controlState.globalScale || 1;

    let rotation =
        painter.controlState.rotationForce || 0;

    let alpha =
        lerp(255,0,p);

    stroke(c[0], c[1], c[2], alpha);

    strokeWeight(10);

    translate(width/2,height/2);

    rotate(p * rotation);

    let w = width * scale;

    switch (painter.currentSnareShape) {

        case "tripleHorizontal":

            line(-w/2,-100,w/2,-100);
            line(-w/2,0,w/2,0);
            line(-w/2,100,w/2,100);

            break;

        case "vertical":

            line(0,-height/2,0,height/2);

            break;

        case "tripleVertical":

            line(-200,-height/2,-200,height/2);
            line(0,-height/2,0,height/2);
            line(200,-height/2,200,height/2);

            break;

        case "diagRight":

            line(-w/2,-height/2,w/2,height/2);
            line(-w/2,-height/2+200,w/2,height/2+200);
            line(-w/2,-height/2-200,w/2,height/2-200);

            break;

        case "diagLeft":

            line(w/2,-height/2,-w/2,height/2);
            line(w/2,-height/2+200,-w/2,height/2+200);
            line(w/2,-height/2-200,-w/2,height/2-200);

            break;

        case "crossLines":

            line(-w/2,0,w/2,0);
            line(0,-height/2,0,height/2);

            break;
    }

    noStroke();
}

// ========================================
// HATS
// ========================================

function dibujarHat(anim, p, c) {

    let scale =
        painter.controlState.globalScale || 1;

    let rotation =
        painter.controlState.rotationForce || 0;

    let sz =
        lerp(40, 0, p) * scale;

    fill(c[0], c[1], c[2]);

    push();

    translate(anim.x, anim.y);

    rotate(p * rotation);

    switch (painter.currentParticleShape) {

        case "square":
            rect(0,0,sz,sz);
            break;

        case "circle":
            circle(0,0,sz);
            break;

        case "triangle":

            triangle(
                0,-sz/2,
                -sz/2,sz/2,
                sz/2,sz/2
            );

            break;

        case "diamond":

            beginShape();

            vertex(0,-sz/2);
            vertex(sz/2,0);
            vertex(0,sz/2);
            vertex(-sz/2,0);

            endShape(CLOSE);

            break;

        case "x":

            stroke(c[0], c[1], c[2]);

            line(-sz/2,-sz/2,sz/2,sz/2);
            line(sz/2,-sz/2,-sz/2,sz/2);

            noStroke();

            break;
    }

    pop();
}

// ========================================
// DEFAULT
// ========================================

function dibujarDefault(anim, p, c) {

    let scale =
        painter.controlState.globalScale || 1;

    let rotation =
        painter.controlState.rotationForce || 0;

    let size =
        lerp(100, 0, p) * scale;

    translate(anim.x, anim.y);

    rotate(p * rotation);

    stroke(c[0], c[1], c[2]);

    noFill();

    rect(0,0,size,size);

    noStroke();
}

// ========================================
// COLOR GLOBAL OSC
// ========================================

function getColorForSound(s) {

    let baseColors = {

        tr909bd: [255,0,120],
        tr909sd: [0,200,255],
        tr909hh: [255,255,0],
        tr909oh: [255,150,0]
    };

    let base =
        baseColors[s] || [255,255,255];

    let tint =
        painter.controlState.globalColor ||
        [255,255,255];

    return [

        (base[0] + tint[0]) / 2,
        (base[1] + tint[1]) / 2,
        (base[2] + tint[2]) / 2
    ];
}

// ========================================
// HELPERS
// ========================================

function polygon(x, y, radius, npoints) {

    beginShape();

    for (let a = 0; a < TWO_PI; a += TWO_PI / npoints) {

        let sx = x + cos(a) * radius;
        let sy = y + sin(a) * radius;

        vertex(sx, sy);
    }

    endShape(CLOSE);
}

function estrella(x, y, radius1, radius2, npoints) {

    let angle = TWO_PI / npoints;
    let halfAngle = angle / 2.0;

    beginShape();

    for (let a = 0; a < TWO_PI; a += angle) {

        let sx = x + cos(a) * radius1;
        let sy = y + sin(a) * radius1;

        vertex(sx, sy);

        sx = x + cos(a + halfAngle) * radius2;
        sy = y + sin(a + halfAngle) * radius2;

        vertex(sx, sy);
    }

    endShape(CLOSE);
}

function windowResized() {

    resizeCanvas(windowWidth, windowHeight);
}
```


#### bridgeServer.js

```
//   Uso:
//     node bridgeServer.js
//     node bridgeServer.js --serialPort COM5 --baud 115200

const { WebSocketServer } = require("ws");
const { SerialPort } = require("serialport");

const MicrobitAsciiAdapter = require("./adapters/MicrobitASCIIAdapter");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");

const log = {
  info: (...args) =>
    console.log(`[${new Date().toISOString()}] [INFO]`, ...args),

  warn: (...args) =>
    console.warn(`[${new Date().toISOString()}] [WARN]`, ...args),

  error: (...args) =>
    console.error(`[${new Date().toISOString()}] [ERROR]`, ...args)
};

// =========================================================
// HELPERS
// =========================================================

function getArg(name, def = null) {

  const i = process.argv.indexOf(`--${name}`);

  if (i >= 0 && i + 1 < process.argv.length) {
    return process.argv[i + 1];
  }

  return def;
}

function hasFlag(name) {

  return process.argv.includes(`--${name}`);
}

function nowMs() {

  return Date.now();
}

function safeJsonParse(s) {

  try {

    return JSON.parse(s);

  } catch (e) {

    log.warn("Failed to parse JSON:", s);

    return null;
  }
}

function broadcast(wss, obj) {

  const text = JSON.stringify(obj);

  for (const client of wss.clients) {

    if (client.readyState === 1) {

      client.send(text);
    }
  }
}

function status(wss, state, detail = "") {

  broadcast(wss, {
    type: "status",
    state,
    detail,
    t: nowMs()
  });
}

// =========================================================
// CONFIG
// =========================================================

const WS_PORT =
  parseInt(getArg("wsPort", "8081"), 10);

const SERIAL_PATH =
  getArg("serialPort", null);

const BAUD =
  parseInt(getArg("baud", "115200"), 10);

const VERBOSE =
  hasFlag("verbose");

// =========================================================
// MICROBIT AUTO DETECT
// =========================================================

async function findMicrobitPort() {

  const ports = await SerialPort.list();

  const microbit = ports.find(p =>
    p.vendorId &&
    parseInt(p.vendorId, 16) === 0x0D28
  );

  return microbit?.path ?? null;
}

// =========================================================
// MAIN
// =========================================================

async function main() {

  // =======================================================
  // WS SERVER
  // =======================================================

  const wss = new WebSocketServer({
    port: WS_PORT
  });

  log.info(
    `WS listening on ws://127.0.0.1:${WS_PORT}`
  );

  // =======================================================
  // MULTIPLE ADAPTERS
  // =======================================================

  const microbitPath =
    SERIAL_PATH ??
    await findMicrobitPort();

  if (!microbitPath) {

    log.error(
      "micro:bit not found. Use --serialPort manually."
    );

    process.exit(1);
  }

  log.info(
    `micro:bit found at ${microbitPath}`
  );

  // 🔵 MICROBIT
  const microbitAdapter =
    new MicrobitAsciiAdapter({
      path: microbitPath,
      baud: BAUD,
      verbose: VERBOSE
    });

  // 🟣 STRUDEL
  const strudelAdapter =
    new StrudelAdapter({
      verbose: VERBOSE
    });

  // 🎛 OSC
  const oscAdapter =
    new OpenStageControlAdapter({
      port: 9000
    });

  // =======================================================
  // MICROBIT EVENTS
  // =======================================================

  microbitAdapter.onConnected = (detail) => {

    log.info(
      `[MICROBIT] Connected: ${detail}`
    );

    status(
      wss,
      "connected",
      detail
    );
  };

  microbitAdapter.onDisconnected = (detail) => {

    log.warn(
      `[MICROBIT] Disconnected: ${detail}`
    );

    status(
      wss,
      "disconnected",
      detail
    );
  };

  microbitAdapter.onError = (detail) => {

    log.error(
      `[MICROBIT] Error: ${detail}`
    );

    status(
      wss,
      "error",
      detail
    );
  };

  // 🔥 BUG FIX #1
  // retransmitir microbit al frontend

  microbitAdapter.onData = (data) => {

    broadcast(wss, {

      type: "microbit",

      x: data.x,
      y: data.y,

      btnA: !!data.btnA,
      btnB: !!data.btnB,

      t: nowMs()
    });
  };

  // =======================================================
  // STRUDEL EVENTS
  // =======================================================

  strudelAdapter.onConnected = (detail) => {

    log.info(
      `[STRUDEL] Connected: ${detail}`
    );
  };

  strudelAdapter.onDisconnected = (detail) => {

    log.warn(
      `[STRUDEL] Disconnected: ${detail}`
    );
  };

  strudelAdapter.onError = (detail) => {

    log.error(
      `[STRUDEL] ${detail}`
    );
  };

  strudelAdapter.onData = (data) => {

    broadcast(wss, data);
  };

  // =======================================================
  // OSC EVENTS
  // =======================================================

  oscAdapter.onConnected = (detail) => {

    log.info(
      `[OSC] Connected: ${detail}`
    );
  };

  oscAdapter.onDisconnected = (detail) => {

    log.warn(
      `[OSC] Disconnected: ${detail}`
    );
  };

  oscAdapter.onError = (detail) => {

    log.error(
      `[OSC] ${detail}`
    );
  };

  oscAdapter.onData = (data) => {

    broadcast(wss, data);
  };

  // =======================================================
  // READY
  // =======================================================

  status(
    wss,
    "ready",
    "bridge active"
  );

  // =======================================================
  // CLIENT CONNECTIONS
  // =======================================================

  wss.on("connection", (ws, req) => {

    log.info(
      `[NETWORK] Client connected from ${req.socket.remoteAddress}`
    );

    ws.send(JSON.stringify({

      type: "status",

      state: "ready",

      detail: "bridge active",

      t: nowMs()
    }));

    // =====================================================
    // CLIENT -> SERVER
    // =====================================================

    ws.on("message", async (raw) => {

      const msg =
        safeJsonParse(
          raw.toString("utf8")
        );

      if (!msg) return;

      // ===================================================
      // CONNECT
      // ===================================================

      if (msg.cmd === "connect") {

        log.info(
          `[NETWORK] connect requested`
        );

        try {

          // 🔥 conectar TODOS los adapters

          if (!microbitAdapter.connected) {
            await microbitAdapter.connect();
          }

          if (!strudelAdapter.connected) {
            await strudelAdapter.connect();
          }

          if (!oscAdapter.connected) {
            await oscAdapter.connect();
          }

          status(
            wss,
            "connected",
            "multi-adapter bridge active"
          );

        } catch (e) {

          const detail =
            `connect failed: ${e.message || e}`;

          log.error(detail);

          status(
            wss,
            "error",
            detail
          );
        }

        return;
      }

      // ===================================================
      // DISCONNECT
      // ===================================================

      if (msg.cmd === "disconnect") {

        log.info(
          `[NETWORK] disconnect requested`
        );

        try {

          // 🔥 BUG FIX #2
          // desconectar TODOS los adapters

          if (microbitAdapter.connected) {
            await microbitAdapter.disconnect();
          }

          if (strudelAdapter.connected) {
            await strudelAdapter.disconnect();
          }

          if (oscAdapter.connected) {
            await oscAdapter.disconnect();
          }

          status(
            wss,
            "disconnected",
            "all adapters disconnected"
          );

        } catch (e) {

          const detail =
            `disconnect failed: ${e.message || e}`;

          log.error(detail);

          status(
            wss,
            "error",
            detail
          );
        }

        return;
      }

      // ===================================================
      // MICROBIT LED
      // ===================================================

      if (msg.cmd === "setLed") {

        try {

          await microbitAdapter.handleCommand?.(msg);

        } catch (e) {

          const detail =
            `command failed: ${e.message || e}`;

          log.error(detail);

          status(
            wss,
            "error",
            detail
          );
        }

        return;
      }
    });

    // =====================================================
    // CLOSE
    // =====================================================

    ws.on("close", () => {

      log.info(
        `[NETWORK] Client disconnected`
      );
    });
  });
}

// =========================================================
// START
// =========================================================

main().catch((e) => {

  log.error("Fatal:", e);

  process.exit(1);
});
```


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

  // =====================================================
  // GETTERS
  // =====================================================

  get isOpen() {

    return this._isOpen;
  }

  // =====================================================
  // CALLBACKS
  // =====================================================

  onData(callback) {

    this._onData = callback;
  }

  onConnect(callback) {

    this._onConnect = callback;
  }

  onDisconnect(callback) {

    this._onDisconnect = callback;
  }

  onStatus(callback) {

    this._onStatus = callback;
  }

  // =====================================================
  // OPEN
  // =====================================================

  open() {

    // ya existe socket abierto
    if (
      this._ws &&
      this._ws.readyState === WebSocket.OPEN
    ) {

      if (!this._isOpen) {

        this.send({
          cmd: "connect"
        });
      }

      return;
    }

    // limpiar socket previo
    if (this._ws) {

      this.close();
    }

    this._ws = new WebSocket(this._url);

    // ===================================================
    // WS OPEN
    // ===================================================

    this._ws.onopen = () => {

      // mantener arquitectura original
      this.send({
        cmd: "connect"
      });
    };

    // ===================================================
    // MESSAGE
    // ===================================================

    this._ws.onmessage = (event) => {

      let msg;

      try {

        msg = JSON.parse(event.data);

      } catch (e) {

        console.warn(
          "WS message is not JSON:",
          event.data
        );

        return;
      }

      // ================================================
      // STATUS
      // ================================================

      if (msg.type === "status") {

        this._onStatus?.(msg);

        // ============================================
        // CONNECTED
        // ============================================

        if (msg.state === "connected") {

          this._isOpen = true;

          this._onConnect?.();

          return;
        }

        // ============================================
        // READY
        // ============================================

        // 🔥 READY NO ES DISCONNECT
        if (msg.state === "ready") {

          return;
        }

        // ============================================
        // DISCONNECTED / ERROR
        // ============================================

        if (
          msg.state === "disconnected" ||
          msg.state === "error"
        ) {

          this._isOpen = false;

          this._onDisconnect?.();

          if (msg.state === "error") {

            this._ws?.close();

            this._ws = null;
          }

          return;
        }
      }

      // ================================================
      // MICROBIT
      // ================================================

      if (msg.type === "microbit") {

        this._onData?.(msg);

        return;
      }

      // ================================================
      // STRUDEL
      // ================================================

      if (msg.type === "strudel") {

        this._onData?.(msg);

        return;
      }

      // ================================================
      // OSC
      // ================================================

      if (msg.type === "osc") {

        this._onData?.(msg);

        return;
      }
    };

    // ===================================================
    // ERROR
    // ===================================================

    this._ws.onerror = (err) => {

      console.warn(
        "WS error:",
        err
      );
    };

    // ===================================================
    // CLOSE EVENT
    // ===================================================

    this._ws.onclose = () => {

      this._handleDisconnect();
    };
  }

  // =====================================================
  // CLOSE
  // =====================================================

  close() {

    if (
      !this._ws ||
      this._ws.readyState !== WebSocket.OPEN
    ) {
      return;
    }

    try {

      // avisar al bridge
      this.send({
        cmd: "disconnect"
      });

      this._isOpen = false;

      // cerrar websocket real
      this._ws.close();

    } catch (e) {

      console.warn(
        "Failed to disconnect:",
        e
      );
    }
  }

  // =====================================================
  // SEND
  // =====================================================

  send(obj) {

    if (
      !this._ws ||
      this._ws.readyState !== WebSocket.OPEN
    ) {
      return;
    }

    this._ws.send(
      JSON.stringify(obj)
    );
  }

  // =====================================================
  // HANDLE DISCONNECT
  // =====================================================

  _handleDisconnect() {

    this._isOpen = false;

    this._ws = null;

    this._onDisconnect?.();
  }
}
```






## Bitácora de reflexión
