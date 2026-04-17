# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
```js
const WebSocket = require("ws");
const BaseAdapter = require("./BaseAdapter");

// Mapeo de tipo de sonido a valor X en rango -2048 a 2047
const SUPPORTED_SOUNDS = new Set(["bd", "sd", "cp", "hh"]);

// Timestamp de referencia para normalizar X
// Se establece con el primer mensaje recibido
let firstTimestamp = null;
const MAX_TIMESTAMP_WINDOW = 60000; // ventana de 60 segundos para normalizar

// Delta máximo esperado en segundos para normalizar Y
const MAX_DELTA = 2.0;

class StrudelAdapter extends BaseAdapter {
  constructor({ port = 8080, verbose = false } = {}) {
    super();
    this.port = port;
    this.verbose = verbose;
    this.wss = null;
    this.firstTimestamp = null;
  }

  async connect() {
    if (this.connected) return;

    this.wss = new WebSocket.Server({ port: this.port });

    await new Promise((resolve) => {
      this.wss.on("listening", resolve);
    });

    this.connected = true;
    this.firstTimestamp = null; // reset al conectar
    this.onConnected?.(`strudel websocket listening on ws://localhost:${this.port}`);

    this.wss.on("connection", (ws) => {
      if (this.verbose) console.log("[StrudelAdapter] Strudel connected");

      ws.on("message", (raw) => {
        try {
          const msg = JSON.parse(raw.toString("utf8"));
          const parsed = this._parseStrudelMessage(msg);
          if (parsed) this.onData?.(parsed);
        } catch (e) {
          if (this.verbose) console.warn("[StrudelAdapter] Failed to parse message:", e.message);
        }
      });

      ws.on("error", (err) => {
        if (this.verbose) console.warn("[StrudelAdapter] Client error:", err.message);
      });
    });

    this.wss.on("error", (err) => this._fail(err));
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    await new Promise((resolve, reject) => {
      this.wss.close((err) => (err ? reject(err) : resolve()));
    });

    this.wss = null;
    this.firstTimestamp = null;
    this.onDisconnected?.("strudel websocket closed");
  }

  getConnectionDetail() {
    return `strudel ws://localhost:${this.port}`;
  }

  __parseStrudelMessage(msg) {
  // Validar el nuevo contrato
  if (msg.type !== "strudel") return null;
  if (msg.payload?.eventType !== "noteEvent") return null;

  const rawSound = String(msg.payload.s ?? "").toLowerCase();
  const delta = Number(msg.payload.delta ?? 0);
  const timestamp = Number(msg.timestamp ?? 0);

  // Extraer el tipo de sonido quitando el prefijo del banco
  // "tr909bd" → "bd", "tr909sd" → "sd", "hh" → "hh"
  const sound = this._extractSound(rawSound);

  if (!SUPPORTED_SOUNDS.has(sound)) {
    if (this.verbose) console.log(`[StrudelAdapter] Ignoring unsupported sound: ${rawSound} → ${sound}`);
    return null;
  }

  // Establecer timestamp de referencia con el primer mensaje
  if (this.firstTimestamp === null) this.firstTimestamp = timestamp;

  // X = timestamp normalizado a rango -2048 a 2047
  const elapsed = timestamp - this.firstTimestamp;
  const x = Math.round(Math.min(elapsed / MAX_TIMESTAMP_WINDOW, 1) * 4095 - 2048);

  // Y = delta normalizado a rango -2048 a 2047
  const y = Math.round(Math.min(delta / MAX_DELTA, 1) * 2047);

  // btnA = true si hay cualquier sonido activo soportado
  const btnA = SUPPORTED_SOUNDS.has(sound);

  // btnB = true si es hihat
  const btnB = sound === "hh";

  if (this.verbose) {
    console.log(`[StrudelAdapter] sound=${sound} delta=${delta} ts=${timestamp} → x=${x} y=${y} btnA=${btnA} btnB=${btnB}`);
  }

  return { x, y, btnA, btnB };
}

_extractSound(rawSound) {
  // Busca si el string termina en alguno de los sonidos soportados
  // "tr909bd" → "bd", "rolandsd" → "sd", "hh" → "hh"
  for (const sound of SUPPORTED_SOUNDS) {
    if (rawSound.endsWith(sound)) return sound;
  }
  return rawSound; // si no tiene prefijo, lo devuelve tal cual
}

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }
}

module.exports = StrudelAdapter;
```
```js
const EVENTS = {
    CONNECT: "CONNECT",
    DISCONNECT: "DISCONNECT",
    DATA: "DATA",
};

// ── CONSTANTES VISUALES ──────────────────────────────────────

const SOUND_COLORS = {
    'bd': [255, 0, 80],
    'sd': [0, 200, 255],
    'cp': [0, 200, 255],
    'hh': [255, 255, 0],
};

const LATENCY_CORRECTION = 0;

// ── FSM ──────────────────────────────────────────────────────

class StrudelTask extends FSMTask {
    constructor() {
        super();

        this.rxData = {
            ready: false,
            activeAnimations: [],  // animaciones vivas que drawRunning lee
        };

        this.transitionTo(this.estado_esperando);
    }

    estado_esperando = (ev) => {
        if (ev.type === "ENTRY") {
            cursor();
            console.log("Waiting for connection...");
        } else if (ev.type === EVENTS.CONNECT) {
            this.transitionTo(this.estado_corriendo);
        }
    };

    estado_corriendo = (ev) => {
        if (ev.type === "ENTRY") {
            background(0);
            rectMode(CENTER);
            noStroke();
            console.log("Strudel ready");
            this.rxData = {
                ready: true,
                activeAnimations: [],
            };
        }
        else if (ev.type === EVENTS.DISCONNECT) {
            this.transitionTo(this.estado_esperando);
        }
        else if (ev.type === EVENTS.DATA) {
            this.updateLogic(ev.payload);
        }
        else if (ev.type === "EXIT") {
            cursor();
        }
    };

    updateLogic(data) {
        // data llega normalizado desde bridge.onData:
        // { x, y, btnA, btnB } donde:
        // x  = timestamp normalizado
        // y  = delta normalizado
        // btnA = cualquier sonido activo
        // btnB = es hihat

        if (!data.btnA) return; // no hay sonido activo

        // Recuperar el sonido y delta desde los campos normalizados
        // x mapea al tipo de sonido, y mapea al delta en ms
        const sound  = this._soundFromX(data.x);
        const deltaMs = map(data.y, -2048, 2047, 0, 2000);
        const now    = Date.now() + LATENCY_CORRECTION;

        // Agregar animación a la cola
        this.rxData.activeAnimations.push({
            startTime: now,
            duration:  deltaMs,
            type:      sound,
            x:         random(width * 0.2, width * 0.8),
            y:         random(height * 0.2, height * 0.8),
            color:     SOUND_COLORS[sound] ?? this._colorFromSound(sound),
        });
    }

    // Recupera el tipo de sonido a partir del valor X normalizado
    _soundFromX(x) {
        if (x <= -1500) return "bd";
        if (x <=     0) return "sd";
        if (x <   1500) return "hh";
        return "hh";
    }

    // Color consistente para sonidos no mapeados
    _colorFromSound(sound) {
        const code = sound.charCodeAt(0) || 0;
        return [(code * 123) % 255, (code * 456) % 255, (code * 789) % 255];
    }
}

// ── SETUP / DRAW ─────────────────────────────────────────────

let task;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
    createCanvas(windowWidth, windowHeight);
    background(0);

    task   = new StrudelTask();
    bridge = new BridgeClient();

    bridge.onConnect(() => {
        connectBtn.html("Disconnect");
        task.postEvent({ type: EVENTS.CONNECT });
    });

    bridge.onDisconnect(() => {
        connectBtn.html("Connect");
        task.postEvent({ type: EVENTS.DISCONNECT });
    });

    bridge.onStatus((s) => {
        console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
    });

    bridge.onData((data) => {
        task.postEvent({
            type: EVENTS.DATA,
            payload: {
                x:    data.x,
                y:    data.y,
                btnA: data.btnA,
                btnB: data.btnB,
            }
        });
    });

    connectBtn = createButton("Connect");
    connectBtn.position(10, 10);
    connectBtn.mousePressed(() => {
        if (bridge.isOpen) bridge.close();
        else bridge.open();
    });

    renderer.set(task.estado_corriendo, drawRunning);
}

function draw() {
    task.update();
    renderer.get(task.state)?.();
}

// ── RENDERER ─────────────────────────────────────────────────

function drawRunning() {
    const mb = task.rxData;
    if (!mb.ready) return;

    background(0, 30);

    const now = Date.now() + LATENCY_CORRECTION;

    // Recorrer animaciones de atrás hacia adelante para poder eliminar
    for (let i = mb.activeAnimations.length - 1; i >= 0; i--) {
        const anim     = mb.activeAnimations[i];
        const elapsed  = now - anim.startTime;
        const progress = elapsed / anim.duration;

        if (progress <= 1.0) {
            dibujarElemento(anim, progress);
        } else {
            mb.activeAnimations.splice(i, 1);
        }
    }
}

// ── FUNCIONES DE DIBUJO ──────────────────────────────────────

function dibujarElemento(anim, p) {
    push();
    const c = anim.color;

    switch (anim.type) {
        case "bd": dibujarBombo(p, c);        break;
        case "sd":
        case "cp": dibujarCaja(p, c);         break;
        case "hh": dibujarHat(anim, p, c);    break;
        default:   dibujarDefault(anim, p, c); break;
    }
    pop();
}

function dibujarBombo(p, c) {
    let d     = lerp(100, 600, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    noStroke();
    circle(width / 2, height / 2, d);
}

function dibujarCaja(p, c) {
    let w     = lerp(width, 0, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    noStroke();
    rect(width / 2, height / 2, w, 50);
}

function dibujarHat(anim, p, c) {
    let sz = lerp(40, 0, p);
    fill(c[0], c[1], c[2]);
    noStroke();
    rect(anim.x, anim.y, sz, sz);
}

function dibujarDefault(anim, p, c) {
    let size  = lerp(100, 0, p);
    let angle = p * TWO_PI;
    translate(anim.x, anim.y);
    rotate(angle);
    stroke(c[0], c[1], c[2]);
    strokeWeight(2);
    noFill();
    rect(0, 0, size, size);
    line(-size, 0, size, 0);
    line(0, -size, 0, size);
}

function windowResized() {
    resizeCanvas(windowWidth, windowHeight);
}
```

## Bitácora de reflexión
