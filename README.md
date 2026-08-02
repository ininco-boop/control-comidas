# Control de Comidas

App para registrar el consumo diario de comidas (desayuno / almuerzo / cena) por
trabajador con su costo, cotejarlo al final del día contra la factura del
proveedor, generar reportes por trabajador/fecha, e imprimir tiquetes en una
impresora térmica vía RawBT.

---

## PASO 1 — Crear la base de datos en Firebase

1. Entra a https://console.firebase.google.com con tu cuenta de Google.
2. **Crear proyecto** → ponle un nombre (ej. `control-comidas-inninco`) → sigue
   los pasos (puedes desactivar Google Analytics, no lo necesitas).
3. En el menú lateral, entra a **Compilación → Realtime Database** →
   **Crear base de datos**.
   - Elige la región más cercana (ej. `us-central1`).
   - Empieza en **modo de prueba** (permite leer/escribir sin restricciones
     por 30 días). Más abajo te dejo reglas más estables para cuando quieras
     endurecerlo.
4. Ve a **Configuración del proyecto** (el ícono de engranaje) → pestaña
   **General** → baja hasta "Tus apps" → clic en el ícono **</>** (Web) para
   agregar una app web.
   - Ponle un apodo (ej. "Control Comidas Web") y **no** actives Firebase
     Hosting (usarás GitHub Pages).
   - Copia el objeto `firebaseConfig` que te muestra, se ve así:
     ```js
     const firebaseConfig = {
       apiKey: "AIzaSy...",
       authDomain: "control-comidas-inninco.firebaseapp.com",
       databaseURL: "https://control-comidas-inninco-default-rtdb.firebaseio.com",
       projectId: "control-comidas-inninco",
       storageBucket: "control-comidas-inninco.appspot.com",
       messagingSenderId: "123456789",
       appId: "1:123456789:web:abcdef123456"
     };
     ```
5. Abre `index.html` de esta carpeta, busca el bloque `const firebaseConfig = {`
   (cerca de la línea 760) y **reemplázalo completo** por el que copiaste.

### Reglas de la base de datos

Con el modo de prueba, cualquiera con el enlace de tu base de datos podría
leer o escribir datos, y esas reglas expiran solas a los 30 días (después
de eso, se bloquea todo). Cuando termines de probar, ve a **Realtime
Database → Reglas** y pon esto para que siga funcionando sin fecha de
vencimiento:

```json
{
  "rules": {
    ".read": true,
    ".write": true
  }
}
```

Esto no pone contraseña a nivel de base de datos (equivale al modo de
prueba, pero sin que expire). Para una protección real a nivel de servidor
haría falta Firebase Authentication, que es un paso adicional — dime si lo
quieres más adelante y lo agregamos.

---

## PASO 2 — Publicar en GitHub Pages

1. Crea un repositorio nuevo en GitHub (puede ser privado o público).
2. Sube todo el contenido de esta carpeta: `index.html`, `manifest.json`,
   `service-worker.js`, `icons/` (con los 3 PNG dentro).
3. Ve a **Settings → Pages** del repositorio → en "Build and deployment"
   elige **Deploy from a branch**, rama `main`, carpeta `/ (root)` → **Save**.
4. En un par de minutos tu app queda publicada en:
   `https://tuusuario.github.io/tu-repositorio/`
5. Ábrela desde ahí y prueba: crea tu usuario, registra un consumo, entra
   como Supervisor (usuario `Supervisor`, clave `SUPER2026` — cámbiala antes
   de compartir la app, ver más abajo) y confirma que todo funcione.

---

## PASO 3 — Generar el APK para Android

1. Ve a https://www.pwabuilder.com
2. Pega la URL de tu GitHub Pages y dale **Start**.
3. PWABuilder analiza el `manifest.json` y el `service-worker.js` — deberían
   salir en verde. Si falta algo, te dice exactamente qué corregir.
4. Ve a la pestaña **Android** → **Generate Package**.
   - Puedes dejar las opciones por defecto (Trusted Web Activity).
5. Descarga el paquete y usa el `.apk` (o `.aab` para subirlo a Play Store)
   para instalar en los dispositivos de tu equipo.

### Nota sobre la barra con el dominio de GitHub Pages

Igual que en tus otras dos apps, para que Android confíe en tu dominio y
quite la barra de direcciones necesitas publicar un archivo
`assetlinks.json` en `https://tuusuario.github.io/.well-known/assetlinks.json`
verificando la huella digital (SHA-256) del APK generado por PWABuilder
(Digital Asset Links). PWABuilder te da ese archivo listo para descargar en
el mismo paso de generar el paquete Android — solo tienes que subirlo a la
carpeta `.well-known/` de tu repositorio.

### Nota sobre la impresión con RawBT dentro del APK

El APK generado por PWABuilder es una "Trusted Web Activity" que usa Chrome
por dentro, así que el enlace `intent://...scheme=rawbt...` debería abrir
RawBT igual que en el navegador normal. Si al tocar "Imprimir" no pasa nada
dentro del APK (aunque sí funcione en Chrome), confirma que RawBT esté
instalado y configurado con tu impresora *antes* de abrir la app. Si el
problema persiste, se puede revisar la configuración de Digital Asset
Links / intents del paquete — avísame y lo vemos.

---

## Cuentas y roles

- **Trabajador**: se crea desde "+ Nuevo usuario" en la app. Solo ve y
  registra sus propios consumos. El campo "Trabajador" del formulario queda
  bloqueado con su nombre.
- **Supervisor**: cuenta fija, no se crea desde la app.
  - Usuario: `Supervisor`
  - Clave: `SUPER2026`
  - **Cámbialos** antes de usar la app con tu equipo: en `index.html`, busca
    `const SUPERVISOR_USERNAME` y `const SUPERVISOR_PASSWORD` (cerca de
    `firebaseConfig`) y pon los tuyos.
  - El supervisor ve todos los registros de todos los trabajadores, tiene
    acceso a la pestaña **Reporte** (por trabajador y fecha, exportable a
    Excel), y a los botones de impresión térmica (RawBT).

**Nivel de seguridad:** las claves se guardan en texto simple en la base de
datos, pensado para uso interno de un equipo de confianza — no es seguridad
de nivel bancario. Si más adelante quieres algo más robusto (Firebase
Authentication, claves encriptadas), se puede agregar como siguiente paso.

---

## Cómo funciona la app (resumen)

- **Hoy**: registrar consumo (trabajador + tipo de comida + costo). Al
  registrar, se imprime automáticamente (si hay RawBT configurado) un
  tiquete con todo lo que ese trabajador lleva acumulado ese día. Debajo,
  recibo del día y detalle con botón de eliminar por fila.
- **Quincena**: totales por tipo de comida y por trabajador/fecha, del
  periodo 1–15 o 16–fin de mes que elijas.
- **Reporte** (solo Supervisor): elige un rango de fechas libre, genera una
  tabla por trabajador y fecha con desayuno/almuerzo/cena/total, descarga en
  Excel o la imprime.
- Todo se sincroniza en tiempo real entre dispositivos vía Firebase Realtime
  Database, con respaldo local si se pierde la conexión.
