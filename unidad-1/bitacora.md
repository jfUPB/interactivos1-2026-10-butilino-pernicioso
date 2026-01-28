# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 1


### Activida 2
+ ¿Que es el diseño generativo?
  - el diseño generativo consiste en generar generar un sistema con unas reglas que pueden generar diferentes tipos de salidas
+ ¿Como podrias aplicar esto en tu perfil profecional?
  - Nose
 ### Actividad 5
 
 _codigo de p5.js._
 
```js
let port;
let connectBtn;



function setup() {
    createCanvas(400, 400);
    background(220);
    xCircle = width / 2;   // posición inicial
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
}
 


function draw() {
    background(220);

    if (port.availableBytes() > 0) {
        let dataRx = port.read(1);

        if (dataRx == 'A') {
            xCircle -= 10;   // mover a la izquierda
        }
        else if(dataRx == 'B'){
            xCircle += 10; // mover a la derecha
        }
    }

    // Evita que el círculo se salga del canvas
    xCircle = constrain(xCircle, 50, width - 50);

    fill('white');
    ellipse(xCircle, height / 2, 100, 100);

    if (!port.opened()) {
        connectBtn.html('Connect to micro:bit');
    } else {
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
```
_codigo  de micro:bit._

```py
# Imports go at the top
from microbit import *

uart.init(baudrate=115200)

while True:
    if button_a.is_pressed():
        uart.write('A')
    if button_b.is_pressed():
        uart.write('B')
    
    sleep(100)
```
 ***Explicacion***
 
 el codigo de p5.js parte del codigo presentado en la actividad 3 pero con modificaciones, para empezar en la funcion `setup`, se elimina el boton de send love ya no se instancia aca la elipce si no que se instancia una variable llamada `xCircle` que la inicalizamos en la mitad del canva, luego en la funcion `draw` se quita los condicionales de cuando el micro:bit se sacudia y se modifico los condicionales cuando se preciona A y B para que resten 10 y sume 10 respectivamente, adicional se limita los valores de `xCircle` entre 50 y -50 para que no se salga de este valor, por ultimo se instancia aca la elipce y se usa la variable `xCircle` como el width
## Bitácora de aplicación 



## Bitácora de reflexión

