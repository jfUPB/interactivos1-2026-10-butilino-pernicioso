# Unidad 3

## Bitácora de proceso de aprendizaje
### actividad 1
```py
def estado_nocturno(self,ev): #en este estado entramos, limpiamos los pixeles y medianten una condicion prendemos y apagamos con un timeout sin usar ciclos
        if ev == "ENTRY":
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        
        if ev == "Timeout":
            if display.get_pixel(self.x,self.y+1) == 0:
               display.set_pixel(self.x,self.y+1,9)
            else:
                display.set_pixel(self.x,self.y+1,0)
                
            self.myTimer.start(self.timeInYellow)
        
        if ev == "A":
            self.transicion_a(self.estado_waitInGreen)

```

### actividad 2

```py
if ev == "A" or ev == "B":
            self.secDes.append(ev)
            if len(self.secDes) == 3:
                if self.secDes == self.very:
                    self.transition_to(self.estado_config)
            else:
                self.secDes.clear()
```

### actividad 4

```py
from microbit import *
uart.init(baudrate=115200)


while True:

    if button_a.was_pressed():
        uart.write('A')
        
    if button_b.was_pressed():
        uart.write('B')
       
    if accelerometer.was_gesture('shake'):
        uart.write('S')
       
   
```

```js
let port;
let connectBtn;

const TIMER_LIMITS = {
  min: 15,
  max: 25,
  defaultValue: 20,
};

const EVENTS = {
  DEC: "A",
  INC: "B",
  START: "S",
  TICK: "Timeout",
};

const UI = {
  dialSize: 250,
  ringWeight: 20,
  bigText: 100,
  configText: 120,
  helpText: 18,
};


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
    } else if (ev === EVENTS.INC) {
      if (this.configValue < this.maxValue) this.configValue++;
    } else if (ev === EVENTS.START) {
      this.totalSeconds = this.configValue;
      this.remainingSeconds = this.totalSeconds;
      this.transitionTo(this.estado_armed);
    }
  };


  estado_armed = (ev) => {
    if (ev === ENTRY) {
      this.myTimer.start();
    } else if (ev === EVENTS.TICK) {
      if (this.remainingSeconds > 0) {
        this.remainingSeconds--;
        if (this.remainingSeconds === 0) {
          this.transitionTo(this.estado_timeout);
        } else {
          this.myTimer.start();
        }
      }
    } else if (ev === EXIT) {
      this.myTimer.stop();
    }

  };

  estado_timeout = (ev) => {
    if (ev === ENTRY) {
      console.log("¡TIEMPO!");
    } else if (ev === EVENTS.DEC) {
      this.transitionTo(this.estado_config);
    }
  }
}

let temporizador;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  
  // --- NUEVO: Configuración serial ---
  port = createSerial();
  connectBtn = createButton("Conectar micro:bit");
  connectBtn.position(10, 10);
  connectBtn.mousePressed(() => {
    if (!port.opened()) {
      port.open("MicroPython USB REPL", 115200);
    } else {
      port.close();
    }
  });
  // -----------------------------------

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

function setup() {
  createCanvas(windowWidth, windowHeight);
  
  // --- NUEVO: Configuración serial ---
  port = createSerial();
  connectBtn = createButton("Conectar micro:bit");
  connectBtn.position(10, 10);
  connectBtn.mousePressed(() => {
    if (!port.opened()) {
      port.open('MicroPython', 115200);
    } else {
      port.close();
    }
  });
  // -----------------------------------

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
  // --- NUEVO: Leer datos seriales ---
  if (port.opened()) {
    let data = port.readUntil("\n");  // o simplemente readBytes
    if (!data) data = port.read(1);   // leer byte a byte si no hay \n
    if (data && data.length > 0) {
      let ch = data.trim().toUpperCase();
      if (ch === "A" || ch === "B" || ch === "S") {
        temporizador.postEvent(ch);
      }
    }
    // Actualizar texto del botón
    connectBtn.html(port.opened() ? "Desconectar" : "Conectar micro:bit");
  }
  // ----------------------------------

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
  text("A(-) B(+) S(start)", width / 2, height / 2 + 100);
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
```

```js
const ENTRY = "ENTRY";
const EXIT = "EXIT";

class Timer {
  constructor(owner, eventToPost, duration) {
    this.owner = owner;
    this.event = eventToPost;
    this.duration = duration;
    this.startTime = 0;
    this.active = false;
  }

  start(newDuration = null) {
    if (newDuration !== null) this.duration = newDuration;
    this.startTime = millis();
    this.active = true;
  }

  stop() {
    this.active = false;
  }

  update() {
    if (this.active && millis() - this.startTime >= this.duration) {
      this.active = false;
      this.owner.postEvent(this.event);
    }
  }
}

class FSMTask {
  constructor() {
    this.queue = [];
    this.timers = [];
    this.state = null;
  }

  postEvent(ev) {
    this.queue.push(ev);
  }

  addTimer(event, duration) {
    let t = new Timer(this, event, duration);
    this.timers.push(t);
    return t;
  }

  transitionTo(newState) {
    if (this.state) this.state(EXIT);
    this.state = newState;
    this.state(ENTRY);
  }

  update() {
    for (let t of this.timers) {
      t.update();
    }
    while (this.queue.length > 0) {
      let ev = this.queue.shift();
      if (this.state) this.state(ev);
    }
  }
}
```

```js
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0">

    <title>Sketch</title>

    <link rel="stylesheet" type="text/css" href="style.css">

    <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.js"></script>
  </head>
  
<script src="https://unpkg.com/@gohai/p5.webserial@^1/libraries/p5.webserial.js"></script>
  
  <body>
    <script src="fsm.js"></script>
    <script src="sketch.js"></script>
  </body>
</html>
```

notemos que la lista no se esta limpiando cuando pasamos de estado, por lo que cuando volvemos a este estado la lista ya esta llena, debemos agregar el el clear en el entry del estado

## Bitácora de aplicación 



## Bitácora de reflexión

