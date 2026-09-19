# Análisis de la estructura actual y plan de mejora

Este plan es para tu proyecto local (`proyecto-entrada-y-salida` en GitHub), no para esta app de Lovable.

## Qué muestra tu diagrama (diagnóstico)

El flujo confirma tres concentraciones de riesgo:

1. **`server.js` hace de todo**: autenticación, registro de alumnos, equipos, catálogos, enrolamiento de huella, movimientos, acceso manual, reportes, acceso a SQLite y puente serial con el Arduino. Ocho responsabilidades en un solo archivo.
2. **El puente serial vive dentro de ese mismo archivo**: si el puerto COM se cae, se cae dentro del mismo proceso que atiende la web, y no hay reconexión aislada.
3. **`App.jsx` concentra toda la interfaz** (panel del alumno y del vigilante), con un solo hook `useArduino.js` para el stream.

Consecuencias directas de esa estructura:

- Un error en cualquier módulo (por ejemplo el serial) puede tumbar todo el servidor.
- No se puede probar ni reiniciar la conexión al Arduino sin tocar el resto.
- La inestabilidad que sientes (lecturas lentas, conexión que "se muere") no tiene dónde diagnosticarse: no hay estado de hardware, ni latidos, ni reconexión.

## Mejora propuesta

### 1. Separar el backend por módulos (mismo `server.js` desarmado)

```text
server/
  index.js            arranque + config
  config/env.js       puerto, baudios, rutas
  db/                 conexión SQLite y repositorios
  serial/
    manager.js        apertura, reconexión, watchdog
    protocol.js       parseo de mensajes del Arduino
    commands.js       cola de comandos con timeout
  events/bus.js       bus interno -> SSE (stream al navegador)
  routes/
    auth.js  alumnos.js  equipos.js  catalogos.js
    enrolamiento.js  movimientos.js  acceso-manual.js
    reportes.js  hardware.js  stream.js
  middleware/         auth, validación, errores
```

Cada nodo ámbar de tu diagrama pasa a ser su propio archivo de rutas; el puente serial (`node_serial_bridge`) pasa a `serial/` y deja de compartir vida con las rutas web.

### 2. Estabilidad Arduino + AS608 (lo que más se nota)

- **Reconexión automática del puerto**: `serial/manager.js` reabre el COM solo, con reintentos, si el cable se mueve o Windows reasigna el puerto.
- **Watchdog con latido**: el Arduino envía `PONG` cada 2 s; si no llega en 5 s, el servidor reabre el puerto. Hoy no hay forma de distinguir "nadie puso el dedo" de "el sensor murió".
- **Cola de comandos**: un comando a la vez con timeout, para que enrolamiento y lectura no se pisen.
- **Estado visible**: nueva ruta `/api/hardware/estado` y un indicador en pantalla (conectado / reconectando / sin sensor).
- **En el sketch**: quitar `delay()` del bucle, subir baudios (sensor a 57600, PC a 115200), reintento automático de `verifyPassword()`.
- **Físico**: alimentar el AS608 con fuente 5V aparte del USB del Arduino — causa frecuente de lecturas erráticas.

### 3. Frontend ordenado por funciones

```text
src/
  routes/             panel alumno, panel vigilante, reportes
  features/           estudiante/  vigilante/  reportes/  hardware/
  hooks/useArduinoStream.js   (stream con reconexión y estado)
  components/ui/      piezas reutilizables
```

`App.jsx` se rompe en rutas; `useArduino.js` gana reconexión automática y estado del hardware.

## Orden de trabajo sugerido

1. Sketch del Arduino: quitar `delay()`, latido, reintento del sensor.
2. `serial/manager.js`: reconexión, watchdog, cola de comandos.
3. Stream con keep-alive + indicador de estado del hardware en pantalla.
4. Separar `server.js` en rutas por módulo.
5. Separar `App.jsx` por rutas y features.
6. Después: validación de datos, sesiones más robustas, migraciones de SQLite.

## Detalles técnicos

- Reconexión: re-escaneo de puertos por VID/PID del chip USB (CH340/FTDI) en vez de "COM3 fijo"; reintento con espera creciente (1 s → 15 s).
- Cola de comandos: un comando activo, timeout 3 s (25 s para enrolamiento).
- Watchdog: intervalo de 2 s comparando la hora del último `PONG`.
- Stream (SSE): cabeceras sin caché + `:ping` cada 15 s para que el navegador no pierda el flujo.
- Medir antes/después: registrar el tiempo entre "dedo detectado" y "huella identificada" para comprobar la mejora con números.

## Para afinar el plan, dime

- ¿La falla es más "tarda en leer la huella" o "se desconecta solo y hay que reiniciar"?
- ¿Puedes cambiar de placa (Nano Every / ESP32) o debes quedarte con el Nano actual?
- ¿Cuántos lectores/puestos van a funcionar a la vez?
