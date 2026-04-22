# 🚀 wildfly-lab

Laboratorio personal para practicar despliegues con **WildFly 39** en Windows.
Cubre instalación como servicio, despliegue de WARs, múltiples instancias, automatización y consola de administración.

> **Objetivo**: replicar en local el flujo de trabajo de un entorno PRO antes de tocarlo.

---

## 📁 Estructura del repositorio

```
wildfly-lab/
│
├── README.md
│
├── docs/
│   ├── 01-setup-service.md            ← instalar WildFly como servicio Windows
│   ├── 02-single-deployment.md        ← despliegue básico de un WAR
│   ├── 03-multi-instance.md           ← dos instancias en puertos distintos
│   ├── 04-deploy-script.md            ← script .bat de despliegue automatizado
│   ├── 05-admin-console.md            ← consola de administración (puerto 9990)
│   ├── 06-logging.md                  ← configuración de logs (size-rotating)
│   ├── 07-log-reading.md              ← lectura e interpretación de logs
│   ├── 08-ssl.md                      ← certificados SSL y HTTPS puerto 8443
│   ├── 09-datasources.md              ← datasources JDBC, pool de conexiones, JNDI
│   ├── 10-properties.md               ← variables de entorno y system-properties
│   ├── 11-backup-rollback.md          ← backup y rollback de despliegues
│   └── architecture.md               ← diagrama y descripción de arquitectura
│
├── scripts/
│   ├── deploy.bat                     ← despliegue automatizado con verificación
│   ├── deploy-versioned.bat           ← despliegue con versionado automático
│   ├── rollback.bat                   ← rollback a versión anterior
│   ├── start-instance2-service.bat    ← arrancar instancia 2 manualmente
│   └── WildFly2Service.exe            ← wrapper .exe para servicio Windows
│
├── versions\                          ← WARs versionados para rollback
│
├── config/
│   ├── instance1/
│   │   └── standalone.xml             ← configuración WildFly instancia 1
│   └── instance2/
│       └── standalone.xml             ← configuración WildFly instancia 2 (port-offset +100)
│
├── sample-app/
│   ├── mi-mini-app.war
│   └── src/
│       ├── WEB-INF/
│       │   └── web.xml
│       └── index.html
│
└── .gitignore
```

---

## 🗺️ Arquitectura del laboratorio

```
  ┌──────────────────────────────────────────────────────┐
  │                   Windows Host                       │
  │                                                      │
  │  ┌───────────────────────┐  ┌──────────────────────┐ │
  │  │  WildFly inst. 1      │  │  WildFly inst. 2     │ │
  │  │  Servicio: wildfly    │  │  Servicio: WildFly2  │ │
  │  │  HTTP  → :8080        │  │  HTTP  → :8180       │ │
  │  │  Admin → :9990        │  │  Admin → :10090      │ │
  │  └───────────────────────┘  └──────────────────────┘ │
  │           │                          │               │
  │  standalone\deployments\    standalone2\deployments\ │
  │   mi-mini-app.war            mi-mini-app.war         │
  │                                                      │
  │  ┌───────────────────────────────────────────────┐   │
  │  │  WildFly2Service.exe  (wrapper servicio Win)  │   │
  │  │  → llama a standalone.bat con base-dir inst2  │   │
  │  └───────────────────────────────────────────────┘   │
  └──────────────────────────────────────────────────────┘
```

---

## 🧭 Fases del laboratorio

| Fase | Descripción | Estado |
|------|-------------|--------|
| 1 | Instalar WildFly como servicio Windows | ✅ |
| 2 | Despliegue básico de un WAR | ✅ |
| 3 | Dos instancias en puertos distintos | ✅ |
| 4 | Script .bat automatizado | ✅ |
| 5 | Consola de administración 9990 | ✅ |
| 6 | Configuración de logs (size-rotating) | ✅ |
| 7 | Lectura e interpretación de logs | ✅ |
| 8 | Instalación de certificados SSL / HTTPS | ✅ |
| 9 | Configuración de datasources | ✅ |
| 10 | Variables de entorno y propiedades | ✅ |
| 11 | Backup y rollback de despliegues | ✅ |

---

## 🔖 Marcadores de despliegue

WildFly usa un sistema de ficheros indicadores para gestionar el ciclo de vida de cada despliegue:

| Archivo | Significado |
|---------|-------------|
| `.war.dodeploy` | Solicita despliegue — WildFly lo procesa y lo borra automáticamente |
| `.war.deployed` | ✅ Desplegado correctamente — no borrar manualmente |
| `.war.failed` | ❌ Error — abrirlo para leer la causa |
| `.war.undeployed` | Detenido explícitamente |
| `.war.isdeploying` | En proceso de despliegue (transitorio) |
| `.war.skipdeploy` | Ignorado aunque esté en la carpeta |

---

## ✅ Fase 1 — Instalar WildFly como servicio Windows

### Requisitos previos

- Java 11+ instalado y `JAVA_HOME` configurado en variables de entorno
- Ejecutar todos los comandos como Administrador

### Directorios clave

```
C:\wildfly-39.0.1.Final\
  ├── bin\
  │   ├── standalone.bat          ← arranque manual
  │   └── wildfly-service.exe     ← gestor del servicio Windows
  ├── standalone\
  │   ├── configuration\
  │   │   └── standalone.xml      ← configuración principal
  │   ├── deployments\            ← aquí van los WARs
  │   └── log\
  │       └── server.log          ← logs del servidor
  └── modules\                    ← drivers y módulos externos
```

### Registrar e iniciar el servicio

```bat
cd C:\wildfly-39.0.1.Final\bin

REM Instalar el servicio
wildfly-service.exe install

REM Configurar inicio automático
sc config wildfly start= auto

REM Arrancar
net start wildfly

REM Verificar estado
sc query wildfly
```

### Comandos de control del servicio

| Acción | Comando |
|--------|---------|
| Iniciar | `net start wildfly` |
| Detener | `net stop wildfly` |
| Reiniciar | `net stop wildfly && net start wildfly` |
| Ver estado | `sc query wildfly` |
| Desinstalar | `wildfly-service.exe uninstall` |

### Ver logs en tiempo real

```bat
REM Últimas 100 líneas
powershell "Get-Content C:\wildfly-39.0.1.Final\standalone\log\server.log -Tail 100"

REM Seguimiento en vivo (equivalente a tail -f)
powershell "Get-Content C:\wildfly-39.0.1.Final\standalone\log\server.log -Wait -Tail 50"
```

### Palabras clave a buscar en los logs

| Texto en log | Significado |
|-------------|-------------|
| `WFLYSRV0025` | Servidor arrancado y listo ✅ |
| `WFLYSRV0010` | WAR desplegado correctamente ✅ |
| `WFLYUT0021` | Despliegue iniciado |
| `WFLYUT0013` | Despliegue fallido ❌ |
| `ERROR` | Error genérico — revisar stack trace |
| `ClassNotFoundException` | Falta dependencia en el WAR |
| `PortAlreadyInUse` | Puerto 8080 ocupado por otro proceso |

### 🧠 Lecciones aprendidas — Fase 1

| Problema | Causa | Solución |
|----------|-------|----------|
| Servicio no arranca | `JAVA_HOME` no configurado | Añadir `JAVA_HOME` a variables de entorno del sistema |
| Puerto 8080 ocupado | Otro proceso usa el puerto | `netstat -ano \| findstr :8080` para identificarlo |

---

## ✅ Fase 2 — Despliegue básico de un WAR

### Estructura mínima de un WAR válido

```
mi-mini-app.war
  ├── WEB-INF/
  │   └── web.xml          ← descriptor obligatorio
  └── index.html           ← página de prueba (opcional)
```

Contenido mínimo del `web.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
             https://jakarta.ee/xml/ns/jakartaee/web-app_6_0.xsd"
         version="6.0">
  <display-name>Mi Mini App</display-name>
</web-app>
```

### Despliegue paso a paso

```bat
REM 1. Copiar el WAR a la carpeta de despliegues
copy mi-mini-app.war C:\wildfly-39.0.1.Final\standalone\deployments\

REM 2. Crear el marcador .dodeploy para forzar el despliegue
echo. > C:\wildfly-39.0.1.Final\standalone\deployments\mi-mini-app.war.dodeploy

REM 3. Verificar que aparece el marcador .deployed
dir C:\wildfly-39.0.1.Final\standalone\deployments\

REM 4. Probar en el navegador
REM    http://localhost:8080/mi-mini-app/
```

### Actualizar un WAR ya desplegado

```bat
REM 1. Copiar el nuevo WAR (sobreescribir)
copy /Y mi-mini-app-v2.war C:\wildfly-39.0.1.Final\standalone\deployments\mi-mini-app.war

REM 2. Forzar redespliegue
echo. > C:\wildfly-39.0.1.Final\standalone\deployments\mi-mini-app.war.dodeploy
```

### Verificar el estado del despliegue

```bat
REM Ver todos los marcadores activos
dir C:\wildfly-39.0.1.Final\standalone\deployments\*.deployed
dir C:\wildfly-39.0.1.Final\standalone\deployments\*.failed
```

### Personalizar el context path

Por defecto la URL es el nombre del WAR sin extensión: `mi-mini-app.war` → `/mi-mini-app/`.

Para cambiarlo añadir `WEB-INF/jboss-web.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jboss-web>
  <context-root>/mi-api-v2</context-root>
</jboss-web>
```

### 🧠 Lecciones aprendidas — Fase 2

| Problema | Causa | Solución |
|----------|-------|----------|
| WAR copiado pero no se despliega | El scanner no detectó el fichero nuevo | Crear manualmente el `.dodeploy` |
| Error 404 al acceder | Context path incorrecto | Verificar nombre del WAR o añadir `jboss-web.xml` |
| Aparece `.war.failed` | Error en el WAR o configuración | Abrir el `.failed` y revisar `server.log` |

---

## ✅ Fase 3 — Dos instancias en puertos distintos

### Concepto

Con una sola instalación de WildFly se pueden correr múltiples instancias apuntando cada una
a una carpeta `standalone` diferente. El parámetro clave es:

```
-Djboss.server.base.dir=<ruta_a_la_carpeta_standalone>
```

Cada instancia tiene su propia carpeta de deployments, logs y configuración independientes.

### Puertos

| Puerto | Instancia 1 | Instancia 2 |
|--------|-------------|-------------|
| HTTP | 8080 | 8180 |
| HTTPS | 8443 | 8543 |
| Admin | 9990 | 10090 |
| Management | 9999 | 10099 |

### Paso 1 — Copiar la carpeta standalone

```bat
xcopy /E /I /H C:\wildfly-39.0.1.Final\standalone C:\wildfly-39.0.1.Final\standalone2
```

### Paso 2 — Editar standalone.xml de la instancia 2

Abrir:
```
C:\wildfly-39.0.1.Final\standalone2\configuration\standalone.xml
```

Buscar `socket-binding-group` y cambiar el `port-offset` de `0` a `100`:

```xml
<!-- ANTES -->
<socket-binding-group name="standard-sockets"
    default-interface="public"
    port-offset="${jboss.socket.binding.port-offset:0}">

<!-- DESPUÉS -->
<socket-binding-group name="standard-sockets"
    default-interface="public"
    port-offset="${jboss.socket.binding.port-offset:100}">
```

> ⚠️ Este es el único cambio necesario en el XML. Un solo número desplaza todos los puertos +100.

### Paso 3 — Crear el script de arranque

Crear `C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat` desde CMD línea a línea:

```bat
echo @echo off > C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat
echo setlocal >> C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat
echo set WILDFLY_HOME=C:\wildfly-39.0.1.Final >> C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat
echo call "%%WILDFLY_HOME%%\bin\standalone.bat" -Djboss.server.base.dir="%%WILDFLY_HOME%%\standalone2" -Djboss.node.name=wildfly-node2 >> C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat
echo endlocal >> C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat
```

Verificar contenido:

```bat
type C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat
```

Debe mostrar exactamente:

```bat
@echo off
setlocal
set WILDFLY_HOME=C:\wildfly-39.0.1.Final
call "%WILDFLY_HOME%\bin\standalone.bat" -Djboss.server.base.dir="%WILDFLY_HOME%\standalone2" -Djboss.node.name=wildfly-node2
endlocal
```

### Paso 4 — Compilar el wrapper .exe para el servicio Windows

> ⚠️ Windows solo acepta `.exe` como `binPath` en un servicio. Los `.bat` no funcionan
> directamente porque `cmd.exe` no mantiene el proceso vivo. La solución es compilar
> un wrapper `.exe` con PowerShell sin necesidad de herramientas externas.

Abrir PowerShell como Administrador y pegar este bloque completo:

```powershell
$code = @"
using System;
using System.Diagnostics;
using System.ServiceProcess;
using System.ComponentModel;

[RunInstaller(true)]
public class WildFly2Service : ServiceBase {
    private Process process;

    static void Main() {
        ServiceBase.Run(new WildFly2Service());
    }

    public WildFly2Service() {
        this.ServiceName = "WildFly2";
        this.CanStop = true;
        this.AutoLog = true;
    }

    protected override void OnStart(string[] args) {
        ProcessStartInfo psi = new ProcessStartInfo();
        psi.FileName = "C:\\wildfly-39.0.1.Final\\bin\\standalone.bat";
        psi.Arguments = "-Djboss.server.base.dir=C:\\wildfly-39.0.1.Final\\standalone2 -Djboss.node.name=wildfly-node2";
        psi.UseShellExecute = false;
        psi.CreateNoWindow = true;
        psi.WorkingDirectory = "C:\\wildfly-39.0.1.Final\\bin";
        process = new Process();
        process.StartInfo = psi;
        process.Start();
    }

    protected override void OnStop() {
        if (process != null && !process.HasExited) {
            process.Kill();
            process.WaitForExit(5000);
        }
    }
}
"@

Add-Type -TypeDefinition $code -Language CSharp `
    -OutputAssembly "C:\wildfly-39.0.1.Final\bin\WildFly2Service.exe" `
    -OutputType WindowsApplication `
    -ReferencedAssemblies "System.ServiceProcess","System.ComponentModel"
```

### Paso 5 — Registrar el servicio Windows

En PowerShell como Administrador usar `sc.exe` (no `sc`, que en PowerShell es alias de `Set-Content`):

```powershell
sc.exe create "WildFly2" binPath= "C:\wildfly-39.0.1.Final\bin\WildFly2Service.exe" DisplayName= "WildFly Application Server_ 2" start= auto

sc.exe description "WildFly2" "Servidor WildFly instancia 2 - puerto 8180"
```

### Paso 6 — Reiniciar y verificar

> ⚠️ Si aparece el error "marcado para eliminación" es necesario reiniciar la máquina
> para que Windows limpie los servicios pendientes de borrado.

Tras el reinicio verificar que los dos servicios arrancan automáticamente:

```bat
sc query wildfly
sc query WildFly2
```

Ambos deben mostrar `STATE: 4 RUNNING`.

### Verificación final en el navegador

```
http://127.0.0.1:8080/mi-mini-app/    ← instancia 1 ✅
http://127.0.0.1:8180/mi-mini-app/    ← instancia 2 ✅
```

### Desplegar un WAR en la instancia 2

```bat
copy mi-app.war C:\wildfly-39.0.1.Final\standalone2\deployments\
echo. > C:\wildfly-39.0.1.Final\standalone2\deployments\mi-app.war.dodeploy
```

Log de la instancia 2:

```bat
powershell "Get-Content C:\wildfly-39.0.1.Final\standalone2\log\server.log -Wait -Tail 50"
```

### 🧠 Lecciones aprendidas — Fase 3

| Problema encontrado | Causa | Solución |
|--------------------|-------|----------|
| `sc` falla en PowerShell | Es alias de `Set-Content` | Usar `sc.exe` explícitamente |
| Servicio no responde con `.bat` | `cmd.exe` no mantiene el proceso vivo | Compilar wrapper `.exe` con `Add-Type` |
| Error "marcado para eliminación" | Servicio pendiente de borrado en memoria | Reiniciar la máquina |
| Conflicto de puertos al arrancar inst. 2 | Dos procesos en el mismo puerto | Configurar `port-offset=100` en `standalone.xml` |
| `.bat` creado en una sola línea | Uso incorrecto de bloque `()` en CMD | Crear fichero con `echo` línea a línea usando `>>` |

---

## ✅ Fase 4 — Script .bat automatizado de despliegue

### Concepto

Sustituir el proceso manual de crear `.dodeploy` + revisar log por un único comando
que lo hace todo y confirma si el despliegue fue bien o mal.

```
deploy.bat mi-mini-app.war 1       ← despliega en instancia 1 (puerto 8080)
deploy.bat mi-mini-app.war 2       ← despliega en instancia 2 (puerto 8180)
```

### Crear el script desde PowerShell

Abrir PowerShell como Administrador y pegar este bloque:

```powershell
$lines = @(
'@echo off',
'setlocal enabledelayedexpansion',
'set WAR=%1',
'set INSTANCIA=%2',
'set WILDFLY_HOME=C:\wildfly-39.0.1.Final',
'if "%WAR%"=="" ( echo [ERROR] Indica el nombre del WAR && exit /b 1 )',
'if "%INSTANCIA%"=="" set INSTANCIA=1',
'if "%INSTANCIA%"=="1" set DEPLOY_DIR=!WILDFLY_HOME!\standalone\deployments',
'if "%INSTANCIA%"=="1" set PUERTO=8080',
'if "%INSTANCIA%"=="2" set DEPLOY_DIR=!WILDFLY_HOME!\standalone2\deployments',
'if "%INSTANCIA%"=="2" set PUERTO=8180',
'if not exist "!DEPLOY_DIR!\!WAR!" ( echo [ERROR] No existe !WAR! en !DEPLOY_DIR! && exit /b 1 )',
'echo.',
'echo [INFO]  Instancia   : !INSTANCIA! (puerto !PUERTO!)',
'echo [INFO]  WAR         : !WAR!',
'echo [INFO]  Carpeta     : !DEPLOY_DIR!',
'echo.',
'echo [INFO]  Creando .dodeploy...',
'echo. > "!DEPLOY_DIR!\!WAR!.dodeploy"',
'echo [INFO]  .dodeploy creado OK',
'echo.',
'echo [INFO]  Esperando despliegue...',
'set CONTADOR=0',
':WAIT_LOOP',
'timeout /t 2 /nobreak >nul',
'set /a CONTADOR+=2',
'if exist "!DEPLOY_DIR!\!WAR!.deployed" goto SUCCESS',
'if exist "!DEPLOY_DIR!\!WAR!.failed"   goto FAILED',
'if !CONTADOR! geq 60 goto TIMEOUT',
'goto WAIT_LOOP',
':SUCCESS',
'echo.',
'echo [OK]    Desplegado en !CONTADOR!s',
'echo [OK]    URL: http://127.0.0.1:!PUERTO!/!WAR:.war=!/',
'echo !date! !time! OK !WAR! inst!INSTANCIA! >> "!WILDFLY_HOME!\deploy-history.log"',
'echo [LOG]   Registrado en deploy-history.log',
'goto END',
':FAILED',
'echo.',
'echo [ERROR] Despliegue fallido en !CONTADOR!s',
'echo [ERROR] Revisa: !DEPLOY_DIR!\!WAR!.failed',
'echo !date! !time! FAILED !WAR! inst!INSTANCIA! >> "!WILDFLY_HOME!\deploy-history.log"',
'goto END',
':TIMEOUT',
'echo.',
'echo [ERROR] Timeout - no respondio en 60s',
'echo !date! !time! TIMEOUT !WAR! inst!INSTANCIA! >> "!WILDFLY_HOME!\deploy-history.log"',
'goto END',
':END',
'endlocal'
)

$lines | Out-File -FilePath "C:\wildfly-39.0.1.Final\bin\deploy.bat" -Encoding ASCII
```

### Uso

Ejecutar siempre desde la carpeta donde está el WAR o indicando la ruta completa:

```bat
cd C:\wildfly-39.0.1.Final\standalone\deployments
C:\wildfly-39.0.1.Final\bin\deploy.bat mi-mini-app.war 1
C:\wildfly-39.0.1.Final\bin\deploy.bat mi-mini-app.war 2
```

### Historial de despliegues

Cada operación queda registrada en:
```
C:\wildfly-39.0.1.Final\deploy-history.log
```

```bat
type C:\wildfly-39.0.1.Final\deploy-history.log
```

### 🧠 Lecciones aprendidas — Fase 4

| Problema encontrado | Causa | Solución |
|--------------------|-------|----------|
| Variables no se expanden dentro de `if` | CMD no expande `%var%` dentro de bloques | Usar `setlocal enabledelayedexpansion` y `!var!` |
| Fichero `.bat` creado vacío o en una línea | Crear con bloque `()` en CMD | Usar PowerShell con `Out-File -Encoding ASCII` |
| Script sobreescribe el `.bat` al ejecutarse | Ejecutar el script desde su propia carpeta | Ejecutar siempre desde la carpeta del WAR |
| `copy` falla si origen y destino son iguales | El WAR ya está en `deployments` | El script solo gestiona `.dodeploy` y monitoriza — no copia |

---

## ✅ Fase 5 — Consola de administración (puerto 9990)

### Concepto

WildFly incluye una consola web de administración que permite gestionar despliegues,
monitorizar el servidor y configurar subsistemas sin tocar ficheros ni usar comandos.

En PRO es especialmente útil cuando no tienes acceso directo al sistema de ficheros del servidor.

### Acceso

| Instancia | URL consola |
|-----------|-------------|
| Instancia 1 | `http://127.0.0.1:9990` |
| Instancia 2 | `http://127.0.0.1:10090` |

### Paso 1 — Crear usuario administrador

WildFly bloquea la consola por defecto. Desde CMD como Administrador:

```bat
cd C:\wildfly-39.0.1.Final\bin
add-user.bat
```

Seguir el asistente:

```
What type of user do you wish to add?
 a) Management User  ← elegir esta
Username: admin
Password: admin123*
Groups: (dejar vacío)
Is this correct? yes
Used for AS process interconnection? no
```

Para la instancia 2:

```bat
add-user.bat --user-properties C:\wildfly-39.0.1.Final\standalone2\configuration\mgmt-users.properties --group-properties C:\wildfly-39.0.1.Final\standalone2\configuration\mgmt-groups.properties
```

### Paso 2 — Gestionar despliegues desde la consola

Desde `http://127.0.0.1:9990` → **Deployments**:

| Acción | Cómo |
|--------|------|
| Ver WARs desplegados | Listado con estado OK / FAILED |
| Deshabilitar un WAR | Clic en el WAR → Disable |
| Habilitar / redesplegar | Clic en el WAR → Enable |
| Subir un WAR nuevo | Deployments → Add → seleccionar fichero |
| Eliminar un despliegue | Clic en el WAR → Undeploy → Remove |

> ⚠️ Para subir un WAR desde la consola, el fichero debe tener extensión `.war`.
> Windows trata los `.war` como carpetas comprimidas — si el navegador no permite
> seleccionarlo, renombrarlo temporalmente a `.jar`, subirlo y cambiar el nombre
> del contexto manualmente en la consola.

### Paso 3 — Monitorizar el servidor

Desde `http://127.0.0.1:9990`:

**Memoria y JVM:**
```
Runtime → Server → JVM
```
Muestra Heap Memory, Non-Heap Memory y Thread Count.

**Estado general:**
```
Runtime → Server → Status
```
Muestra tiempo de arranque, versión de WildFly y Java.

**Subsistemas:**
```
Configuration → Subsystems
```
Permite explorar Datasources, Undertow y configuración de Logging.

### 🧠 Lecciones aprendidas — Fase 5

| Problema encontrado | Causa | Solución |
|--------------------|-------|----------|
| Consola no abre con `localhost:9990` | Fichero `hosts` de Windows con entradas comentadas | Descomentar `127.0.0.1 localhost` en `C:\Windows\System32\drivers\etc\hosts` |
| Enlace a la app abre `0.0.0.0:8080` | bind address configurado como `0.0.0.0` | El enlace es orientativo — acceder manualmente a `http://127.0.0.1:8080/mi-mini-app/` |
| WAR subido como `.war.zip` | Windows trata `.war` como carpeta comprimida | Renombrar a `.jar`, subir y cambiar el nombre del contexto en la consola |
| Error Forbidden tras reemplazar WAR | WAR sobreescrito con fichero incorrecto | Undeploy → Remove desde consola y recuperar WAR original con `.dodeploy` |

---

## ✅ Fase 6 — Configuración de logs

### Concepto

WildFly gestiona los logs a través de **handlers** (destinos de escritura) y **loggers** (categorías
y niveles de captura), configurados en el subsistema `urn:jboss:domain:logging` dentro de `standalone.xml`.

El handler por defecto (`periodic-rotating-file-handler`) rota por fecha pero no tiene límite de
tamaño ni retención. En PRO es habitual sustituirlo por `size-rotating-file-handler` para controlar
el espacio en disco.

### Tipos de handler

| Handler | Cuándo rota | Retención | Uso recomendado |
|---------|-------------|-----------|-----------------|
| `periodic-rotating-file-handler` | Cada día | ❌ Sin límite | Tráfico bajo y predecible |
| `size-rotating-file-handler` | Al alcanzar X MB | ✅ `max-backup-index` | Tráfico alto o impredecible |

### Paso 1 — Hacer copia de seguridad

```powershell
# Instancia 1
Copy-Item "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml" `
          "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml.bak"

# Instancia 2
Copy-Item "C:\wildfly-39.0.1.Final\standalone2\configuration\standalone.xml" `
          "C:\wildfly-39.0.1.Final\standalone2\configuration\standalone.xml.bak"
```

### Paso 2 — Sustituir el handler en standalone.xml

Ejecutar en PowerShell para cada instancia (cambiar la ruta de `$file` para la instancia 2):

```powershell
$file = "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml"

$old = '            <periodic-rotating-file-handler name="FILE" autoflush="true">
                <formatter>
                    <named-formatter name="PATTERN"/>
                </formatter>
                <file relative-to="jboss.server.log.dir" path="server.log"/>
                <suffix value=".yyyy-MM-dd"/>
                <append value="true"/>
            </periodic-rotating-file-handler>'

$new = '            <size-rotating-file-handler name="FILE" autoflush="true" rotate-on-boot="true">
                <formatter>
                    <named-formatter name="PATTERN"/>
                </formatter>
                <file relative-to="jboss.server.log.dir" path="server.log"/>
                <rotate-size value="50m"/>
                <max-backup-index value="5"/>
                <append value="true"/>
            </size-rotating-file-handler>'

(Get-Content $file -Raw).Replace($old, $new) | Set-Content $file -NoNewline
```

### Atributos clave del handler

| Atributo | Valor | Significado |
|---|---|---|
| `rotate-size` | `50m` | Rota cuando el fichero llega a 50 MB |
| `max-backup-index` | `5` | Conserva máximo 5 ficheros rotados (`server.log.1` … `server.log.5`) |
| `rotate-on-boot` | `true` | Crea un `server.log` limpio en cada arranque de WildFly |

Con esta configuración el disco nunca acumulará más de ~300 MB de logs por instancia (50m × 5 + el activo).

### Paso 3 — Verificar el cambio

```powershell
Select-String -Path "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml" `
              -Pattern "size-rotating|rotate-size|max-backup-index"
```

Deben aparecer las tres líneas.

### Paso 4 — Reiniciar y verificar arranque

```powershell
net stop wildfly
net start wildfly

Get-Content "C:\wildfly-39.0.1.Final\standalone\log\server.log" -Tail 20 |
Select-String "WFLYSRV0025|ERROR"
```

`WFLYSRV0025` confirma que WildFly aceptó la nueva configuración.

### Paso 5 — Comprobar la rotación

```powershell
dir C:\wildfly-39.0.1.Final\standalone\log\
```

Gracias a `rotate-on-boot="true"` debe aparecer `server.log` (activo) y `server.log.1` (rotado del arranque anterior).

### 🧠 Lecciones aprendidas — Fase 6

| Qué aprendiste | Detalle |
|---|---|
| `periodic-rotating-file-handler` | Rota por fecha pero sin límite de tamaño ni retención |
| `size-rotating-file-handler` | Rota por tamaño, controla cuántos ficheros se conservan |
| `rotate-size` | Tamaño máximo antes de rotar (`50m` = 50 MB) |
| `max-backup-index` | Número máximo de ficheros rotados conservados |
| `rotate-on-boot` | Crea un `server.log` limpio en cada arranque |
| Reemplazo con PowerShell | `(Get-Content -Raw).Replace()` para editar XML sin abrir editor |

---

## ✅ Fase 7 — Lectura e interpretación de logs

### Concepto

Saber leer el log de WildFly es la habilidad más útil en PRO. El objetivo es poder abrir
`server.log` ante cualquier incidente y determinar qué pasó, cuándo y por qué sin herramientas externas.

### Anatomía de una línea de log

```
2026-04-15 09:50:20,303 INFO  [org.jboss.as] (Controller Boot Thread) WFLYSRV0025: WildFly arrancado
│                       │      │              │                         │
│                       │      │              │                         └─ Mensaje
│                       │      │              └─ Thread que lo generó
│                       │      └─ Categoría (clase o subsistema)
│                       └─ Nivel: TRACE / DEBUG / INFO / WARN / ERROR / FATAL
└─ Timestamp
```

### Niveles de log

| Nivel | ¿Cuándo actuar? |
|---|---|
| `TRACE` / `DEBUG` | Solo en desarrollo, demasiado verboso para PRO |
| `INFO` | Normal — arranques, despliegues, operaciones correctas |
| `WARN` | Algo raro pero el servidor sigue funcionando — revisar |
| `ERROR` | Algo falló — revisar siempre |
| `FATAL` | El servidor no puede continuar — acción inmediata |

### Filtrar por nivel

```powershell
# Solo ERRORs
Get-Content "C:\wildfly-39.0.1.Final\standalone\log\server.log" |
Select-String " ERROR "

# Solo WARNs
Get-Content "C:\wildfly-39.0.1.Final\standalone\log\server.log" |
Select-String " WARN "

# ERRORs y WARNs juntos
Get-Content "C:\wildfly-39.0.1.Final\standalone\log\server.log" |
Select-String " ERROR | WARN "
```

> 💡 Los espacios alrededor de `ERROR` y `WARN` evitan falsos positivos si esas palabras
> aparecen dentro del texto del mensaje.

### Filtrar por fecha

```powershell
# Todo lo que pasó en un minuto concreto
Get-Content "C:\wildfly-39.0.1.Final\standalone\log\server.log" |
Select-String "2026-04-15 09:50"

# Todo lo que pasó en una hora concreta
Get-Content "C:\wildfly-39.0.1.Final\standalone\log\server.log" |
Select-String "2026-04-15 09:"
```

### Leer un stack trace

Un stack trace típico en WildFly tiene esta forma:

```
2026-04-15 09:50:00,000 ERROR [org.jboss.as.server] (MSC service thread)
  WFLYCTL0013: Operation failed - address: [...]
  java.lang.NullPointerException: Cannot invoke method getX() on null
        at com.miempresa.MiServlet.doGet(MiServlet.java:42)   ← TU código, línea 42
        at javax.servlet.http.HttpServlet.service(HttpServlet.java:655)
        at io.undertow.servlet.handlers.ServletHandler.handleRequest(...)
        ... 25 more
```

**Cómo leerlo en 3 pasos:**

| Paso | Dónde mirar | Qué buscas |
|---|---|---|
| 1 | Primera línea del ERROR | El código WFLY y el mensaje corto |
| 2 | Primera línea del stack trace | El tipo de excepción (`NullPointerException`, `ClassNotFoundException`…) |
| 3 | Primera línea con **tu paquete** | El fichero y la línea exacta del fallo |

> 💡 Las líneas de `javax`, `io.undertow`, `org.jboss` son del framework. Tu código aparece
> con el paquete de tu empresa. Esa es la línea que importa.

### Fases de despliegue en WildFly

Cuando un WAR falla al desplegarse, el error indica en qué fase ocurrió:

```
STRUCTURE     → ¿Es un ZIP válido? ¿Tiene WEB-INF/web.xml?
    ↓
PARSE         → ¿Los descriptores XML son válidos?
    ↓
DEPENDENCIES  → ¿Están todas las clases y librerías?
    ↓
INSTALL       → Registrar en el servidor
    ↓
VERIFY        → Comprobación final
    ↓
RUNTIME       → Arrancando la aplicación
```

Un fallo en `STRUCTURE` significa que el fichero no es un WAR válido. Un fallo en `DEPENDENCIES`
o `RUNTIME` suele indicar problemas en el código o en las dependencias de la aplicación.

### Provocar y localizar un error de despliegue

```powershell
# Crear un WAR inválido (contenido que no es un ZIP)
"esto no es un war valido" | Out-File -FilePath "C:\wildfly-39.0.1.Final\standalone\deployments\app-rota.war" -Encoding ASCII

# Forzar el despliegue
"" | Out-File -FilePath "C:\wildfly-39.0.1.Final\standalone\deployments\app-rota.war.dodeploy" -Encoding ASCII

# Verificar marcador .failed y buscar el error en el log
Get-ChildItem "C:\wildfly-39.0.1.Final\standalone\deployments\" | Where-Object { $_.Name -like "*app-rota*" }
Get-Content "C:\wildfly-39.0.1.Final\standalone\log\server.log" -Tail 40 | Select-String "ERROR|app-rota"

# Ver la causa exacta dentro del fichero .failed
Get-Content "C:\wildfly-39.0.1.Final\standalone\deployments\app-rota.war.failed"

# Limpiar
Remove-Item "C:\wildfly-39.0.1.Final\standalone\deployments\app-rota.war"
Remove-Item "C:\wildfly-39.0.1.Final\standalone\deployments\app-rota.war.failed"
```

### 🧠 Lecciones aprendidas — Fase 7

| Qué aprendiste | Detalle |
|---|---|
| Anatomía de una línea | Timestamp · Nivel · Categoría · Thread · Mensaje |
| Filtrar por nivel | `Select-String " ERROR \| WARN "` con espacios para evitar falsos positivos |
| Filtrar por fecha | `Select-String "2026-04-15 09:50"` para acotar incidentes |
| Leer un stack trace | Excepción → fase → primera línea con tu paquete |
| Fases de despliegue | STRUCTURE → PARSE → DEPENDENCIES → INSTALL → VERIFY → RUNTIME |
| Fichero `.failed` | Contiene la causa exacta, útil sin acceso directo al log |
| `echo.` no existe en PowerShell | Usar `"" \| Out-File` en su lugar |

---

## ✅ Fase 8 — Instalación de certificados SSL / HTTPS

### Concepto

WildFly sirve HTTPS en el puerto 8443 usando un keystore que contiene la clave privada y el certificado.
En LAB/PRE se usa un certificado autofirmado — el navegador avisa pero el cifrado funciona igual.
En PRO se importa un certificado firmado por una CA (DigiCert, Let's Encrypt…).

### Conceptos clave

| Concepto | ¿Qué es? |
|---|---|
| **keystore** | Fichero que guarda la clave privada y el certificado |
| **keytool** | Herramienta de Java para crear y gestionar keystores |
| **certificado autofirmado** | Lo firmas tú mismo — válido para LAB/PRE |
| **certificado de CA** | Lo firma una autoridad — válido para PRO |
| **PKCS12** | Formato moderno de keystore (`.p12`) — recomendado en WildFly 39 |

### Paso 1 — Crear el keystore con certificado autofirmado

```powershell
& "$env:JAVA_HOME\bin\keytool.exe" -genkeypair `
  -alias wildfly `
  -keyalg RSA `
  -keysize 2048 `
  -validity 365 `
  -keystore "C:\wildfly-39.0.1.Final\standalone\configuration\wildfly.keystore" `
  -storetype PKCS12 `
  -storepass wildfly123 `
  -keypass wildfly123 `
  -dname "CN=localhost, OU=LAB, O=MiEmpresa, L=Madrid, ST=Madrid, C=ES"
```

Repetir para instancia 2 cambiando la ruta a `standalone2\configuration\`.

### Paso 2 — Añadir keystore y SSL context en elytron

Localizar el bloque `<tls>` en `standalone.xml` y añadir `wildflyKS`, `wildflyKM` y `wildflySSC`
junto a los existentes `applicationKS`, `applicationKM` y `applicationSSC`:

```xml
<key-store name="wildflyKS">
    <credential-reference clear-text="wildfly123"/>
    <implementation type="PKCS12"/>
    <file path="wildfly.keystore" relative-to="jboss.server.config.dir"/>
</key-store>

<key-manager name="wildflyKM" key-store="wildflyKS">
    <credential-reference clear-text="wildfly123"/>
</key-manager>

<server-ssl-context name="wildflySSC" key-manager="wildflyKM"/>
```

### Paso 3 — Activar el https-listener en Undertow

Cambiar `applicationSSC` por `wildflySSC` en el listener HTTPS:

```powershell
$file = "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml"
$old = '                <https-listener name="https" socket-binding="https" ssl-context="applicationSSC" enable-http2="true"/>'
$new = '                <https-listener name="https" socket-binding="https" ssl-context="wildflySSC" enable-http2="true"/>'
(Get-Content $file -Raw).Replace($old, $new) | Set-Content $file -NoNewline
```

### Verificación

```
https://127.0.0.1:8443/mi-mini-app/    ← instancia 1 ✅
https://127.0.0.1:8543/mi-mini-app/    ← instancia 2 ✅
```

El navegador mostrará aviso de seguridad — normal con certificado autofirmado.
Avanzado → Continuar para acceder.

### 🧠 Lecciones aprendidas — Fase 8

| Qué aprendiste | Detalle |
|---|---|
| `keytool` | Viene incluido con el JDK — usar con ruta completa `$env:JAVA_HOME\bin\keytool.exe` |
| Keystore PKCS12 | Formato moderno recomendado en WildFly 39, sustituye al antiguo JKS |
| `CN=localhost` | El Common Name debe coincidir con el hostname — en PRO va el FQDN real |
| `key-store` → `key-manager` → `server-ssl-context` | Cadena de configuración SSL en elytron |
| `https-listener` en Undertow | Punto de entrada HTTPS — apunta al SSL context |
| Puerto 8443 / 8543 | HTTPS instancia 1 / instancia 2 (port-offset +100) |
| Certificado de CA en PRO | Mismo proceso pero con `-importcert` en lugar de `-genkeypair` |

---

## ✅ Fase 9 — Configuración de datasources

### Concepto

Un datasource es la configuración de conexión a base de datos gestionada por WildFly.
La aplicación no se conecta directamente a la BD — le pide la conexión a WildFly via JNDI.
WildFly mantiene un pool de conexiones abiertas y las reutiliza.

### Conceptos clave

| Concepto | ¿Qué es? |
|---|---|
| **JDBC** | Driver Java para conectarse a una base de datos |
| **Datasource** | Configuración de conexión a BD gestionada por WildFly |
| **JNDI** | Sistema de nombres — la app busca el datasource por nombre (`java:/MiDS`) |
| **Pool de conexiones** | WildFly mantiene conexiones abiertas y las reutiliza |
| **H2** | Base de datos embebida en Java — incluida en WildFly, ideal para LAB |

### Datasource H2 por defecto

WildFly incluye `ExampleDS` con H2 en memoria:

```xml
<datasource jndi-name="java:jboss/datasources/ExampleDS" pool-name="ExampleDS" ...>
    <connection-url>jdbc:h2:mem:test;DB_CLOSE_DELAY=-1;...</connection-url>
    <driver>h2</driver>
    <security user-name="sa" password="sa"/>
</datasource>
```

> ⚠️ `jdbc:h2:mem:` — los datos viven en memoria y se pierden al parar WildFly.

### Añadir datasource H2 persistente (LabDS)

```powershell
$file = "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml"

# Añadir LabDS justo después del cierre de ExampleDS
$old = '                </datasource>
                <drivers>'

$new = '                </datasource>
                <datasource jndi-name="java:jboss/datasources/LabDS" pool-name="LabDS" enabled="true" use-java-context="true">
                    <connection-url>jdbc:h2:file:${jboss.server.data.dir}/labdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE</connection-url>
                    <driver>h2</driver>
                    <pool>
                        <min-pool-size>2</min-pool-size>
                        <max-pool-size>10</max-pool-size>
                        <prefill>true</prefill>
                    </pool>
                    <security user-name="sa" password="sa"/>
                    <validation>
                        <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.novendor.JDBC4ValidConnectionChecker"/>
                        <background-validation>true</background-validation>
                    </validation>
                </datasource>
                <drivers>'

(Get-Content $file -Raw).Replace($old, $new) | Set-Content $file -NoNewline
```

### Atributos del pool

| Atributo | Valor | Significado |
|---|---|---|
| `min-pool-size` | 2 | Conexiones mínimas abiertas siempre |
| `max-pool-size` | 10 | Conexiones máximas simultáneas |
| `prefill` | true | Abre las conexiones mínimas al arrancar |
| `background-validation` | true | Comprueba periódicamente que las conexiones siguen vivas |
| `${jboss.server.data.dir}` | `standalone\data\` | WildFly resuelve la ruta automáticamente |

### Verificación

```
http://127.0.0.1:9990
Configuration → Subsystems → Datasources & Drivers → Datasources
LabDS → Test Connection ✅
```

```powershell
# El fichero de BD persistente debe existir en disco
dir "C:\wildfly-39.0.1.Final\standalone\data\" | Where-Object { $_.Name -like "*labdb*" }
```

### 🧠 Lecciones aprendidas — Fase 9

| Qué aprendiste | Detalle |
|---|---|
| `ExampleDS` | Datasource H2 en memoria incluido por defecto — datos no persisten |
| `jdbc:h2:file:` | H2 persistente en disco — los datos sobreviven al reinicio |
| `${jboss.server.data.dir}` | Variable de WildFly que resuelve a `standalone\data\` |
| `min-pool-size` / `max-pool-size` | Controlan cuántas conexiones mantiene el pool |
| `Test Connection` en consola | Forma rápida de verificar que el datasource funciona |
| Puerto ocupado por proceso zombie | `netstat -ano \| findstr :PUERTO` → `taskkill /PID X /F` |

---

## ✅ Fase 10 — Variables de entorno y propiedades

### Concepto

Las `system-properties` permiten parametrizar WildFly para que el mismo WAR funcione en LAB, PRE
y PRO sin recompilar — solo cambiando propiedades en el servidor. El código Java las lee con
`System.getProperty("nombre")`.

### Añadir system-properties en standalone.xml

El bloque va justo después de `</extensions>`:

```powershell
$file = "C:\wildfly-39.0.1.Final\standalone\configuration\standalone.xml"

$old = '    </extensions>'

$new = '    </extensions>

    <system-properties>
        <property name="app.env" value="LAB"/>
        <property name="app.version" value="1.0"/>
        <property name="db.jndi" value="java:jboss/datasources/LabDS"/>
        <property name="app.log.level" value="DEBUG"/>
    </system-properties>'

(Get-Content $file -Raw).Replace($old, $new) | Set-Content $file -NoNewline
```

### Propiedades configuradas

| Propiedad | Valor LAB | Uso |
|---|---|---|
| `app.env` | `LAB` | Identifica el entorno — en PRO valdría `PRO` |
| `app.version` | `1.0` | Versión de la app desplegada |
| `db.jndi` | `java:jboss/datasources/LabDS` | JNDI del datasource — cambia por entorno |
| `app.log.level` | `DEBUG` | Nivel de log — en PRO sería `WARN` |

### Leer y modificar propiedades via CLI

```powershell
# Leer una propiedad
& "C:\wildfly-39.0.1.Final\bin\jboss-cli.bat" --connect --command="/system-property=app.env:read-resource"

# Modificar en caliente (sin reiniciar)
& "C:\wildfly-39.0.1.Final\bin\jboss-cli.bat" --connect --command="/system-property=app.env:write-attribute(name=value,value=PRE)"
```

> 💡 `write-attribute` via CLI aplica el cambio en caliente — sin reiniciar WildFly ni cortar el servicio.

### Verificación en consola

```
http://127.0.0.1:9990
Configuration → System Properties
```

Deben aparecer las 4 propiedades listadas.

### 🧠 Lecciones aprendidas — Fase 10

| Qué aprendiste | Detalle |
|---|---|
| `system-properties` | Bloque en `standalone.xml` para definir propiedades del servidor |
| `System.getProperty()` | Cómo el código Java lee las propiedades del servidor |
| Cambio en caliente | `write-attribute` via CLI aplica el cambio sin reiniciar WildFly |
| `app.env` | Patrón habitual para identificar el entorno LAB / PRE / PRO |
| `db.jndi` | El datasource cambia por entorno sin tocar el WAR |
| CLI desde cualquier ruta | Usar ruta completa `& "C:\wildfly-39.0.1.Final\bin\jboss-cli.bat"` |

---

## ✅ Fase 11 — Backup y rollback de despliegues

### Concepto

Ante un fallo en PRO tras un despliegue, el objetivo es volver a la versión anterior en el menor
tiempo posible. El sistema de versionado guarda automáticamente una copia del WAR antes de cada
despliegue, permitiendo rollback inmediato.

### Estructura de versiones

```
C:\wildfly-39.0.1.Final\
  ├── versions\
  │   ├── mi-mini-app_v1.0_20260415-1000.war   ← versión anterior
  │   └── mi-mini-app_v2.0_20260415-1005.war   ← versión actual
  ├── bin\
  │   ├── deploy-versioned.bat                  ← despliega y guarda versión
  │   └── rollback.bat                          ← restaura versión anterior
  └── deploy-history.log                        ← historial completo de operaciones
```

### Crear la carpeta de versiones

```powershell
New-Item -ItemType Directory -Path "C:\wildfly-39.0.1.Final\versions" -Force
```

### Uso de deploy-versioned.bat

```bat
REM Sintaxis: deploy-versioned.bat <WAR> <VERSION> <INSTANCIA>
cd C:\wildfly-39.0.1.Final\standalone\deployments
C:\wildfly-39.0.1.Final\bin\deploy-versioned.bat mi-mini-app.war v2.0 1
```

El script: guarda copia versionada → despliega → registra en `deploy-history.log`.

### Uso de rollback.bat

```bat
REM Sintaxis: rollback.bat <WAR> <INSTANCIA>
C:\wildfly-39.0.1.Final\bin\rollback.bat mi-mini-app.war 1
```

El script: lista versiones disponibles → pide elección → restaura el WAR → redesplega.

### Consultar historial de despliegues

```powershell
type "C:\wildfly-39.0.1.Final\deploy-history.log"
```

Cada línea registra: fecha · hora · resultado (OK/FAILED/ROLLBACK) · WAR · versión · instancia.

### 🧠 Lecciones aprendidas — Fase 11

| Qué aprendiste | Detalle |
|---|---|
| Carpeta `versions\` | Repositorio local de WARs versionados |
| `deploy-versioned.bat` | Guarda copia antes de desplegar — nunca pierdes la versión anterior |
| `rollback.bat` | Lista versiones disponibles y restaura en segundos |
| `deploy-history.log` | Registro de todas las operaciones — quién desplegó qué y cuándo |
| `call set "%%VAR%%"` | Truco CMD para expandir variables dentro de bloques `for` |
| Nunca desplegar sin backup | El script fuerza guardar la versión antes de cada despliegue |

---

## 🛠️ Entorno

- **WildFly**: 39.0.1.Final
- **Java**: 17+
- **OS**: Windows 10/11
- **Instancia 1**: servicio `wildfly` — puerto 8080 — admin 9990
- **Instancia 2**: servicio `WildFly2` — puerto 8180 — admin 10090

---

## 📝 Licencia

Uso personal / educativo.
