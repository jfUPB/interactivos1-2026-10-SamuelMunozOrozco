# Unidad 2

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

### Actividad 4
```py
from microbit import *
import utime
import music


def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()
# Para mostrar usas display.show(FILL[n]) donde n será
# un valor de 0 a 25


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


class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []
        # Personalizas el nombre del evento y la duración
        self.timer = self.createTimer("Timeout", 1000)

        self.count = 20
        self.estado_actual = None
        self.transicion_a(self.estado_config)

    def createTimer(self, event, duration):
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
        while self.event_queue:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

   # ESTADOS 
def estado_config(self, ev):
    if ev == "ENTRY":
        self.count = 20
        display.show(FILL[self.count])
    else:
        if ev == "A":
            if self.count < 25:
                self.count += 1
                display.show(FILL[self.count])
        else:
            if ev == "B":
                if self.count > 15:
                    self.count -= 1
                    display.show(FILL[self.count])
            else:
                if ev == "S":
                    self.transicion_a(self.estado_count)


def estado_count(self, ev):
    if ev == "ENTRY":
        display.show(FILL[self.count])
        self.timer.start()
    else:
        if ev == "Timeout":
            self.count -= 1
            display.show(FILL[self.count])

            if self.count == 0:
                self.transicion_a(self.estado_fin)
            else:
                self.timer.start()


def estado_fin(self, ev):
    if ev == "ENTRY":
        display.show(Image.SKULL)
        music.play(music.BA_DING)
    else:
        if ev == "A":
            self.transicion_a(self.estado_config)

task = Task()

while True:
    # Aquí generas los eventos de los botones y el gesto
    if button_a.was_pressed():
        task.post_event("A")

    if button_b.was_pressed():
        task.post_event("B")

    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    task.update()
    utime.sleep_ms(20)

```

#### Diagrama
<img width="831" height="453" alt="image" src="https://github.com/user-attachments/assets/baec7444-2298-4e33-aee5-9b0410163f89" />




### Actividad 5
#### Codigo microbit
```py
from microbit import *
import utime
import music

# IMÁGENES
def make_fill_images(on='9', off='0'):
    imgs = []
    for n in range(26):
        rows = []
        k = 0
        for y in range(5):
            row = []
            for x in range(5):
                row.append(on if k < n else off)
                k += 1
            rows.append(''.join(row))
        imgs.append(Image(':'.join(rows)))
    return imgs

FILL = make_fill_images()

# TIMER
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

# TASK / FSM
class Task:
    def __init__(self):
        self.event_queue = []
        self.timers = []

        self.timer_1s = self.createTimer("Timeout", 1000)

        self.pixeles = 20
        self.estado_actual = None
        self.transicion_a(self.estado_configuracion)

    def createTimer(self, event, duration):
        t = Timer(self, event, duration)
        self.timers.append(t)
        return t

    def post_event(self, ev):
        self.event_queue.append(ev)

    def update(self):
        for t in self.timers:
            t.update()

        while self.event_queue:
            ev = self.event_queue.pop(0)
            if self.estado_actual:
                self.estado_actual(ev)

    def transicion_a(self, nuevo_estado):
        if self.estado_actual:
            self.estado_actual("EXIT")
        self.estado_actual = nuevo_estado
        self.estado_actual("ENTRY")

    # ESTADO CONFIGURACIÓN
    def estado_configuracion(self, ev):
        if ev == "ENTRY":
            self.timer_1s.stop()
            self.pixeles = 20
            display.show(FILL[self.pixeles])
            return

        if ev == "A":
            if self.pixeles < 25:
                self.pixeles += 1
                display.show(FILL[self.pixeles])

        if ev == "B":
            if self.pixeles > 15:
                self.pixeles -= 1
                display.show(FILL[self.pixeles])

        if ev == "S":
            self.transicion_a(self.estado_conteo)

    #ESTADO CONTEO
    def estado_conteo(self, ev):
        if ev == "ENTRY":
            display.show(FILL[self.pixeles])
            self.timer_1s.start()
            return

        if ev == "Timeout":
            self.pixeles -= 1

            if self.pixeles > 0:
                display.show(FILL[self.pixeles])
                self.timer_1s.start()
            else:
                display.show(Image.SKULL)
                music.play(music.WAWAWAWAA)
                self.timer_1s.stop()

        if ev == "A":
            self.transicion_a(self.estado_configuracion)

# LOOP
task = Task()

while True:
    if button_a.was_pressed():
        task.post_event("A")
    if button_b.was_pressed():
        task.post_event("B")
    if accelerometer.was_gesture("shake"):
        task.post_event("S")

    if uart.any():
        data = uart.read(1)
        if data:
            if data == b'A':
                task.post_event("A")
            elif data == b'B':
                task.post_event("B")
            elif data == b'S':
                task.post_event("S")

    task.update()
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
  connectBtn.position(140, 300);
  connectBtn.mousePressed(connectBtnClick);
  
  let A = createButton("A");
  A.position(50, 150);
  A.size(50, 50);
  A.mousePressed(() => sendChar("A"));

  let B = createButton("B");
  B.position(200, 150);
  B.size(50, 50);
  B.mousePressed(() => sendChar("B"));

  let S = createButton("S");
  S.position(350, 150);
  S.size(50, 50);
  S.mousePressed(() => sendChar("S"));
}

function draw() {
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

function sendChar(charToSend) {
  port.write(charToSend);
}
```



## Bitácora de reflexión





