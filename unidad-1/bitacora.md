# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 01
#### Que es un sistema fisico interactivo?
Un sistema fisico interactivo es aquel que combina lo digital con lo real, nos permite que por medio diferentes formas de interactuar con nuestro entorno, se tengan una respuesta personalizada y variada, segun lo que el sistema tenga para a ofrecer

#### Como podria aplicar los visto en mi perfil profesional?
Como yo quiero ir por la linea de animacion, seria chevere que algunos efectos visuales dentro de un corto o una serie, reaccionen segun los sonidos, que dependiendo de la musica, el tono de los dialogos, la gravedad o agudez de un sonido, estos efectos varien, dandole un toque mas espontaneo y tambien serviria para optimizar el trabajo, para evitar animar TODO frame por frame


### Actividad 02
#### que es el diseño/arte generatio?
En este tipo de diseño el artista o diseñor no hace hasta el minimo detalle a mano, por medio de algoritmos y codigos, crea un sistema que al darle unas reglas especificas, ese sistema tira un resultado basado en esas reglas. Aqui el artista no hace todo, solo establece el contexto y las reglas de su vision, y el sistema se encarga de darle un resultado en base a eso.

#### Como se puede aplicar en tu perfil profesional?
Tal vez para crear una historia o un mundo en el cual basar mi historia, podria dar instrucciones del tono que quiero que tenga la historia, en que clase de hambiente o epoca se desarrolle y que el sistema me de un mundo ya estructurado segun mis especificaciones


### Actividad 4
####  ¿Por qué no funcionaba el programa con was_pressed() y por qué funciona con is_pressed()?
Porque el WAS PRESSED funciona como un evento, al oprimir la tecla se manda una señal de que fue oprimida, pero no mas, solo una vez durante un frame y en el siguiente ya no resive nada. En cambio IS PRESSED es un estado ya que manda una señal de que esta siendo presionado todo el tiempo, hasta que se deje de oprimir el boton


## Bitácora de aplicación 

### Actividad 5
#### Que hicimos
Teniamos que crear un programa en p5.js que creara un circulo en un canvas y que cuando se presionara el boton A o B, el circulo se moviera a cualquier direccion en el eje X.

#### Como lo hicimos?
Usamos el mismo codigo de la clase anterior, porque ya tenia un circulo dibujado y el boton que conecta al microbit, solo teniamos que borrar las funciones innecesarias y colocar las nuevas.

#### Codigo Original

``` js
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
    let sendBtn = createButton('Send Love');
    sendBtn.position(220, 300);
    sendBtn.mousePressed(sendBtnClick);
    fill('white');
    ellipse(width / 2, height / 2, 100, 100);
}

function draw() {

    if(port.availableBytes() > 0){
        let dataRx = port.read(1);
        if(dataRx == 'A'){
            fill('red');
        }
        else if(dataRx == 'B'){
            fill('yellow');
        }
        else{
            fill('green');
        }
        background(220);
        ellipse(width / 2, height / 2, 100, 100);
        fill('black');
        text(dataRx, width / 2, height / 2);
    }


    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    }
    else {
        connectBtn.html('Disconnect');
    }
}

function connectBtnClick() {
    if (!port.opened()) {
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}

function sendBtnClick() {
    port.write('h');
}

´´´
```

#### Codigo de la actividad 5

``` js
let port;
let connectBtn;
let x = 200;

function setup() {
  createCanvas(400, 400);
  //x = width/2;

  background(220);

  port = createSerial();

  connectBtn = createButton("Connect to micro:bit");

  connectBtn.position(width / 3.1, 300);

  connectBtn.mousePressed(connectBtnClick);

  fill("white");

  ellipse(width / 2, height / 2, 100, 100);
}

function draw() {
  if (port.availableBytes() > 0) {
    let dataRx = port.read(1);

    if (dataRx == "A") {
      x -= 20;
    } else if (dataRx == "B") {
      x += 20;
    }
  }

  background(220);

  ellipse(x, height / 2, 100, 100);

  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } else {
    connectBtn.html("Disconnect");
  }
}

function connectBtnClick() {
  if (!port.opened()) {
    port.open("MicroPython", 115200);
  } else {
    port.close();
  }
}

```
```py
from microbit import *

uart.init(baudrate=115200)
display.show(Image.BUTTERFLY)

while True:
    if button_a.is_pressed():
        uart.write('A')
        sleep(500)
    if button_b.is_pressed():
        uart.write('B')
        sleep(500)
    if accelerometer.was_gesture('shake'):
        uart.write('C')
        sleep(500)
    if uart.any():
        data = uart.read(1)
        if data:
            if data[0] == ord('h'):
                display.show(Image.HEART)
                sleep(500)
                display.show(Image.HAPPY)

```

* Como habia dicho, se quitaron cosas del codigo original que ya no se necesitaban. Primero quitamos el boton de Send Love y eliminamos la ultima parte del cdigo que mandaba el dato "h" al microbit, "sendBtnClick() { port.write('h'); }", esta se encarga de mostrar el corazon y la carita feliz cuando se presionaba el boton Send Love. Tambien se borro la parte de "fill('black'); text(dataRx, width / 2, height / 2);", que se encargaba de poner una letra dentro del circulo cuando se presionaba uno de los dos botones. Y por ultimo se borro la parte del else if, que volvia el circulo verde si el microbit se agitaba.
* Algo que no se borro pero que si se cambio fue la siguiente parte "background(220); ellipse(width / 2, height / 2, 100, 100);", esta parte se saco del ciclo que de if y se puso afuera para que el circulo ya no estubiera fijo en la pantalla cambiando el parte del "width / 2" por una variable X ya previamente creada, para que asi se redibujara asi no llegaran datos nuevos y el movimiento fuera fluido.
* Ahora si cuando empezamos a poner el el codigo pal ejercicio 5, primero, como dije antes, se creo una variable X, que representara la posicion del circulo, valga la redundancia, en su eje X.
* Luego empezamos con las funciones de los botones. Ya borrado lo que hacia que el circulo cambiara de color, poniamos, dentro de la funcion del boton "a" "x -= 20", lo que hacia era correr el circulo 20 unidades, o pixeles a la izquierda. Y para el boton "b" "x += 20" que es exactamente lo mismo que en el boton a, pero hacia la derecha.

Y ya con eso finaliza la actividad 5.


## Bitácora de reflexión

#### Actividad 6

```py
from microbit import *

uart.init(baudrate=115200)

while True:

    if button_a.is_pressed():
        uart.write('A')
    else:
        uart.write('N')

    sleep(100)

```

```pj
  let port;
  let connectBtn;
  let connectionInitialized = false;

  function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton("Connect to micro:bit");
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
  }

  function draw() {
    background(220);

    if (port.opened() && !connectionInitialized) {
      port.clear();
      connectionInitialized = true;
    }

    if (port.availableBytes() > 0) {
      let dataRx = port.read(1);
      if (dataRx == "A") {
        fill("red");
      } else if (dataRx == "N") {
        fill("green");
      }
    }

    rectMode(CENTER);
    rect(width / 2, height / 2, 50, 50);

    if (!port.opened()) {
      connectBtn.html("Connect to micro:bit");
    } else {
      connectBtn.html("Disconnect");
    }
  }

  function connectBtnClick() {
    if (!port.opened()) {
      port.open("MicroPython", 115200);
      connectionInitialized = false;
    } else {
      port.close();
    }
  }

```

* Primero el objetivo de este sistema físico es crear un cuadrado de color verde, que al recibir un dato al oprimir un boton se vuelva de color rojo y cuando se deje de oprimir vuelva a ser verde.
#### Microbit
* Primero empezamos con el microbit. Creamos un ciclo While True, y ponemos "if button_a.is_pressed(): uart.write('A')" la funcion del "is_pressed" es hacer entender al sistema que lo que se manda es un estado, no un evento, que cuando se presiona, siga leyendo si sigue presionado, de esa manera cuando se presiona el boton A siga siendo de color rojo hasta que se deje de oprimir. Y la linea "uart.write('A')" es la encargada de mandar un dato por el cable al p5.
* Luego ponemos "else: uart.write('N')", esto se encarga de que cuando no se este mandando el dato A, se mande un dato N.
* Y por ultimo ponemos "sleep(100)", este lo ponemos fuera del ciclo if, pero dentro del ciclo While, ya que es el encargado de dictar cuanto se demora en quitar el color rojo.

#### p5.js
* Primero creamos las variables, las primeras dos "let port;" y " let connectBtn;" son las variables encargadas de la comunicación serial y el botón. Y por ultimo "let connectionInitialized = false;" que se encarga de no leer la información basura que quede al conectarse, así no active nada por accidente.
* En esta parte reamos el canvas con un ondo gris y el puerto serial.
  
"function setup() {

createCanvas(400, 400);

background(220);

port = createSerial();" 


* Y acá reamos el botón para conectar y desconectar el microbit.

"connectBtn = createButton("Connect to micro:bit");

connectBtn.position(80, 300);

connectBtn.mousePressed(connectBtnClick);", 

Creamos el botón para conectar y desconectar el microbit

* La parte del "function draw" es donde va toda la logica del sistema. Esta parte "if (port.opened() && !connectionInitialized) {port.clear();connectionInitialized = true;}" es la encargada de limpiar datos viejos que no interfieran con la logica del sistema.
*  "if (port.availableBytes() > 0)

*  { let dataRx = port.read(1);"

   la primera parte se encarga de preguntar si llego algún dato y la otra lo lee.
   
*  "if (dataRx == "A")

*  { fill("red"); }

*  else if (dataRx == "N") { fill("green");"

*  Esta ya es la logica principal. "if (dataRx == "A") { fill("red");" esto dice que si el dato que se lee es A, el cuadrado se vuelva rojo. "else if (dataRx == "N") { fill("green");" acá es cuando deje de llegar el dato A y llegue el N, se vuelva verde el cuadrado.
*  " rectMode(CENTER);
   
    rect(width / 2, height / 2, 50, 50);"

    Esta es la posición del cuadrado

* "if (!port.opened()) {
   
  connectBtn.html("Connect to micro:bit");
   
    } else {
   
  connectBtn.html("Disconnect");
   
    }
   
  }"

Y por ultimo, esta parte se encarga de cambiar el texto del botón que creamos
  









