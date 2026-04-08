# 🚀 wildfly-lab

Laboratorio personal para practicar despliegues con **WildFly 39** en Windows.  
Cubre instalación como servicio, despliegue de WARs, múltiples instancias y automatización.

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
│   └── architecture.md               ← diagrama y descripción de arquitectura
│
├── scripts/
│   ├── deploy.bat                     ← despliegue automatizado con verificación
│   ├── start-instance2-service.bat    ← arrancar instancia 2 manualmente
│   └── WildFly2Service.exe            ← wrapper .exe para servicio Windows
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
  │           │                          │                │
  │  standalone\deployments\    standalone2\deployments\  │
  │   mi-mini-app.war            mi-mini-app.war          │
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
| 4 | Script .bat automatizado | 🔄 |
| 5 | Consola de administración 9990 | 🔄 |

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
- Ejecutar todos los comandos como **Administrador**

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
  └── modules\                   ← drivers y módulos externos
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

Para cambiarlo, añadir `WEB-INF/jboss-web.xml`:

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

> ⚠️ Este es el **único cambio** necesario en el XML. Un solo número desplaza todos los puertos +100.

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

Abrir **PowerShell como Administrador** y pegar este bloque completo:

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

En **PowerShell como Administrador** usar `sc.exe` (no `sc`, que en PowerShell es alias de `Set-Content`):

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
http://localhost:8080/mi-mini-app/    ← instancia 1 ✅
http://localhost:8180/mi-mini-app/    ← instancia 2 ✅

http://localhost:9990                 ← admin consola instancia 1
http://localhost:10090                ← admin consola instancia 2
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

## 🛠️ Entorno

- **WildFly**: 39.0.1.Final
- **Java**: 17+
- **OS**: Windows 10/11
- **Instancia 1**: servicio `wildfly` — puerto 8080 — admin 9990
- **Instancia 2**: servicio `WildFly2` — puerto 8180 — admin 10090

---

## 📝 Licencia

Uso personal / educativo.
