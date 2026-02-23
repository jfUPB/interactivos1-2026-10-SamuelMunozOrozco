# Unidad 3

## Bitácora de proceso de aprendizaje
### Actividad 1
```py
#Código original

from microbit import *
import utime

class Timer:
    def __init__(self, owner, event_to_post, duration):
        self.owner = owner
        self.event = event_to_post
        self.duration = duration

        self.start_time = 0
        self.active = False

    def start(self, new_duration=None):
        if new_duration is not None:
            self.duration = new_duration
        self.start_time = utime.ticks_ms()
        self.active = True

    def stop(self):
        self.active = False

    def update(self):
        if self.active:
            if utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
                self.active = False
                self.owner.post_event(self.event)


class Semaforo:
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        self.event_queue = []
        self.timers = []
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.createTimer("Timeout",self.timeInRed)

        self.estado_actual = None
        self.transicion_a(self.estado_waitInRed)

    def createTimer(self,event,duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        # 1. Actualizar todos los timers internos automáticamente
        for t in self.timers:
            t.update()

        # 2. Procesar la cola de eventos resultante
        while len(self.event_queue) > 0:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual: self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)

    def estado_waitInRed(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transicion_a(self.estado_waitInGreen)

    def estado_waitInGreen(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)

        if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transicion_a(self.estado_waitInYellow)

        if ev == "A":
            self.transicion_a(self.estado_waitInYellow)

        if ev == "B":
            self.transicion_a(self.estado_nocturno)

    def estado_waitInYellow(self, ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transicion_a(self.estado_waitInRed)

        if ev == "B":
            self.transicion_a(self.estado_waitInRed)

    def estado_nocturno(self,ev):
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)

        if ev == "Timeout":
            pixelState = display.get_pixel(self.x,self.y+1)
            if pixelState == 9: display.set_pixel(self.x,self.y+1,0)
            else: display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)

        if ev == "A":
            self.transicion_a(self.estado_waitInRed)

semaforo1 = Semaforo(0,0,2000,1000,500)

while True:
    if button_a.was_pressed():
        semaforo1.post_event("A")
        
    if button_b.was_pressed():
        semaforo1.post_event("B")
        
    
    semaforo1.update()
    utime.sleep_ms(20)
```

### Actividad 2
```py
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music

class Temporizador(FSMTask):

    def __init__(self):
        super().__init__()

        # ========= TIMER =========
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)

        # ========= CONTROL DE PAUSA =========
        self.paused = False

        # ========= SECUENCIA A-B-A =========
        self.password = ["A","B","A"]
        self.sequence = []

        self.transition_to(self.estado_config)

    # =====================================================
    # ========= ESTADO CONFIGURACIÓN =========
    # A = Aumenta tiempo
    # B = Disminuye tiempo
    # Shake = Inicia cuenta regresiva
    # =====================================================

    def estado_config(self, ev):

        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])
            self.sequence = []

        if ev == "A":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])

        if ev == "B":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])

        if ev == "S":
            self.transition_to(self.estado_armed)


    # =====================================================
    # ========= TEMPORIZADOR CORRIENDO =========
    # A = Pausa / Reanuda
    # Secuencia A-B-A = Reinicia todo
    # =====================================================

    def estado_armed(self, ev):

        if ev == ENTRY:
            self.paused = False
            self.sequence = []
            self.myTimer.start()

        # ===== CUENTA REGRESIVA =====
        if ev == "Timeout" and not self.paused:

            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])

                if self.counter == 0:
                    self.transition_to(self.estado_timeout)
                else:
                    self.myTimer.start()


        # ===== BOTONES DURANTE CUENTA =====
        if ev == "A" or ev == "B":

            # Guardar secuencia
            self.sequence.append(ev)

            # Mantener solo últimos 3
            if len(self.sequence) > 3:
                self.sequence.pop(0)

            # Detectar A-B-A
            if self.sequence == self.password:
                self.myTimer.stop()
                self.transition_to(self.estado_config)
                return

            # Pausar o reanudar con A
            if ev == "A":
                if self.paused:
                    self.paused = False
                    self.myTimer.start()
                else:
                    self.paused = True
                    self.myTimer.stop()


    # =====================================================
    # ========= TIMEOUT =========
    # A = Reiniciar todo
    # =====================================================

    def estado_timeout(self, ev):

        if ev == ENTRY:
            display.show(Image.SKULL)
            music.play(music.FUNERAL)

        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)



temporizador = Temporizador()

# =====================================================
# ========= LOOP PRINCIPAL =========
# =====================================================

while True:

    if button_a.was_pressed():
        temporizador.post_event("A")

    if button_b.was_pressed():
        temporizador.post_event("B")

    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")

    temporizador.update()
    utime.sleep_ms(20)
```

### Actividad 3 Notas

tabulador x2 -- completa lo que se escribe o muestra las opciones que empiecen con esas letras o números

PWD -- Pad Working Director

ls -al -- Listar contenido de un directorio

clear -- Limpia la consola

flechas -- muestra historial de comandos

pw + tab x2 -- Muestra todos los comandos de una cierta letra

cd + nombre del directorio -- change directori (cambiar de directorio)

git clone + link del repositorio -- para descargar el repositorio de github

code . -- abre visual studio code 

git fetch -- para ver cambios que se hicieron en un repositorio

git pull -- actualiza los cambios en el repositorio


### Adctividad 4

#### Codigo Microbit
```py
from microbit import *
from fsm import FSMTask, ENTRY, EXIT
from utils import FILL
import utime
import music


class Temporizador(FSMTask):

    def __init__(self):
        super().__init__()

        # Timer
        self.counter = 20
        self.myTimer = self.add_timer("Timeout",1000)

        # Pausa
        self.paused = False

        # Secuencia
        self.password = ["A","B","A"]
        self.sequence = []

        self.transition_to(self.estado_config)


    # Estado de configuracion Botones
    def estado_config(self, ev):

        if ev == ENTRY:
            self.counter = 20
            display.show(FILL[self.counter])
            self.sequence = []

        if ev == "A":
            if self.counter < 25:
                self.counter += 1
            display.show(FILL[self.counter])

        if ev == "B":
            if self.counter > 15:
                self.counter -= 1
            display.show(FILL[self.counter])

        if ev == "S":
            self.transition_to(self.estado_armed)


    # Temporizador
    def estado_armed(self, ev):

        if ev == ENTRY:
            self.paused = False
            self.sequence = []
            self.myTimer.start()

        # Cuenta Regresiva
        if ev == "Timeout" and not self.paused:

            if self.counter > 0:
                self.counter -= 1
                display.show(FILL[self.counter])

                if self.counter == 0:
                    self.transition_to(self.estado_timeout)
                else:
                    self.myTimer.start()


        # Botones durante Cuenta Regresiva
        if ev == "A" or ev == "B":

            # Guardar secuencia
            self.sequence.append(ev)

            # Mantener solo últimos 3
            if len(self.sequence) > 3:
                self.sequence.pop(0)

            # Detectar A-B-A
            if self.sequence == self.password:
                self.myTimer.stop()
                self.transition_to(self.estado_config)
                return

            # Pausar o reanudar con A
            if ev == "A":
                if self.paused:
                    self.paused = False
                    self.myTimer.start()
                else:
                    self.paused = True
                    self.myTimer.stop()


    # Timeout

    def estado_timeout(self, ev):

        if ev == ENTRY:
            display.show(Image.SKULL)
            music.play(music.WAWAWAWAA)

        if ev == "A":
            music.stop()
            self.transition_to(self.estado_config)



temporizador = Temporizador()

# Loop Principal

uart.init(115200)

uart.init(115200)

while True:

    if button_a.was_pressed():
        temporizador.post_event("A")

    if button_b.was_pressed():
        temporizador.post_event("B")

    if accelerometer.was_gesture("shake"):
        temporizador.post_event("S")

    if uart.any():

        data = uart.read(1)

        if data is not None:

            char = data.decode()

            if char == "A":
                temporizador.post_event("A")

            elif char == "B":
                temporizador.post_event("B")

            elif char == "S":
                temporizador.post_event("S")

    temporizador.update()
    utime.sleep_ms(20)
```

#### Codigo p5
```js
let port;
let connectBtn;

function setup() {
  createCanvas(400, 400);
  background(220);

  port = createSerial();

  connectBtn = createButton("Connect to micro:bit");
  connectBtn.position(120, 300);
  connectBtn.mousePressed(connectBtnClick);
}

function draw() {

  if (!port.opened()) {
    connectBtn.html("Connect to micro:bit");
  } 
  else {
    connectBtn.html("Disconnect");
  }
}


// Lee El Teclado
function keyPressed(){

  if(port.opened()){

    if(key === 'A' || key === 'a'){
      port.write('A');
    }

    if(key === 'B' || key === 'b'){
      port.write('B');
    }

    if(key === 'S' || key === 's'){
      port.write('S');
    }

  }
}


function connectBtnClick() {
  if (!port.opened()) {
    port.open("MicroPython", 115200);
  } 
  else {
    port.close();
  }
}
```



## Bitácora de aplicación 



## Bitácora de reflexión








