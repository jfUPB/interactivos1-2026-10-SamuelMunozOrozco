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

    // 🔥 normalización (LO QUE TE PIDE EL CURSO)
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

    // 🔥 pasar al adapter (NO lógica aquí)
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
* (Preguntale a chat porque se creo un setrvidor nuevo dentro del bridgeServer, y porque no se hizo algo parecido con los adapter. Y tambien si esto fue implementado po chat como una solucion al problema de que no me funcionaba el sistema y si antes de eso era diferente y mas simple. Explicame porque lo hiciste)

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
* (pregunale a chat si esto esta bien, porque no pues que el bridge no ponia hacer nada que tuviera que ver con traducir datos y eso?)

#### 4. Pasar los datos a el adapter
```js
adapter.handleIncoming(msg);
```
* El bridge le pasa los datos al adapter (Preguntale a chat, si esto lo que quiere decir es que primero llegan los datos de Strudel, bridgeserver los guarda y luego se los pasa a adapter para traducirlos, o no es bridgeserver sino el nuevo que se creo dentro del mismo y si es asi, porque no se mantiene la logica de que el adapter es a quien le llegan los datos, el bridgeserver los recibe y luego los manda al frontend)

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






<br><br>

## Bitácora de reflexión
