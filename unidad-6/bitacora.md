# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Proposito
* En base a una nueva fuente de datos (Strudel) crear un nuevo tipo de adapter dentro de el codigo ya existente
* Asegurarnos que cada parte del codigo cumpla su funcion. Que el **adapter** solo traduzca, se transporten esos datos correctamente por el **bridge** y se ejecuten en el frontend en el timing correcto, usando timestamps

<br><br>

### Cambios
* En las unidades anteriores, usabamos datos que veniandirectamente de hardware, o sea, de un microbit; ese tipo de datos se procesa inmediatamente
* En cambio, Strudel no solo envia datos, sino que envia eventos con tiempo, con lo que hay que tener cuidado con el timing para cuando estos datos lleguen y s eprocese, los eventos ocurran correctamente en el tiepo correcto y que no se sienta un delay


<br><br>
<br><br>

### Codigo Nuevo Adapter (StrudelAdapter)

```js
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error {}

class StrudelAdapter extends BaseAdapter {
  constructor({ verbose = false } = {}) {
    super();
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;

    this.connected = true;
    this.onConnected?.("Strudel adapter ready");
  }

  async disconnect() {
    if (!this.connected) return;

    this.connected = false;
    this.onDisconnected?.("Strudel adapter stopped");
  }

  getConnectionDetail() {
    return "Strudel WebSocket input";
  }

  handleIncoming(msg) {
  try {
    if (!msg.args || !msg.timestamp) return;

    let params = {};

    // 🔥 convertir array a objeto
    for (let i = 0; i < msg.args.length; i += 2) {
      params[msg.args[i]] = msg.args[i + 1];
    }

    const sound = params.s;
    const delta = params.delta || 0.25;

    // 🔥 normalización 
    const normalized = {
      type: "strudel",
      timestamp: msg.timestamp,
      payload: {
        s: sound,
        delta: delta
      }
    };

    this.onData?.(normalized);

  } catch (e) {
    this.onError?.("Strudel parse error: " + e.message);
  }
}

  _parseMessage(msg) {
    if (!msg.args || !Array.isArray(msg.args)) {
      throw new ParseError("Missing args array");
    }

    let data = {};

    // 🔥 Igual que en microbit: transformamos datos
    for (let i = 0; i < msg.args.length; i += 2) {
      const key = msg.args[i];
      const value = msg.args[i + 1];
      data[key] = value;
    }

    if (!data.s) {
      throw new ParseError("Missing sound 's'");
    }

    return {
      type: "strudel",
      timestamp: msg.timestamp,
      payload: {
        s: String(data.s),
        delta: Number(data.delta ?? 0.25)
      }
    };
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
  }
}

module.exports = StrudelAdapter;
```

#### Que tiene de diferente

```js
handleIncoming(msg)
```
* Este lo que hace es reemplazar toda la parte de "_onChunk(chunk)" y "parseCsvLine(...)" porque ya no nos mandan datos en serial, sino mensajes directos
* En microbit, los datos llegaban como texto crudos, apenas separados por una "," y habia que reconstruir los datos en el otro formatoy completos
* En Strudel los datos ya vienen con una estuctura 

<br><br>

##### 1. Vreficacion de los mensajes
```js
if (!msg.args || !msg.timestamp) return;
```
* Se encarga de ver si los mesajes son tipo **arg**
* Y si tienen **timestamps**
* Si no los tiene pues error, no nos sirve

<br><br>

##### 2. Crear objeto params
```js
let params = {};
```
* Aqui se reconstruyen los datos
* **params** es donde guardas los datos organizados
Strudel manda:
```js
["s", "tr909bd", "delta", 0.25]
```
Entonces params hace:
```js
params["s"] = "tr909bd"
params["delta"] = 0.25
```
Y nos da al final esto:
```js
params = {
  s: "tr909bd",
  delta: 0.25
}
```
* Convierte la lista en un objeto util

<br><br>

##### 3. Conversion de los datos
```js
for (let i = 0; i < msg.args.length; i += 2) {
  params[msg.args[i]] = msg.args[i + 1];
}
```
* Convierte los datos tipo:
```js
["s", "tr909bd", "delta", 0.25]
```
* En:
```js
{
  s: "tr909bd",
  delta: 0.25
}
```

<br><br>

##### 4. Extraer Datos
```js
const sound = params.s;
const delta = params.delta || 0.25;
```
* Selecciona solo los datos necesarios no todo el mensaje
* Porque el frontend espera los siguientes datos
```js
data.payload.s
data.payload.delta
data.timestamp
```
* Si se manda todo, pueden llegar cosas como address, args, etc., y eso no lo necesitamos

<br><br>

##### 5. Normalizacion
```js
const normalized = {
  type: "strudel",
  timestamp: msg.timestamp,
  payload: {
    s: sound,
    delta: delta
  }
};
```
* Convierte los datos a formato estandar del sistema
Ejemplo Antes:
```js
{
  address: "/dirt/play",
  args: ["s", "tr909bd", "delta", 0.25],
  timestamp: 17123456789
}
```
Ejemplo Despues
```js
{
  type: "strudel",
  timestamp: 17123456789,
  payload: {
    s: "tr909bd",
    delta: 0.25
  }
}
```
* Ordena los datos, filtra lo importante y lo convierte a formato estandar

<br><br>

##### 6. Emitir Datos
```js
this.onData?.(normalized);
```
* Esta parte manda los datos al bridge


###### Preguntas
¿Por qué el adapter NO envía todo el mensaje?
* R/= Porque el mensaje viene con cosas innecesarias
Ejemplo:
```js
{
  address: "/dirt/play",
  args: ["s", "tr909bd", "delta", 0.25],
  timestamp: 17123456789,
  otrasCosas: "..."
}
```
Y el sistema solo necesita:
```js
s → qué sonido
delta → cuánto dura
timestamp → cuándo ocurre
```

<br><br>

¿Qué ventaja tiene convertir args en objeto?
* R/= Emensaje original:
```js
["s", "tr909bd", "delta", 0.25]
```
* Puede salir una lista desordenada y no sabes que posicion es que
Despues de convertir los datos:
```js
{
  s: "tr909bd",
  delta: 0.25
}
```
* Es mas claro el codigo
* Se puede acceder directamente a los datos

<br><br>
<br><br>

### Cambios en el bridgeServer
```js
 //===============Nuevo servidor para Strudel================
const wssStrudel = new WebSocketServer({ port: 8080 });

log.info(`Strudel WS listening on ws://127.0.0.1:8080`);

wssStrudel.on("connection", (ws) => {
  log.info("[STRUDEL] Connected");

  ws.on("message", (raw) => {
    const msg = safeJsonParse(raw.toString("utf8"));
    if (!msg) return;

    // pasar al adapter (NO lógica aquí)
    adapter.handleIncoming(msg);
  });

  ws.on("close", () => {
    log.info("[STRUDEL] Disconnected");
  });
});

  // =========================================================

```

#### 1. Creamos un nuevo servidor para Strudel
```js
const wssStrudel = new WebSocketServer({ port: 8080 });
```
* Antes solo teniamos que el bridge de una pasaba al frontend (8081)
* Ahora pimero va Strudel, luego siguie el bridge (8080) y luego sigue ahora si el frontend (8081)
* Estamos creando una nueva puerta para los datos

#### 2. Detectar cuando Strudel se conecta
```js
wssStrudel.on("connection", (ws) => {
  log.info("[STRUDEL] Connected");
```
* Detecta cuando Strudel se conecta

#### 3. Recibe mensajes
```js
ws.on("message", (raw) => {
  const msg = safeJsonParse(raw.toString("utf8"));
```
* Recibe los mensajes y los convierte a JSON usable

#### 4. Pasar los datos a el adapter
```js
adapter.handleIncoming(msg);
```
* El bridge le pasa los datos al adapter 

#### 5. Envio de datos al Frontend
```js
adapter.onData = (d) => {
  if (DEVICE === "strudel") {
    broadcast(wss, d);
  }
};
```
* Aqui el adapter ya creo los datos limpios y el bridge solo los envia al frontend

#### 6. Diferencias con el microbit
```js
 if (DEVICE.startsWith("microbit") || DEVICE === "sim") {
      broadcast(wss, {
        type: "microbit",
        x: d.x,
        y: d.y,
        btnA: !!d.btnA,
        btnB: !!d.btnB,
        t: nowMs(),
      });
      return;
    }
```
* El microbit necesitaba transformar los datos primero (que chat explique mejor esto, porque los datos no venian ya transformados en el adapter??)
* En cambio en Strudel ya vienen normalizados
```js
broadcast(wss, d);
```

##### Preguntas

¿Por qué el bridge NO debería procesar los datos de Strudel?
* R/= Para asi repartir mejor las responsabilidades entre transporte y procesamiento de datos

<br><br>

¿Por qué se creó un nuevo servidor dentro del bridgeServer?
* R/= Porque tenemos dos fuentes distintas de conexion
* Antes teniamos solo frontend -> bridge (8081). Ahora tenemos Strudel -> bridge (8080) -> frontend -> bridge (8081)
* Sigue siendo el mismo bridge, pero con dos entradas

<br><br>

¿Qué pasaría si el adapter manejara WebSockets directamente?
* Se romperia la arqutectura, tendria una responsabilidad que no le corresponde

<br><br>

¿Por qué es importante que el bridge no almacene datos?
* Porque el d=bridge solo deberia pasar los datos

<br><br>
<br><br>

### Cmabios en el bridgeClient
* El bridgeClient es quien recibe los datos en el frontend
* Antes sol oentendia mensajes del microbit:
```js
if (msg.type === "microbit") {
  this._onData?.(msg);
}
```

<br><br>
* Ahora se agrega mensajes de Strudel:
```js
 // 🟣 strudel
       if (msg.type === "strudel") {
        this._onData?.(msg);
       }
```
* Detecta que el mensaje viene de Strudel

<br><br>
```js
this._onData?.(...)
```
* Pasa los datos al sketch

<br><br>
<br><br>

### FRONTEND (StrudelSketch + FSM + Scheduling)

#### 1. Recepcion de datos
```js
bridge.onData((data) => {
  if (data.type === "strudel") {
    task.postEvent({
      type: EVENTS.DATA,
      payload: data
    });
  }
});
```
* El frontend recibe los datos del bridge y los convierte en eventos FSM

#### 2. FSM (StrudelTask)
```js
class StrudelTask extends FSMTask
```
* Maneja la logica del sistema
* Antes solo llegaba el dato y dibujaba
* Ahora es evento -> cola -> ejecucion -> dibujo

#### 3. Event queue (lo nuevo)
```js
this.eventQueue.push({
  timestamp,
  sound,
  delta
});
```
* Esta es la cola de eventos, donde se guardan

#### 4. Scheduling
```js
while (eventQueue.length > 0 && now >= timestamp)
```
* Se encarga de ejecutar los eventos cuando corresponde

#### 5. Active Animations
```js
activeAnimations.push(...)
```
* Son las listas de animaciones

##### 6. Render
```js
dibujarElemento(anim, progress);
```
* Simplemente se encarga de las visuales

### Respuesta puntos bitacora

#### Qué estructura final de mensaje decidiste usar
```js
{
  type: "strudel",
  timestamp: number,
  payload: {
    s: string,
    delta: number
  }
}
```
* Usamos un formato normalizado
* Separa el tipo de fuente
* incluye el timepo
* Encapsula los datos relevantes

#### Cómo conectaste bridgeClient.js, FSMTask, updateLogic y drawRunning
Este fue el flujo
```
bridge → BridgeClient → FSMTask → updateLogic → draw()
```

<br><br>
1. Primero el CridgeClient recibe datos
```js
bridge.onData((data) => {
```
* Los datos que llegan del servidor

<br><br>
2. Se envian a la FSM
```js
task.postEvent({
  type: EVENTS.DATA,
  payload: data
});
```
* Convierte los datos en eventos

<br><br>
3. FSM procesa los eventos
```js
estado_corriendo → updateLogic(data)
```

<br><br>
4. updateLogic guarda en cola 
```js
this.eventQueue.push(...)
```
* Almacena los eventos

<br><br>
5. draw ejecuta y renderiza
```js
draw() → scheduling → animaciones → render
```
* Ocurren todas las visuales

#### Cómo separaste recepción, cola temporal y renderizado

1. Recepcion
```js
bridge.onData(...)
```
* Recibe los datos

<br><br>
2. Cola temporal
```js
updateLogic()
```
* Guarda los eventos en:
```js
eventQueue
```
* Ordenados por el tiempo

<br><br>
3. Renderizado
```js
draw()
```
* Hace el Scheduling
* Animacion
* Dibujo







<br><br>
<br><br>

## Bitácora de reflexión
