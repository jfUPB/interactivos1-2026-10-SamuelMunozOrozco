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

#### 1. Primero borramos lo que no necesitabamos.
* El boton de send love en el "function setup"
let sendBtn = createButton('Send Love');

sendBtn.position(220, 300);

sendBtn.mousePressed(sendBtnClick);

* Las funciones del "function draw"
if(port.availableBytes() > 0){  "Se deja ya que comprueba si hay datos en el puerto serial"

let dataRx = port.read(1); "Se deja po9rque lee 1 byte del puerto y lo guarda en dataRx

if(dataRx == 'A'){--"Es el encargado de que si el dato que recibe es A, cumpla la función que tenga

fill('red');--"Se borra y se pone "x-=20;". Lo que hace es mover el circulo 20 unidades o pixeles, no se, a la izquierda por el menos que tiene a la izquierda del igual"

}

else if(dataRx == 'B'){--"El encargado que si llega un dato B cumpla la función que tenga"

fill('yellow');--"Se borra y se pone "x+=20;". Lo que hace es mover el circulo 20 unidades o pixeles a la derecha por el mas que tiene a la izquierda del igual. 

}

else{

fill('green');--"Este se borra y no hace nada"

}

background(220);
        
ellipse(width / 2, height / 2, 100, 100);
        
fill('black');
        
text(dataRx, width / 2, height / 2);
#### Que se borra y porque se reemplaza

* El inicio el 


