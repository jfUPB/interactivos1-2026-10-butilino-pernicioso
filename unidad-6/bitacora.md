# Unidad 6

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 
codigo del brigdeServer

```js
const { WebSocketServer } = require("ws");
const StrudelAdapter = require("./adapters/StrudelAdapter");
const OpenStageControlAdapter = require("./adapters/OpenStageControlAdapter");
const MicrobitASCIIAdapter = require("./adapters/MicrobitASCIIAdapter");

const log = {
  info:  (...a) => console.log(`[${new Date().toISOString()}] [INFO]`,  ...a),
  warn:  (...a) => console.warn(`[${new Date().toISOString()}] [WARN]`,  ...a),
  error: (...a) => console.error(`[${new Date().toISOString()}] [ERROR]`, ...a),
};

function getArg(name, def = null) {
  const i = process.argv.indexOf(`--${name}`);
  if (i >= 0 && i + 1 < process.argv.length) return process.argv[i + 1];
  return def;
}

function nowMs() { return Date.now(); }

function safeJsonParse(s) {
  try { return JSON.parse(s); }
  catch (e) { log.warn("Failed to parse JSON:", s, e); return null; }
}

function broadcast(wss, obj) {
  const text = JSON.stringify(obj);
  for (const client of wss.clients) {
    if (client.readyState === 1) client.send(text);
  }
}

function status(wss, state, detail = "") {
  broadcast(wss, { type: "status", state, detail, t: nowMs() });
}

const WS_PORT = parseInt(getArg("wsPort", "8081"), 10);
const VERBOSE = process.argv.includes("--verbose");

async function main() {
  const wss = new WebSocketServer({ port: WS_PORT });
  log.info(`WS listening on ws://127.0.0.1:${WS_PORT}`);

  // ─── Adapter Strudel ──────────────────────────────────────────────────────
  const strudelAdapter = new StrudelAdapter({ verbose: VERBOSE });
  strudelAdapter.onConnected    = (d) => { log.info(`[Strudel] Connected: ${d}`); };
  strudelAdapter.onDisconnected = (d) => { log.warn(`[Strudel] Disconnected: ${d}`); };
  strudelAdapter.onError        = (d) => { log.error(`[Strudel] Error: ${d}`); };
  strudelAdapter.onData         = (msg) => {
    broadcast(wss, msg);
  };

  // ─── Adapter Open Stage Control ───────────────────────────────────────────
  const oscAdapter = new OpenStageControlAdapter({ verbose: VERBOSE });
  oscAdapter.onConnected    = (d) => { log.info(`[OSC] Connected: ${d}`); };
  oscAdapter.onDisconnected = (d) => { log.warn(`[OSC] Disconnected: ${d}`); };
  oscAdapter.onError        = (d) => { log.error(`[OSC] Error: ${d}`); };
  oscAdapter.onData         = (msg) => {
    broadcast(wss, msg);
  };

  // ─── Adapter Micro:bit ────────────────────────────────────────────────────
const microbitAdapter = new MicrobitASCIIAdapter({ 
  path: "COM15",
  verbose: VERBOSE 
  });
  microbitAdapter.onConnected    = (d) => { log.info(`[Microbit] Connected: ${d}`); };
  microbitAdapter.onDisconnected = (d) => { log.warn(`[Microbit] Disconnected: ${d}`); };
  microbitAdapter.onError        = (d) => { log.error(`[Microbit] Error: ${d}`); };
  microbitAdapter.onData         = (msg) => {
    broadcast(wss, { 
      type: "microbit",
      payload: msg
    });
  };

// Conectar con manejo de error
try {
  await microbitAdapter.connect();
  log.info("Microbit connected OK");
} catch (e) {
  log.error("Microbit failed to connect:", e.message);
}

  // ─── WebSocket server ─────────────────────────────────────────────────────
  status(wss, "ready", "bridge up (strudel + osc + microbit)");

  wss.on("connection", (ws, req) => {
    log.info(`[NETWORK] Client connected from ${req.socket.remoteAddress}`);
    ws.send(JSON.stringify({
      type: "status", state: "ready",
      detail: "bridge (strudel + osc + microbit)", t: nowMs()
    }));

    ws.on("message", async (raw) => {
      const msg = safeJsonParse(raw.toString("utf8"));
      if (!msg) return;
    });

    ws.on("close", () => {
      log.info(`[NETWORK] Client disconnected`);
    });
  });

  // ─── Conectar los tres adapters ───────────────────────────────────────────
  await strudelAdapter.connect();
  await oscAdapter.connect();
  await microbitAdapter.connect();

  log.info("All adapters connected.");
}

main().catch((e) => {
  log.error("Fatal:", e);
  process.exit(1);
});
```


strudel adapter

```js
// StrudelAdapter.js
// Recibe mensajes crudos de Strudel por WebSocket (puerto 8080),
// los normaliza y los entrega al bridgeServer para reenvío.

const { WebSocketServer } = require("ws");

const STRUDEL_PORT = 8080;

const BaseAdapter = require("./BaseAdapter");  

class StrudelAdapter extends BaseAdapter {     
  constructor({ verbose = false } = {}) {
    super();                                   
    this.verbose = verbose;
    this.onData = null;       // callback: (normalizedMsg) => void
    this.onConnected = null;
    this.onDisconnected = null;
    this.onError = null;
    this.connected = false;
    this._wss = null;
  }

  // Requerido por bridgeServer para obtener detalle de conexión
  getConnectionDetail() {
    return `strudel ws://localhost:${STRUDEL_PORT}`;
  }

  connect() {
    return new Promise((resolve) => {
      this._wss = new WebSocketServer({ port: STRUDEL_PORT });

      this._wss.on("listening", () => {
        this.connected = true;
        this.onConnected?.(`Strudel WebSocket listening on port ${STRUDEL_PORT}`);
        resolve();
      });

      this._wss.on("connection", (ws) => {
        if (this.verbose) console.log("[StrudelAdapter] Strudel conectado");

        ws.on("message", (raw) => {
          try {
            const msg = JSON.parse(raw.toString("utf8"));
            const normalized = this._normalize(msg);
            if (normalized) {
              if (this.verbose) console.log("[StrudelAdapter] Evento normalizado:", normalized);
              this.onData?.(normalized);
            }
          } catch (e) {
            this.onError?.(`Error al parsear mensaje de Strudel: ${e.message}`);
          }
        });

        ws.on("close", () => {
          if (this.verbose) console.log("[StrudelAdapter] Strudel desconectado");
        });
      });

      this._wss.on("error", (e) => {
        this.onError?.(`StrudelAdapter error: ${e.message}`);
      });
    });
  }

  disconnect() {
    return new Promise((resolve) => {
      if (this._wss) {
        this._wss.close(() => {
          this.connected = false;
          this.onDisconnected?.("StrudelAdapter cerrado");
          resolve();
        });
      } else {
        resolve();
      }
    });
  }

  // Transforma el mensaje crudo de Strudel en un objeto limpio
  _normalize(msg) {
    if (!msg || !Array.isArray(msg.args)) return null;

    // Convertir la lista plana de args a un objeto clave-valor
    const params = {};
    for (let i = 0; i < msg.args.length; i += 2) {
      params[msg.args[i]] = msg.args[i + 1];
    }

    // Verificar que tenga los datos mínimos necesarios
    if (!params.s || !msg.timestamp) return null;

    return {
      type: "strudel",
      timestamp: msg.timestamp,
      payload: {
        s: params.s,
        delta: params.delta ?? 0.25,
        cps: params.cps ?? 0.5,
        cycle: params.cycle ?? 0,
      }
    };
  }

  // No aplica para Strudel pero requerido por la interfaz del bridge
  async handleCommand(msg) {}
}

module.exports = StrudelAdapter;
```

microbit ASCI adapeter

```js

const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

class ParseError extends Error { }

function parseCsvLine(line) {
  const values = line.trim().split(",");
  if (values.length !== 4) throw new ParseError(`Expected 4 values, got ${values.length}`);

  const x = Number(values[0]);
  const y = Number(values[1]);
  const btnA = String(values[2]).trim().toLowerCase();
  const btnB = String(values[3]).trim().toLowerCase();

  if (!Number.isFinite(x) || !Number.isFinite(y)) throw new ParseError("Invalid numeric data");
  if (x < -2048 || x > 2047 || y < -2048 || y > 2047) throw new ParseError("Out of expected range");
  if (!["true", "false"].includes(btnA) || !["true", "false"].includes(btnB)) throw new ParseError("Invalid button data");

  return { x: x | 0, y: y | 0, btnA: btnA === "true", btnB: btnB === "true" };
}


class MicrobitAsciiAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.port = null;
    this.buf = "";
    this.verbose = verbose;
  }

  async connect() {
    if (this.connected) return;
    if (!this.path) throw new Error("serialPort is required for microbit device mode");

    this.port = new SerialPort({
      path: this.path,
      baudRate: this.baud,
      autoOpen: false,
    });

    await new Promise((resolve, reject) => {
      this.port.open((err) => (err ? reject(err) : resolve()));
    });

    this.connected = true;
    this.onConnected?.(`serial open ${this.path} @${this.baud}`);

    this.port.on("data", (chunk) => this._onChunk(chunk));
    this.port.on("error", (err) => this._fail(err));
    this.port.on("close", () => this._closed());
  }

  async disconnect() {
    if (!this.connected) return;
    this.connected = false;

    if (this.port && this.port.isOpen) {
      await new Promise((resolve, reject) => {
        this.port.close((err) => {
          if (err) reject(err);
          else resolve();
        });
      });
    }
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    this.buf += chunk.toString("utf8");

    let idx;
    while ((idx = this.buf.indexOf("\n")) >= 0) {
      const line = this.buf.slice(0, idx).trim();
      this.buf = this.buf.slice(idx + 1);

      if (!line) continue;

      try {
        const parsed = parseCsvLine(line);
        this.onData?.(parsed);
      } catch (e) {
        if (e instanceof ParseError) {
          if (this.verbose) console.log("Bad data:", e.message, "raw:", line);
        } else {
          this._fail(e);
        }
      }
    }

    if (this.buf.length > 4096) this.buf = "";
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = "";
    this.onDisconnected?.("serial closed (event)");
  }

  async writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this.writeLine(`LED,${x},${y},${v}\n`);
    }
  }
}

module.exports = MicrobitAsciiAdapter;

```

skecht 

```js
const EVENTS = {
  CONNECT:       "CONNECT",
  DISCONNECT:    "DISCONNECT",
  STRUDEL:       "STRUDEL",
  OSC:           "OSC",
  MICROBIT:      "MICROBIT",
  KEY_PRESSED:   "KEY_PRESSED",
  KEY_RELEASED:  "KEY_RELEASED",
};

class PainterTask extends FSMTask {
  constructor() {
    super();

    // --- Estado Strudel ---
    this.eventQueue       = [];
    this.activeAnimations = [];
    this.LATENCY_CORRECTION = 0;

    // --- Estado persistente OSC ---
    this.oscColor            = { r: 255, g: 50, b: 80 };
    this.oscScale            = 1.0;
    this.oscBackgroundActive = true;

    // --- Estado Micro:bit ---
    this.mb = {
      rawX:  0,
      rawY:  0,
      x:     0,
      y:     0,
      btnA:  false,
      btnB:  false,
      ready: false
    };

    this.transitionTo(this.estado_esperando);
  }

  // ─── ESTADOS ────────────────────────────────────────────────────────────

  estado_esperando = (ev) => {
    if (ev.type === "ENTRY") {
      cursor();
      console.log("Waiting for connection...");
    } else if (ev.type === EVENTS.CONNECT) {
      this.transitionTo(this.estado_corriendo);
    } else if (ev.type === EVENTS.OSC) {
      this.updateOsc(ev.payload);
    }
  };

  estado_corriendo = (ev) => {
    if (ev.type === "ENTRY") {
      noCursor();
      background(0);
      console.log("Sistema listo (Strudel + OSC + Microbit)");
    }

    else if (ev.type === EVENTS.DISCONNECT) {
      this.transitionTo(this.estado_esperando);
    }

    else if (ev.type === EVENTS.STRUDEL) {
      this.updateStrudel(ev.payload);
    }

    else if (ev.type === EVENTS.OSC) {
      this.updateOsc(ev.payload);
    }

    else if (ev.type === EVENTS.MICROBIT) {
      this.updateMicrobit(ev.payload);
    }

    else if (ev.type === "EXIT") {
      cursor();
    }
  };

  // ─── UPDATE STRUDEL ──────────────────────────────────────────────────────

  updateStrudel(payload) {
    this.eventQueue.push({
      timestamp: payload.timestamp,
      s:         payload.s,
      delta:     payload.delta,
      cps:       payload.cps,
    });
    this.eventQueue.sort((a, b) => a.timestamp - b.timestamp);
  }

  processStrudelQueue() {
    const now = Date.now() + this.LATENCY_CORRECTION;
    while (this.eventQueue.length > 0 && now >= this.eventQueue[0].timestamp) {
      const ev = this.eventQueue.shift();
      this.activeAnimations.push({
        startTime: ev.timestamp,
        duration:  ev.delta * 1000,
        type:      ev.s,
        x:         random(width  * 0.15, width  * 0.85),
        y:         random(height * 0.15, height * 0.85),
        baseColor: this.getColorForSound(ev.s),
      });
    }
  }

  // ─── UPDATE OSC ──────────────────────────────────────────────────────────

  updateOsc(payload) {
    const { address, args } = payload;

    if (address === "/rgb_bd") {
      this.oscColor = {
        r: constrain(Math.round(args[0]), 0, 255),
        g: constrain(Math.round(args[1]), 0, 255),
        b: constrain(Math.round(args[2]), 0, 255),
      };
    }
    else if (address === "/scale") {
      this.oscScale = map(args[0], 0, 1, 0.2, 3.0);
    }
    else if (address === "/background_toggle") {
      this.oscBackgroundActive = args[0] === 1 || args[0] === true;
    }
  }

  // ─── UPDATE MICROBIT ─────────────────────────────────────────────────────

  updateMicrobit(payload) {
    this.mb.ready = true;
    this.mb.rawX  = payload.x;
    this.mb.rawY  = payload.y;
    this.mb.btnA  = payload.btnA;
    this.mb.btnB  = payload.btnB;
  }

  // ─── COLOR POR FAMILIA ───────────────────────────────────────────────────

  getColorForSound(s) {
    if (s.includes("bd"))                     return [this.oscColor.r, this.oscColor.g, this.oscColor.b];
    if (s.includes("sd") || s.includes("cp")) return [0, 200, 255];
    if (s.includes("hh") || s.includes("oh")) return [255, 220, 0];
    const code = s.charCodeAt(0) || 0;
    return [(code * 123) % 255, (code * 456) % 255, (code * 789) % 255];
  }
}

// ─── SETUP ───────────────────────────────────────────────────────────────────

let painter;
let bridge;
let connectBtn;
const renderer = new Map();

function setup() {
  createCanvas(windowWidth, windowHeight);
  background(0);
  painter = new PainterTask();
  bridge  = new BridgeClient();

  bridge.onConnect(() => {
    connectBtn.html("Disconnect");
    painter.postEvent({ type: EVENTS.CONNECT });
  });

  bridge.onDisconnect(() => {
    connectBtn.html("Connect");
    painter.postEvent({ type: EVENTS.DISCONNECT });
  });

  bridge.onStatus((s) => {
    console.log("BRIDGE STATUS:", s.state, s.detail ?? "");
  });

  // Strudel
  bridge.onStrudel((msg) => {
    painter.postEvent({
      type: EVENTS.STRUDEL,
      payload: {
        timestamp: msg.timestamp,
        s:         msg.payload.s,
        delta:     msg.payload.delta,
        cps:       msg.payload.cps,
      }
    });
  });

  // OSC
  bridge.onOsc((msg) => {
    painter.postEvent({
      type:    EVENTS.OSC,
      payload: msg.payload
    });
  });

  // Micro:bit
  bridge.onMicrobit((msg) => {
    painter.postEvent({
      type:    EVENTS.MICROBIT,
      payload: msg.payload
    });
  });

  connectBtn = createButton("Connect");
  connectBtn.position(10, 10);
  connectBtn.mousePressed(() => {
    if (bridge.isOpen) bridge.close();
    else bridge.open();
  });

  renderer.set(painter.estado_corriendo, drawRunning);
}

// ─── DRAW ─────────────────────────────────────────────────────────────────────

function draw() {
  painter.update();
  renderer.get(painter.state)?.();
}

// ─── DRAW RUNNING ─────────────────────────────────────────────────────────────

function drawRunning() {
  // Mapear acelerómetro donde p5.js está disponible
  if (painter.mb.ready) {
    painter.mb.x = map(painter.mb.rawX, -2048, 2047, -200, 200);
    painter.mb.y = map(painter.mb.rawY, -2048, 2047, -200, 200);
  }

  if (painter.oscBackgroundActive) {
    background(0, 25);
  }

  painter.processStrudelQueue();

  const now = Date.now() + painter.LATENCY_CORRECTION;

  for (let i = painter.activeAnimations.length - 1; i >= 0; i--) {
    const anim     = painter.activeAnimations[i];
    const elapsed  = now - anim.startTime;
    const progress = elapsed / anim.duration;

    if (progress <= 1.0) {
      dibujarElemento(anim, progress, painter.oscScale);
    } else {
      painter.activeAnimations.splice(i, 1);
    }
  }

  // Cursor del Micro:bit
  if (painter.mb.ready) {
    dibujarCursorMicrobit();
  }
}

// ─── FUNCIONES DE DIBUJO ─────────────────────────────────────────────────────

function dibujarElemento(anim, p, scale = 1.0) {
  push();
  const c   = anim.baseColor;
  const mbX = painter.mb.x;
  const mbY = painter.mb.y;

  if (anim.type.includes("bd")) {
    dibujarBombo(p, c, scale, mbX, mbY);
  } else if (anim.type.includes("sd") || anim.type.includes("cp")) {
    dibujarCaja(p, c, scale, mbX, mbY);
  } else if (anim.type.includes("hh") || anim.type.includes("oh")) {
    dibujarHat(anim, p, c, scale, mbX, mbY);
  } else {
    dibujarDefault(anim, p, c, scale, mbX, mbY);
  }
  pop();
}

function dibujarBombo(p, c, scale, mbX, mbY) {
  let d     = lerp(80, 600, p) * scale;
  let alpha = lerp(255, 0, p);
  noStroke();
  fill(c[0], c[1], c[2], alpha);
  circle(width / 2 + mbX, height / 2 + mbY, d);
}

function dibujarCaja(p, c, scale, mbX, mbY) {
  let w     = lerp(width, 0, p) * scale;
  let alpha = lerp(255, 0, p);
  noStroke();
  fill(c[0], c[1], c[2], alpha);
  rectMode(CENTER);
  rect(width / 2 + mbX, height / 2 + mbY, w, 6 * scale);
}

function dibujarHat(anim, p, c, scale, mbX, mbY) {
  let sz = lerp(30, 0, p) * scale;
  noStroke();
  fill(c[0], c[1], c[2]);
  rectMode(CENTER);
  rect(anim.x + mbX, anim.y + mbY, sz, sz);
}

function dibujarDefault(anim, p, c, scale, mbX, mbY) {
  let size  = lerp(80, 0, p) * scale;
  let angle = p * TWO_PI;
  translate(anim.x + mbX, anim.y + mbY);
  rotate(angle);
  stroke(c[0], c[1], c[2]);
  strokeWeight(2);
  noFill();
  rectMode(CENTER);
  rect(0, 0, size, size);
}

function dibujarCursorMicrobit() {
  const cx = width  / 2 + painter.mb.x;
  const cy = height / 2 + painter.mb.y;

  // Cursor base siempre visible
  push();
  noFill();
  stroke(255, 255, 255, 100);
  strokeWeight(1);
  circle(cx, cy, 20);
  pop();

  // Boton A → marca blanca en la posicion actual
  if (painter.mb.btnA) {
    push();
    noStroke();
    fill(255, 255, 255, 180);
    circle(cx, cy, 40);
    pop();
  }

  // Boton B → limpia la pantalla
  if (painter.mb.btnB) {
    background(0);
  }
}

// ─── UTILS ───────────────────────────────────────────────────────────────────

function windowResized() {
  resizeCanvas(windowWidth, windowHeight);
}
```

open stage adapter

```js
// OpenStageControlAdapter.js
// Recibe mensajes OSC de Open Stage Control por UDP,
// los normaliza y los entrega a bridgeServer para reenvío por WebSocket.
//
// Protocolo: Open Stage Control envía paquetes OSC (UDP) al puerto OSC_PORT.
// Este adapter decodifica los paquetes crudos sin dependencia de librerías pesadas,
// usando parsing manual del formato OSC 1.0 (string de address + type tag string + args).

const dgram = require("dgram");

const OSC_PORT = 9000; // Puerto en el que este adapter escucha mensajes OSC

const BaseAdapter = require("./BaseAdapter"); 

class OpenStageControlAdapter extends BaseAdapter { 
  constructor({ verbose = false } = {}) {
    super();                                        
    this.verbose = verbose;
    this.onData = null;        // callback: (normalizedMsg) => void
    this.onConnected = null;
    this.onDisconnected = null;
    this.onError = null;
    this.connected = false;
    this._socket = null;
  }

  getConnectionDetail() {
    return `OSC UDP udp://0.0.0.0:${OSC_PORT}`;
  }

  connect() {
    return new Promise((resolve, reject) => {
      this._socket = dgram.createSocket("udp4");

      this._socket.on("error", (e) => {
        this.onError?.(`OpenStageControlAdapter error: ${e.message}`);
        reject(e);
      });

      this._socket.on("listening", () => {
        this.connected = true;
        const addr = this._socket.address();
        this.onConnected?.(`OSC UDP listening on ${addr.address}:${addr.port}`);
        resolve();
      });

      this._socket.on("message", (buf) => {
        console.log("[OSC RAW] Bytes recibidos:", buf.length, buf.toString("hex"));
        try {
          const msg = this._parseOSC(buf);
          console.log("[OSC PARSED]", JSON.stringify(msg));
          if (!msg) return;

          const normalized = this._normalize(msg);
          if (!normalized) return;

          if (this.verbose) {
            console.log("[OSCAdapter] Mensaje normalizado:", JSON.stringify(normalized));
          }

          this.onData?.(normalized);
        } catch (e) {
          this.onError?.(`Error al parsear OSC: ${e.message}`);
        }
      });

      this._socket.bind(OSC_PORT);
    });
  }

  disconnect() {
    return new Promise((resolve) => {
      if (this._socket) {
        this._socket.close(() => {
          this.connected = false;
          this.onDisconnected?.("OpenStageControlAdapter cerrado");
          resolve();
        });
      } else {
        resolve();
      }
    });
  }

  // ─── PARSER OSC MANUAL ────────────────────────────────────────────────────
  // Formato OSC 1.0:
  //   [address string, null-padded to múltiplo de 4]
  //   [type tag string ",iff" etc., null-padded]
  //   [argumentos en binario big-endian]

  _parseOSC(buf) {
    let offset = 0;

    // Leer address string
    const { str: address, next: o1 } = this._readString(buf, offset);
    offset = o1;

    // Leer type tag string (empieza con ',')
    const { str: typeTag, next: o2 } = this._readString(buf, offset);
    offset = o2;

    if (!typeTag.startsWith(",")) return null;
    const types = typeTag.slice(1); // quitar la coma

    // Leer argumentos según tipos
    const args = [];
    for (const t of types) {
      if (t === "i") {
        args.push(buf.readInt32BE(offset));
        offset += 4;
      } else if (t === "f") {
        args.push(buf.readFloatBE(offset));
        offset += 4;
      } else if (t === "s") {
        const { str, next } = this._readString(buf, offset);
        args.push(str);
        offset = next;
      } else if (t === "T") {
        args.push(true);
      } else if (t === "F") {
        args.push(false);
      }
      // otros tipos ignorados
    }

    return { address, args };
  }

  _readString(buf, offset) {
    let end = offset;
    while (end < buf.length && buf[end] !== 0) end++;
    const str = buf.toString("utf8", offset, end);
    // avanzar al siguiente múltiplo de 4
    const next = Math.ceil((end + 1) / 4) * 4;
    return { str, next };
  }

  // ─── NORMALIZACIÓN ────────────────────────────────────────────────────────
  // Convierte el mensaje OSC crudo en el contrato estable del sistema:
  // { type: "osc", payload: { address, args } }

  _normalize(msg) {
    if (!msg || !msg.address || !Array.isArray(msg.args)) return null;

    return {
      type: "osc",
      payload: {
        address: msg.address,
        args: msg.args
      }
    };
  }

  async handleCommand(_cmd) {}
}

module.exports = OpenStageControlAdapter;

```

fsm

```js
const ENTRY = Object.freeze({ type: "ENTRY" });
const EXIT = Object.freeze({ type: "EXIT" });

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

bridge client

```js
class BridgeClient {
  constructor(url = "ws://127.0.0.1:8081") {
    this._url = url;
    this._ws = null;
    this._isOpen = false;

    this._onData       = null;
    this._onConnect    = null;
    this._onDisconnect = null;
    this._onStatus     = null;
    this._onStrudel    = null;
    this._onOsc        = null;
    this._onMicrobit   = null;  // nuevo
  }

  get isOpen() { return this._isOpen; }

  onData(cb)       { this._onData = cb; }
  onConnect(cb)    { this._onConnect = cb; }
  onDisconnect(cb) { this._onDisconnect = cb; }
  onStatus(cb)     { this._onStatus = cb; }
  onStrudel(cb)    { this._onStrudel = cb; }
  onOsc(cb)        { this._onOsc = cb; }
  onMicrobit(cb)   { this._onMicrobit = cb; }  // nuevo

  open() {
    if (this._ws && this._ws.readyState === WebSocket.OPEN) return;
    if (this._ws) this.close();

    this._ws = new WebSocket(this._url);

    this._ws.onopen = () => {
      this._isOpen = true;
      this._onConnect?.();
    };

    this._ws.onmessage = (event) => {
      let msg;
      try { msg = JSON.parse(event.data); }
      catch (e) { console.warn("WS message is not JSON:", event.data); return; }

      // ── Despacho por tipo ──────────────────────────────────────────────
      if (msg.type === "status") {
        this._onStatus?.(msg);
        if (msg.state === "disconnected" || msg.state === "error") {
          this._isOpen = false;
          this._onDisconnect?.();
        }
        return;
      }

      if (msg.type === "microbit") {
        this._onMicrobit?.(msg);  // nuevo
        return;
      }

      if (msg.type === "strudel") {
        this._onStrudel?.(msg);
        return;
      }

      if (msg.type === "osc") {
        this._onOsc?.(msg);
        return;
      }
    };

    this._ws.onerror = (err) => {
      console.warn("WS error:", err);
    };

    this._ws.onclose = () => {
      this._isOpen = false;
      this._ws = null;
      this._onDisconnect?.();
    };
  }

  close() {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._isOpen = false;
    this._ws.close();
  }

  send(obj) {
    if (!this._ws || this._ws.readyState !== WebSocket.OPEN) return;
    this._ws.send(JSON.stringify(obj));
  }
}
```
## Bitácora de reflexión
