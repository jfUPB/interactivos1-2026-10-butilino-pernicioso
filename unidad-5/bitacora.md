# Unidad 5
## Bitácora de proceso de aprendizaje
### actividad 1

-en esta actividad pasamos del protocolo de comunicacion ascii a un botocolo binario, las diferencia del binario al ascii que mientas para codificar en ascii usabamos 16 bytes para cada paquete de datos en el binaro solo usamos 6 bytes sin, embargo el proceso de codificacion es mas combicado 

-el protocola ascii no tenia problemas con la sincronizacion de los paquetes por el "\n" marcaba por asi decirlo cuando el fin los datos el programa lo detectaba y sabai que cuando habia un "\n" era por que finalizaba una trama e iniciaba otra

-el protocolo binario no se puede usar "\n" por que puede ser interpretado como otro dato mas

## Bitácora de aplicación 
### actividad 2

adapter binario

```js
const { SerialPort } = require("serialport");
const BaseAdapter = require("./BaseAdapter");

const HEADER = 0xAA;
const PACKET_SIZE = 8;

class MicrobitBinaryAdapter extends BaseAdapter {
  constructor({ path, baud = 115200, verbose = false } = {}) {
    super();
    this.path = path;
    this.baud = baud;
    this.verbose = verbose;
    this.port = null;
    this.buf = Buffer.alloc(0);
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
        this.port.close((err) => (err ? reject(err) : resolve()));
      });
    }

    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed");
  }

  getConnectionDetail() {
    return `serial open ${this.path}`;
  }

  _onChunk(chunk) {
    // Acumular bytes entrantes en el buffer
    this.buf = Buffer.concat([this.buf, chunk]);

    // Procesar todos los paquetes completos que haya en el buffer
    while (this.buf.length >= PACKET_SIZE) {

      // Buscar el header 0xAA
      const headerIdx = this.buf.indexOf(HEADER);

      if (headerIdx === -1) {
        // No hay header en todo el buffer, descartarlo
        if (this.verbose) console.warn("[MicrobitBinary] No header found, discarding buffer");
        this.buf = Buffer.alloc(0);
        break;
      }

      if (headerIdx > 0) {
        // Hay bytes basura antes del header, descartarlos
        if (this.verbose) console.warn(`[MicrobitBinary] Discarding ${headerIdx} bytes before header`);
        this.buf = this.buf.slice(headerIdx);
      }

      // Verificar que haya suficientes bytes para un paquete completo
      if (this.buf.length < PACKET_SIZE) break;

      // Extraer el paquete
      const packet = this.buf.slice(0, PACKET_SIZE);

      // Calcular y verificar checksum: suma de bytes 1 a 6, módulo 256
      let sum = 0;
      for (let i = 1; i <= 6; i++) sum += packet[i];
      const expectedChk = sum % 256;
      const receivedChk = packet[7];

      if (expectedChk !== receivedChk) {
        console.warn(`[MicrobitBinary] Corrupted packet discarded — CHK received: ${receivedChk}, expected: ${expectedChk}`);
        // Descartar solo el header para buscar el siguiente paquete válido
        this.buf = this.buf.slice(1);
        continue;
      }

      // Parsear los campos del paquete
      const x    = packet.readInt16BE(1);
      const y    = packet.readInt16BE(3);
      const btnA = packet[5] === 1;
      const btnB = packet[6] === 1;

      this.onData?.({ x, y, btnA, btnB });

      // Avanzar el buffer al siguiente paquete
      this.buf = this.buf.slice(PACKET_SIZE);
    }

    // Prevenir que el buffer crezca indefinidamente
    if (this.buf.length > 4096) {
      console.warn("[MicrobitBinary] Buffer overflow, discarding");
      this.buf = Buffer.alloc(0);
    }
  }

  _fail(err) {
    this.onError?.(String(err?.message || err));
    this.disconnect();
  }

  _closed() {
    if (!this.connected) return;
    this.connected = false;
    this.port = null;
    this.buf = Buffer.alloc(0);
    this.onDisconnected?.("serial closed (event)");
  }

  async handleCommand(cmd) {
    if (cmd?.cmd === "setLed") {
      const x = Math.max(0, Math.min(4, Math.trunc(cmd.x)));
      const y = Math.max(0, Math.min(4, Math.trunc(cmd.y)));
      const v = Math.max(0, Math.min(9, Math.trunc(cmd.value)));
      await this._writeLine(`LED,${x},${y},${v}\n`);
    }
  }

  async _writeLine(line) {
    if (!this.port || !this.port.isOpen) return;
    await new Promise((resolve, reject) => {
      this.port.write(line, (err) => (err ? reject(err) : resolve()));
    });
  }
}

module.exports = MicrobitBinaryAdapter;
```

paquete descartado, pruba con chk incorecto
<img width="836" height="294" alt="image" src="https://github.com/user-attachments/assets/0bc49061-a088-4471-bb5e-7745d198d468" />


## Bitácora de reflexión
