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

#### Codigo Visual
```js
let temporizador;
let port;
let connectBtn;
let renderer = new Map();

const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "B",
  INC: "A",
  START: "S",
  TICK: "Timeout",
}

class Temporizador extends FSMTask {
  constructor(minValue, maxValue, defaultValue) {
    super();

    this.minValue = minValue;
    this.maxValue = maxValue;
    this.defaultValue = defaultValue;
    this.configValue = defaultValue;
    this.totalSeconds = defaultValue;
    this.remainingSeconds = defaultValue;

    this.myTimer = this.addTimer(EVENTS.TICK, 1000);

    // ===== PAUSA =====
    this.paused = false;

    // ===== SECUENCIA A-B-A =====
    this.password = ["A","B","A"];
    this.sequence = [];

    this.transitionTo(this.estado_config);
  }

  get currentState() {
    return this.state;
  }

  estado_config = (ev) => {
    if (ev === ENTRY) {
      this.configValue = this.defaultValue;
    }
    else if (ev === EVENTS.DEC) {
      if (this.configValue > this.minValue) this.configValue--;
    }
    else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    }
    else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };

  estado_armed = (ev) => {

    if (ev === ENTRY) {
      this.paused = false;
      this.sequence = [];
      this.myTimer.start();
    }

    // ===== CUENTA REGRESIVA =====
    else if (ev === EVENTS.TICK && !this.paused) {

      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;

        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        }
        else {
          this.myTimer.start();
        }
      }
    }

    // ===== BOTONES DURANTE CUENTA =====
    else if (ev === EVENTS.DEC || ev === EVENTS.INC) {

      this.sequence.push(ev);

      if (this.sequence.length > 3) {
        this.sequence.shift();
      }

      // Detectar A-B-A
      if (this.sequence.join() === this.password.join()) {
        this.myTimer.stop();
        this.transitionTo(this.estado_config);
        return;
      }

      // Pausar / Reanudar con A
      if (ev === "A") {

        if (this.paused) {
          this.paused = false;
          this.myTimer.start();
        }
        else {
          this.paused = true;
          this.myTimer.stop();
        }
      }
    }

    else if (ev === EXIT) {
      this.myTimer.stop();
    }
  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    }
    else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  };
}

function setup() {
  createCanvas(windowWidth, windowHeight);

  // ===== SERIAL =====
  port = createSerial();

  connectBtn = createButton('Connect to micro:bit');
  connectBtn.position(80, 300);
  connectBtn.mousePressed(connectBtnClick);

  temporizador = new Temporizador(
    TIMER_LIMITS.min,
    TIMER_LIMITS.max,
    TIMER_LIMITS.defaultValue
  );

  textAlign(CENTER, CENTER);

  renderer.set(temporizador.estado_config, () => drawConfig(temporizador.configValue));
  renderer.set(temporizador.estado_armed, () => drawArmed(temporizador.remainingSeconds, temporizador.totalSeconds));
  renderer.set(temporizador.estado_timeout, () => drawTimeout());
}

function draw() {

  if (!port.opened()) {
    connectBtn.html('Connect to micro:bit');
  }
  else {
    connectBtn.html('Disconnect');
  }

  leerMicrobit();

  temporizador.update();
  renderer.get(temporizador.currentState)?.();
}

function drawConfig(val) {
  background(20, 40, 80);
  fill(255);
  textSize(120);
  text(val, width / 2, height / 2);
  textSize(18);
  fill(200);
  text("A(+) B(-) S(start)", width / 2, height / 2 + 100);
}

function drawArmed(val, total) {
  background(20, 20, 20);
  let pulse = sin(frameCount * 0.1) * 10;

  noFill();
  strokeWeight(20);
  stroke(255, 100, 0, 50);
  ellipse(width / 2, height / 2, 250);

  stroke(255, 150, 0);
  let angle = map(val, 0, total, 0, TWO_PI);
  arc(width / 2, height / 2, 250, 250, -HALF_PI, angle - HALF_PI);

  fill(255);
  noStroke();
  textSize(100 + pulse);
  text(val, width / 2, height / 2);
}

function drawTimeout() {
  let bg = frameCount % 20 < 10 ? color(150, 0, 0) : color(255, 0, 0);
  background(bg);
  fill(255);
  textSize(100);
  text("¡TIEMPO!", width / 2, height / 2);
}

function keyPressed() {
  if (key === "a" || key === "A") temporizador.postEvent("A");
  if (key === "b" || key === "B") temporizador.postEvent("B");
  if (key === "s" || key === "S") temporizador.postEvent("S");
}

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}

function connectBtnClick() {
  if (!port.opened()) {
    port.open('MicroPython', 115200);
  }
  else {
    port.close();
  }
}


function leerMicrobit() {

  if (port.opened()) {

    let data = port.read();

    if (data) {

      let char = data.trim();

      if (char === "A") {
        temporizador.postEvent("A");
      }
      else if (char === "B") {
        temporizador.postEvent("B");
      }
      else if (char === "S") {
        temporizador.postEvent("S");
      }

    }
  }
}
```


#### Actividad 4

## Bitácora de aplicación 



## Bitácora de reflexión









