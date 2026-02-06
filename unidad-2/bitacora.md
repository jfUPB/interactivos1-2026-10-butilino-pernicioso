# Unidad 2

## Bitácora de proceso de aprendizaje

### actividad 1

```py

# Imports go at the top
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

class Pixel: 
    
    def __init__(self, _x,_y, _interval):
        self.event_queue = []  #recibe los eventos
        self.timers = []   #guarda los eventos
        self.myTimer = self.createTimer("Timeout",_interval) #temporizador del evento
        
        self.posX =_x
        self.posY =_y
        self.interval = _interval
        
        self.estado_actual = None  #define el estado en que inicia
        self.transicion_a(self.estado_waitInON)
        
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

#aca defino los estados 
    
    def estado_waitInON(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.posX, self.posY,9)
            self.myTimer.start()
        if ev == "Timeout":
            self.transicion_a(self.estado_waitOFF)
            
    def estado_waitOFF(self,ev):
        if ev == "ENTRY":
            display.set_pixel(self.posX, self.posY, 0)
            self.myTimer.start()
        
        if ev == "Timeout":
            self.transicion_a(self.estado_waitInON)
    



pixel1 = Pixel(0,0,1000)

while True:
    pixel1.update()
    utime.sleep_ms(20)
```
tengo que explicar este codigo :ghost:

### Actividad 2

codigo del semaforo

``` py
from microbit import *
import utime
from fsm import Timer, FSMTask, ENTRY # importamos las clases que necesitamos de la maquina de estados

class SemaForo(FSMTask): # heredamos los atributos de FSMTAsk
    def __init__(self,_x,_y,_timeInRed,_timeInGreen,_timeInYellow):
        super().__init__() # con esta linea inicializamos todod los atributos de la clase que heredamos
        self.x = _x
        self.y = _y
        self.timeInRed = _timeInRed
        self.timeInGreen = _timeInGreen
        self.timeInYellow = _timeInYellow
        self.myTimer = self.add_timer("Timeout",self.timeInRed)

        self.transition_to(self.estado_waitInRed)

    def clear(self):
        display.set_pixel(self.x,self.y,0)
        display.set_pixel(self.x,self.y+1,0)
        display.set_pixel(self.x,self.y+2,0)
        
    def estado_waitInRed(self, ev):
        if ev == ENTRY:
            self.clear()
            display.set_pixel(self.x,self.y,9)
            self.myTimer.start(self.timeInRed)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y,0)
            self.transition_to(self.estado_waitInGreen)
            

    def estado_waitInGreen(self, ev):
         if ev == ENTRY:
            self.clear()
            display.set_pixel(self.x,self.y+2,9)
            self.myTimer.start(self.timeInGreen)
         if ev == "Timeout":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)

         if ev == "A":
            display.set_pixel(self.x,self.y+2,0)
            self.transition_to(self.estado_waitInYellow)
             
    def estado_waitInYellow(self, ev):
        if ev == ENTRY:
            self.clear()
            display.set_pixel(self.x,self.y+1,9)
            self.myTimer.start(self.timeInYellow)
        if ev == "Timeout":
            display.set_pixel(self.x,self.y+1,0)
            self.transition_to(self.estado_waitInRed)

SemaForo1 = SemaForo(0,0,2000,1000,500)

while True:

    # aca procesamos la entra del evento A

    if button_a.was_pressed(): SemaForo1.post_event("A") # esto que acabamos de hacer es abstraer el uso del pres boton a, haciendolo mas trasparente para la maquina de estados
    SemaForo1.update()
    utime.sleep_ms(20)

```



cogigo de la maquina de estados

```py
import utime

ENTRY = "ENTRY"
EXIT  = "EXIT"

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
        if self.active and utime.ticks_diff(utime.ticks_ms(), self.start_time) >= self.duration:
            self.active = False
            self.owner.post_event(self.event)


class FSMTask:
    def __init__(self):
        self._q = []
        self._timers = []
        self._state = None

    def post_event(self, ev):
        self._q.append(ev)

    def add_timer(self, event, duration):
        t = Timer(self, event, duration)
        self._timers.append(t)
        return t

    def transition_to(self, new_state):
        if self._state:
            self._state(EXIT)
        self._state = new_state
        self._state(ENTRY)

    def update(self):
        for t in self._timers:
            t.update()
        while self._q:
            ev = self._q.pop(0)
            if self._state:
                self._state(ev)
```
aqui esta todo el codigo basico necesario para la maquina de estados pero en resumen la maquina espera el evento entry

## Bitácora de aplicación 



## Bitácora de reflexión

