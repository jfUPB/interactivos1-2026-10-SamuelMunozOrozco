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

#### MicrobitASCIIAdapter.js
* Lee los datos del puerto serial
* Interpreta el protocolo del microbit

#### SimAdapter.js
* Conecta con el simulador de microbit

#### BaseAdapter.js
* Es la clase base de los adapters, o sea, todos los adapters heredan de esta clase

#### Microbit
* Aqui es donde se encuentra el "hardware" real
* Envia los datos por "serial (UART)"

#### Simulador Microbit
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



## Bitácora de reflexión



