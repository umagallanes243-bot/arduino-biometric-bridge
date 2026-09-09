# Estabilizar Arduino + AS608 y reordenar los módulos

Este plan es para tu proyecto local (`D:\Proyectos\registro-Entrada-Salida`), no para esta app de Lovable. Son cambios que aplicas en `server.js`, `sketch_arduino.ino` y el frontend.

## Por qué se siente lenta e inestable la lectura

Basado en el análisis que compartiste (SerialPort 9600 + `SoftwareSerial` en pines 2 y 3, SSE, sin reconexión descrita):

1. **SoftwareSerial a 9600 en un Nano** es la causa más común de lecturas lentas y perdidas: el sensor y el PC compiten por interrupciones, y cualquier `delay()` o `Serial.print` largo corta bytes del sensor a mitad de trama.
2. **Sin reconexión automática del puerto**: si el cable se mueve o Windows reasigna el COM, el servidor queda "vivo" pero mudo hasta reiniciarlo.
3. **Sin latido (heartbeat) ni timeouts**: no hay forma de distinguir "nadie puso el dedo" de "el sensor murió", así que la interfaz se queda esperando.
4. **SSE sin reintento ni keep-alive**: proxies y suspensión de red cortan el flujo y el navegador deja de recibir eventos.
5. **Alimentación**: el AS608 con su LED consume picos; alimentado desde el 5V del Nano por USB da lecturas erráticas.

## Cambios en el Arduino (mayor impacto)

- Subir el enlace del sensor a **57600 baud** y el enlace con el PC a **115200**. Si el Nano no aguanta SoftwareSerial estable, pasar el sensor a los pines hardware o migrar a un Nano Every / ESP32 (recomendado a mediano plazo).
- Eliminar `delay()` en el bucle principal; usar máquina de estados con `millis()` para enrolamiento y parpadeo de LEDs.
- Añadir `PING` / `PONG` cada 2 s y un mensaje `SENSOR:OK|ERROR` periódico.
- Autodiagnóstico al arranque: `verifyPassword()` en bucle con reintentos y `SENSOR:REINTENTAR` automático, no solo manual.
- Numerar los mensajes o cerrarlos con un terminador claro para descartar tramas partidas.
- Alimentar el sensor con fuente 5V separada y masa común; cable corto y trenzado.

## Cambios en el servidor (Node)

- **Gestor de puerto resiliente**: módulo `serial/manager.js` con reconexión exponencial (1 s → 15 s), detección de desconexión por evento `close`/`error`, y re-escaneo de puertos por VID/PID en vez de "COM3 por defecto".
- **Watchdog**: si no llega `PONG` en 5 s, cerrar y reabrir el puerto.
- **Cola de comandos** con `await` de respuesta y timeout por comando (evita comandos pisados durante enrolamiento).
- **Estado del hardware** expuesto en `/api/hardware/estado`: conectado, puerto, último latido, latencia.
- **SSE con keep-alive** (comentario `:ping` cada 15 s) y `retry:` para que el navegador reconecte solo.
- Buffer de eventos: si el navegador se reconecta, recibe los últimos N eventos por `Last-Event-ID`.

## Reestructuración de módulos

```text
server/
  index.js            arranque + config
  config/env.js       dotenv, puerto, baudios
  db/                 conexión, migraciones, repositorios
  serial/
    manager.js        apertura, reconexión, watchdog
    protocol.js       parseo de mensajes del Arduino
    commands.js       cola de comandos con timeout
  events/bus.js       bus interno -> SSE
  routes/
    auth.js  alumnos.js  equipos.js  movimientos.js
    reportes.js  hardware.js  stream.js
  middleware/         auth, validación (zod), rate limit, errores
  services/           reglas de negocio (entrada/salida, enrolamiento)
```

Frontend (romper el `App.jsx` de ~1700 líneas):

```text
src/
  routes/             welcome, estudiante, admin/*
  features/
    estudiante/  admin/  reportes/  hardware/
  hooks/useArduinoStream.js   (SSE con reconexión y estado)
  store/              zustand: sesión, estado hardware
  components/ui/      piezas reutilizables
```

## Orden sugerido

1. Baudios + quitar `delay()` + heartbeat en el Arduino, y alimentación separada.
2. `serial/manager.js` con reconexión, watchdog y cola de comandos.
3. SSE con keep-alive y estado de hardware visible en pantalla.
4. Separar rutas del backend en módulos.
5. Separar el frontend por rutas y features; añadir estado global.
6. Después: JWT real, validación con zod, rate limit, migraciones de BD.

## Detalles técnicos

- Reconexión: `SerialPort.list()` filtrando por `vendorId` del CH340/FTDI; reintento con backoff y jitter.
- Cola: `Map<idComando, {resolve, reject, timer}>`, timeout 3 s (25 s para enrolamiento), un comando activo a la vez.
- Watchdog: `setInterval` de 2 s que compara `Date.now() - ultimoLatido`.
- SSE: cabeceras `Cache-Control: no-cache`, `Connection: keep-alive`, `X-Accel-Buffering: no`.
- Medir antes/después: registrar el tiempo entre `dedo detectado` y `HUELLA:<id>` para tener números reales de mejora.

## Qué necesito de ti para afinar esto

- Si la inestabilidad es al leer (lento) o al mantenerse conectado (se cae solo).
- Si puedes cambiar de placa o debes quedarte con el Nano.
- Cuántos lectores/puestos hay.
