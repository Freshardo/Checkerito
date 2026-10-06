# Tiendanube Account Manager

Aplicación de escritorio para Windows 10/11 x64. Electron + Node.js + Playwright + Chromium. Gestor independiente, sin afiliación con Tiendanube, para cuentas que el operador está autorizado a administrar.

## Instalar y abrir

1. Ejecutá `dist/TiendanubeManager.exe` y elegí la carpeta de instalación. El asistente puede crear un acceso directo.
2. Abrí **Tiendanube Account Manager**. Chromium viene incluido; no requiere Node.js, Chrome ni descargas adicionales en el equipo de destino.
3. Los datos se crean en `%APPDATA%\TiendanubeManager`. El botón **Archivos y resultados** abre esa carpeta.

El instalador no tiene firma comercial de código. Windows puede mostrar que el editor es desconocido. La aplicación no requiere permisos de administrador para instalarse por usuario.

## Importar cuentas

Usá **Importar cuentas** y seleccioná un archivo UTF-8 `.txt` con una cuenta por línea:

```text
email|password|store_url
```

También se acepta directamente el formato del archivo proporcionado:

```text
https://www.tiendanube.com/login:email:password
```

Se conserva la contraseña completa aunque incluya `:` o `|`. También se reconocen `URL: https://...:email:password` y `https://...|email|password`. La URL de origen se valida como Tiendanube, pero el navegador siempre abre el login oficial; no se navega a enlaces arbitrarios del archivo. Si la URL es el login general, la tienda aparece **Por identificar** y se registra al confirmar un panel autenticado. Si aparece una selección de tienda o una pantalla desconocida, queda para intervención manual. Una URL que apunta a una tienda `*.mitiendanube.com` permite identificar previamente su dominio.

Después de importar se muestra un informe con las líneas no reconocidas, duplicados y conflictos. Las filas cuyo usuario no es un email no se procesan. Cuando la misma cuenta tiene contraseñas distintas, queda **REQUIERE_DATOS**: corregí los duplicados en tu TXT y reimportalo. No se prueban contraseñas alternativas automáticamente. El informe contiene números de línea y motivos, sin contraseñas, y se incluye al exportar resultados.

No incluyas una fila de encabezado. En el formato `email|password|store_url`, `store_url` es el dominio de la tienda, por ejemplo `tienda.mitiendanube.com`, sin rutas. Se acepta `https://` y una barra final. En ese formato la contraseña puede contener espacios, pero no `|` ni saltos de línea. Las líneas vacías y las que comienzan con `#` se ignoran. Los registros inválidos de un archivo mixto se informan sin descartar las cuentas reconocidas.

La identidad de cada perfil se determina por email + dominio indicado en la importación. Las cuentas importadas desde el login general conservan esa identidad aunque luego se identifique su tienda. Reimportar el mismo formato actualiza la contraseña, conserva su número de perfil y proxy, y agrega las cuentas nuevas. Un conflicto de contraseñas deja la cuenta pendiente de corregir. El orden del archivo no intercambia perfiles. No se envía ninguna contraseña al dashboard. El archivo original permanece en la ubicación que elegiste; la aplicación no lo borra.

## Importar y comprobar proxies

Usá **Importar proxies**. Se aceptan archivos `.txt` y `.csv`. Formatos TXT:

```text
192.0.2.10:8080
192.0.2.11:8080:usuario:contraseña
http://usuario:contrase%C3%B1a@proxy.example:8080
https://proxy.example:8443
socks5://usuario:contrase%C3%B1a@proxy.example:1080
[2001:db8::1]:8080
```

Los ejemplos son ficticios. En URLs, codificá los caracteres especiales de usuario y contraseña, como `@` → `%40`. En el formato `host:puerto:usuario:contraseña`, se aceptan `:` dentro de la contraseña. Si no especificás protocolo, se comprueba HTTP, HTTPS y SOCKS5 hasta encontrar uno operativo. Esta detección se hace solamente contra el servicio de comprobación de IP, nunca repitiendo logins en Tiendanube. HTTPS significa conexión TLS al propio servidor proxy.

El CSV del proveedor se puede importar directamente: se lee la columna `Address:Port:Username:Password` y el protocolo de `Network_Protocol`. Las columnas `IP`, ubicación y `Status` del proveedor no sustituyen la comprobación de conectividad. Todas las proxies importadas quedan **SIN_COMPROBAR** hasta ejecutar **Comprobar proxies**. También se aceptan columnas separadas `host,port,username,password,protocol`, con delimitador coma, punto y coma o tabulación. Las comillas CSV y contraseñas con comas se conservan.

La versión 1.2 restaura las funciones de proxies y admite el estado cifrado de las versiones anteriores. Los perfiles y las cuentas se conservan. Si se usó la versión sin proxies, hay que importarlas y comprobarlas antes de procesar; las cuentas con una validación de seguridad pendiente siguen requiriendo intervención manual. Los logs con otro esquema se archivan como `activity-legacy-*.csv` sin sobrescribir su contenido.

Presioná **Comprobar proxies**. Se abre Chromium sin ventana para consultar `https://api.ipify.org?format=json` a través de cada proxy y mostrar su IP de salida, protocolo, latencia y disponibilidad. El proveedor de IP recibe la conexión desde la proxy; no se le envían cuentas ni contraseñas. Si ese proveedor no responde, la comprobación también puede fallar. Se vuelven a comprobar las proxies al iniciar una nueva sesión de la aplicación: los resultados anteriores no se consideran disponibilidad actual.

Las credenciales de proxy se conservan cifradas. Para HTTP, HTTPS y SOCKS5 autenticados, Chromium se conecta a un puente local restringido a `127.0.0.1`, que envía el tráfico a la proxy configurada. Nunca se usa una conexión directa como alternativa si la proxy falla. Las proxies rotativas del proveedor pueden cambiar su IP por sí mismas; para mantener una IP estable necesitás proxies estáticas o una sesión persistente configurada con tu proveedor.

## Procesar cuentas

1. Importá cuentas y proxies.
2. Comprobá las proxies y verificá que haya alguna disponible.
3. Presioná **Iniciar**. Se procesa una cuenta a la vez con un Chromium visible y un perfil independiente.
4. **Pausar** finaliza la cuenta actual y detiene la cola. **Reanudar** continúa las cuentas restantes. **Detener** cancela la cola; una navegación en curso puede tardar hasta su timeout en finalizar.

Las proxies disponibles se asignan por turno. Cada perfil mantiene la misma proxy. Solo hay un reemplazo automático posible antes de enviar las credenciales, si hay un fallo específicamente atribuible a conectividad de proxy y una comprobación separada lo confirma. No se reenvían credenciales automáticamente tras un fallo posterior al envío.

## Interpretar los resultados

| Estado | Significado | Acción |
| --- | --- | --- |
| PENDIENTE | Todavía no se procesó | Iniciar |
| REQUIERE_DATOS | El archivo contiene contraseñas distintas para una misma cuenta | Corregir el TXT y reimportar |
| PROCESANDO | Sesión en curso | Esperar o solicitar pausa |
| VALIDA | Se identificó navegación autenticada en el panel `/admin` de la tienda indicada | Abrir sesión |
| REQUIERE_VALIDACION | OTP, 2FA, CAPTCHA, correo, HTTP 403/429, control de seguridad o pantalla no reconocida | Intervención manual |
| NO_FUNCIONA | El sitio indicó credenciales incorrectas o cuenta/tienda no disponible | Revisar datos y reimportar si corresponde |
| ERROR_TEMPORAL | Timeout, cierre de navegador, conexión o servicio temporalmente no disponible | Revisar sesión y reintentar manualmente |

Una redirección por sí sola no demuestra acceso. Se requieren señales visibles de navegación autenticada en un panel `/admin`. Si se indicó una tienda en el archivo, debe coincidir con ella. Si no se indicó, el panel debe pertenecer a un dominio de Tiendanube reconocido para identificar la tienda. Si Tiendanube cambia su interfaz, usa un panel central con otro dominio, o exige elegir una tienda de una forma no reconocida, el resultado queda para revisión manual. Los selectores y reglas están en `src/core/browser.js`.

## Completar una validación manual

1. Ante una verificación, la cuenta pasa a **REQUIERE_VALIDACION** y la cola se pausa. El navegador queda abierto.
2. Usá **Abrir sesión** para traer su ventana al frente, o volver a abrir su perfil con la misma proxy.
3. Completá vos la verificación en Tiendanube y abrí el panel de la tienda correspondiente. Los códigos no se solicitan ni guardan en esta aplicación.
4. Presioná **Verificar sesión** para observar nuevamente el resultado. Si se reconoce el panel autenticado de la tienda, pasa a **VALIDA**. No se introduce ni se reenvía la contraseña al verificar manualmente.
5. Usá **Reanudar** para continuar las cuentas restantes.

Una cuenta en la que se detectó seguridad queda excluida de los reintentos automáticos y cambios de proxy, incluso después de reiniciar la aplicación. HTTP 403 y 429 se tratan como controles que requieren revisión, no como un fallo de red ni prueba de credenciales incorrectas. No se resuelven CAPTCHA, no se recogen códigos OTP, no se oculta automatización y no se desactivan controles del sitio.

## Reintentos manuales

Seleccioná cuentas **PENDIENTE**, **ERROR_TEMPORAL** o **NO_FUNCIONA** y presioná **Reintentar seleccionadas**. Las cuentas que tuvieron controles de seguridad se gestionan únicamente con **Abrir sesión** y **Verificar sesión**. Corregí contraseñas mediante una nueva importación antes de repetir credenciales rechazadas.

Si la proxy de una cuenta con error temporal falla por conectividad, ejecutá **Comprobar proxies** y luego **Cambiar proxy** en esa cuenta. La aplicación exige que la proxy original haya fallado y que exista otra disponible. No cambia la proxy de una cuenta con validación de seguridad.

## Archivos y privacidad

```text
%APPDATA%/TiendanubeManager/
├── profiles/
│   ├── account_001/
│   └── account_002/
├── resultados/
│   ├── validas/accounts.txt
│   ├── pendientes_validacion/accounts.txt
│   ├── no_funcionan/accounts.txt
│   ├── errores_temporales/accounts.txt
│   └── proxies_no_funcionan/proxies.txt
├── logs/activity.csv
└── state.bin
```

Los `accounts.txt` contienen CSV UTF-8 con encabezados `timestamp,account_id,email,store_url,proxy,ip,status,reason`; no son archivos de credenciales ni deben reimportarse como cuentas. Cada categoría contiene el estado más reciente: al validar una cuenta se elimina de pendientes y se agrega a válidas. `activity.csv` conserva el historial. `proxies.txt` registra exclusivamente protocolo, host y puerto de las proxies fallidas, sin autenticación.

**Exportar resultados** copia resultados y el historial a una carpeta nueva con fecha, sin perfiles ni `state.bin`. No sobrescribe exportaciones anteriores.

`state.bin` se cifra con `Electron safeStorage`, que utiliza la protección del usuario de Windows. No puede trasladarse a otro usuario/equipo como archivo portable de credenciales. Las cookies y otros datos de sesión permanecen dentro del perfil de Chromium correspondiente; tratá esas carpetas como datos privados y protegé el acceso a tu usuario de Windows. `safeStorage` cifra el estado de la aplicación, no toda la carpeta de perfiles. No se guardan capturas, trazas, respuestas del sitio ni excepciones crudas en logs. Las sesiones de cuentas distintas nunca comparten un contexto de navegador.

## Desarrollo y compilación

Requisitos: Windows x64, Node.js 24 LTS con npm y acceso a Internet para descargar las dependencias.

```powershell
npm ci
npm run browser:install
npm test
npm run test:ui
npm start
npm run dist
```

El instalador se genera en `dist/TiendanubeManager.exe`. `dist/win-unpacked/TiendanubeManager.exe` permite probar el programa sin instalar. Se distribuye la carpeta `win-unpacked` completa si se usa esa variante: el ejecutable aislado no contiene sus recursos.

Prueba del ejecutable empaquetado:

```powershell
$env:TM_SMOKE_EXE = (Resolve-Path 'dist/win-unpacked/TiendanubeManager.exe').Path
npm run test:ui
```

Las pruebas usan datos ficticios y perfiles separados. No intentan iniciar sesión con cuentas reales. `VERIFICACION.md` documenta los controles ejecutados y sus límites. Para validar contra Tiendanube se necesitan cuentas y proxies autorizadas y funcionales; los selectores del proveedor pueden cambiar.

Referencias técnicas: [aislamiento de procesos Electron](https://www.electronjs.org/docs/latest/tutorial/security), [perfiles y proxies Playwright](https://playwright.dev/docs/api/class-browsertype), [puente HTTP/HTTPS/SOCKS5](https://github.com/apify/proxy-chain).
"# Checkerito" 
