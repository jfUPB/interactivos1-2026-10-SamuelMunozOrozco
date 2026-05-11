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






## Bitácora de reflexión
