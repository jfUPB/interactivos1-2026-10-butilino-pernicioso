# Unidad 1

## Bitácora de proceso de aprendizaje

### Actividad 1
-
### Activida 2
+ ¿Que es el diseño generativo?
  - el diseño generativo consiste en generar generar un sistema con unas reglas que pueden generar diferentes tipos de salidas
+ ¿Como podrias aplicar esto en tu perfil profecional?
  - consideraria que el diseño gerativo prodria adicionar una caracteristica mas imersiva a las interfaces en juegos ya que se podria hacer que dependiendo de un nivel o etapa del juego las interfaces se muestren con diferentes caracteristicas

 ### Actividad 4
_codigo  de p5.js_
```js
let port;
let connectBtn;

function setup() {
    createCanvas(400, 400);
    background(220);
    port = createSerial();
    connectBtn = createButton('Connect to micro:bit');
    connectBtn.position(80, 300);
    connectBtn.mousePressed(connectBtnClick);
}

function draw() {
        background(220);

        if (port.availableBytes() > 0) {
            let dataRx = port.read(1);
            if (dataRx == "A") {
            fill("red");
            }
        } else {
            fill("green");
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
        port.open('MicroPython', 115200);
    } else {
        port.close();
    }
}
```
_codigo  de micro:bit._
 ```py
from microbit import *

uart.init(baudrate=115200)

 while True:
     if button_a.was_pressed():
          uart.write('A')

 ```
***Porque no funciona***
notese que en el codigo de `p5.js` lee que cuando el mensaje que es mandado cuando precionado A lo pone en rojo y si no recibe un mensaje lo pone en verde cada frame, el problema es que en nuestro codigo del micro bit el usamos ` if button_a.was_pressed():` lo que significa que solo envia un mensaje que es consumido muy rapido, mientra que si si usamos ` if button_a.is_pressed():` se enviara el mansaja cada frame que el boton este precionado.

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
 
 el codigo de p5.js parte del codigo presentado en la actividad 3 pero con modificaciones, para empezar en la funcion `setup`, se elimina el boton de send love ya no se instancia aca la elipce si no que se instancia una variable llamada `xCircle` que la inicalizamos en la mitad del canva, luego en la funcion `draw` se quita los condicionales de cuando el micro:bit se sacudia y se modifico los condicionales cuando se preciona A y B para que resten 10 y sume 10 respectivamente, adicional se limita los valores de `xCircle` entre 50 y -50 para que no se salga de este valor, por ultimo se instancia aca la elipce y se usa la variable `xCircle` como el width para modificar la la pocicion en x del circulo, recordar importar la biblioteca para el micro procesador en el `html` y el microbit, solo lo tiene que poner las condiciones.

### Actividad 6
``` js

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
```
inicializamos 3 variables globales `port` `connectBtn` `connectionInitialized` las primeras 2 varibles nos permiten acceder al puerto de micro bit y la tercera es un buleano que se volvera verdaro cuando nos conectemos al puerto. en `setup` inicalizamos el canvas

```js
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

```js
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

## Bitácora de aplicación 



## Bitácora de reflexión


