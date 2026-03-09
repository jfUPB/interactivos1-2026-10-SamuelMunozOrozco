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

### bridgeClient.js
* Permite que el navegador se comunique con el servidor (Ya que el navegador no puede hablar directamente con el "hardware")

#### Server Bridge (bridgeServer.js)
* Actua como puente entre el navegador y el hardware
* Recibe los comandos del navegador, lee los datos del "hardware" y envia los datos al navegador

#### Adapters
* Es una capa que traduce la comunicacion del hardware, o sea, convierte los datos al formato que usa el sistema

#### MicrobitASCIIAdapter.js
* Lee los datos del puerto serial
* Interpreta el protocolo del microbit

### SimAdapter.js
* Conecta con el simulador de microbit

#### BaseAdapter.js
* Es la clase base de los adapters, o sea, todos los adapters heredan de esta clase

#### Microbit
* Aqui es donde se encuentra el "hardware" real
* Envia los datos por "serial (UART)"

#### Simulador Microbit
* La version virtual del microbit
  - jfhngn\
  - vjnnfdn

### Explicacion codigo Sketch.js

#### Definicion deeventos





## Bitácora de aplicación 



## Bitácora de reflexión

















