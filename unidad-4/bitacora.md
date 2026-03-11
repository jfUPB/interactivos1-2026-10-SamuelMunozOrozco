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


### Explicacion codigo Sketch.js

#### Definicion deeventos
```js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
    KEY_PRESSED: "KEY_PRESSED",
    KEY_RELEASED: "KEY_RELEASED",
};
```
* Es un objeto que contiene los diferentes eventos que el sistema maneja
* Se hace esto para tener un codigo mas organizado y evitar errores de escritura
* Asi solo hay que copiar EVENT."El evento que queramos tener", sin temer a equivocarnos al escribirlo y tener un error

#### Que significa cada evento

##### CONNECT
* Se activa cuando el bridge se conecta con el microbit

#### DISCONNECT
* Se activa cuando se pierde la conexion

##### DATA
* Se activa cuando llegan datos del microbit

##### KEY_PRESSED
* Se activa cuando se presiona una tecla

##### KEY_RELEASED
* Se activa cuando el usuario sualta una tecla


### Clase principal del sistema - PainterTask
```js
class PainterTask extends FSMTask {
```
* Es la clase que controla el dibujo en el sitema
* El "extends FSMTask" significa que hereda de otra clase con ese nombre

Tiene dos estados principales

##### estado_esperando
* El sistema espera a que el microbit se conecte

##### estado_corriendo
* El microbit esta conectado y el programa puede dibujar

#### Constructor
```js
constructor() {
    super();

    this.c = color(181, 157, 0);
    this.lineSize = 100;
    this.angle = 0;
    this.clickPosX = 0;
    this.clickPosY = 0;
```
* recordar que el constructor es una funcion de una clase, que se ejecuta cuando se crea el objeto, en este caso, "painterTask"

Que hace "super()"?
* Llama al constructor de la clase "FSMTask". Porque "painterTask" hereda de "FSMTask"

#### Variables del constructor
* Color de la linea
  - this.c = color(181, 157, 0);

* Tamaño de la línea
  - this.lineSize = 100;

* Ángulo de rotación
  - this.angle = 0;
  - Cada que se dibuja una linea, la linea rota

* Posición del último clic
  - this.clickPosX = 0;
  
    this.clickPosY = 0;
  - Guarda la posicion de donde se empezo a dibujar

#### Datos que llegan del microbit
```js
this.rxData = {
    x: 0,
    y: 0,
    btnA: false,
    btnB: false,
    prevA: false,
    prevB: false,
    ready: false
};
```
* Aqui se guardan los datos que llegan desde el micrbit
Que representa cada variable
* X y Y
  - X:0. Valor del acelerometro en X
  - Y:0. Valor del acelerometro en Y

* btnA, btnB
  - btnA: Estado del boton A del microbit
  - btnB: Estado del boton B del microbit

* prevA, prevB, prevY
  - prevA: Guarda el estado anterior del boton A. Sirve cuando el bton acaba de ser presionado
  - prevB: Guarda el estado anterior del boton B
  - prevY: Indica si ya llegaron datos del microbit

#### Inicio de la maquina de estados
```js
this.transitionTo(this.estado_esperando);
```
Que hace esta linea?
* Pone el sistema en su primer estado. "estado_esperando"
* Es sistema empieza esperando la conexion del microbit

Que hace "transitionTo()" 
* Es una funcion de FSM
* Sirve para cambiar de estado






## Bitácora de aplicación 

Es esta actividad teniamos que simuklar un escenario de trabajo de la vida real, donde tenemos que integrar un nuevo hardware a un sistema software ya construido que no podemos romper.

Primero se nos pedia crear un nuevo adapter, que es una capa que traduce la comunicacion del hardware, el microbit, a el formato del sistema. El adapter es una copia de uno ya existente llamado MicrobitAsciiAdapter.js, en donde solo tenemos que cambiar una pequeña parte de la logica segun nos pide el contrato

### Contexto
Se nos pide implementar una nueva arquitectura de comunicacion entre un dispositivo microbit y una aplicacion web, pero sin romper el sistema software proporcionado.

Hay que crear un nuevo adapter a base de uno que ya tenemos, para permitir la comunicacion  por medio de un servidor puente (bridgeServer.js) que recibe los datos del microbit por puerto serial.

#### Flujo del sistema
El sistema fluiria de la siguiente manera:

1. Microbit: Manda la informacion de los sensores y botones por el puerto serial
2. Adapter: Interpreta la informacion
3. bridgeServer: distribuye los datos a los clientes conectados
4. bridgeClient: Recibe los datos en el navegador
5. PainterTask: Utiliza esos datos para generar el dibujo en el dibujo

#### Lo que pedia el ejercicio
Se nos pedia cambiar el sistema para que el microbit envie los datos a un nuevo adapter con un diferente formato que el anterior.

El primer adapter tenia un formato CSV (Comma Separated Values), ejemplo: 123,-50,true,false. Por lo tando el adapter separa los valores usando "," 

El nuevo adapter usaba separadores con "|" y usaba un checksum, para comparar los valores de los datos que llegaban con los que mandaba el microbit


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
  if (values.length !== 6) throw new ParseError(`Expected 4 values, got ${values.length}`);

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

##### Cambios en el codigo
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



###### Por que usamos values[1].split(":")[1]?
En el nuevo protocolo los valores vienen con una etiqueta

Ejemplo: x:-245

Y solo necesitamos el numero para asi realizar el Checksum. Si solo usamos "values[1]" solo tendriamos "x:-245"

Por eso al usar "values[1].split(":")[1]". Se separa la parte de "x:" de la parte "-245". Y por eso al final ponemos [1] pa quedarnos finalmente con ese valor

###### Para que usamos el Checksum?
Lo usamos como una medida de seguridad para verificar si los datos enviados por el microbit son correctos y no han sido corrompidos.

Pueden pasar como ruido en la comunicacion, datos incompletos o caracteres corruptos

Ejemplo:















Que es un objeto JSON



## Bitácora de reflexión






