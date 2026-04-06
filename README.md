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
│   ├── 03-multi-instance.md           ← dos instancias en puertos distintos ✅
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

| Fase | Descripción | Doc | Estado |
|------|-------------|-----|--------|
| 1 | Instalar WildFly como servicio Windows | [01-setup-service.md](docs/01-setup-service.md) | ✅ |
| 2 | Despliegue básico de un WAR | [02-single-deployment.md](docs/02-single-deployment.md) | ✅ |
| 3 | Dos instancias en puertos distintos | [03-multi-instance.md](docs/03-multi-instance.md) | ✅ |
| 4 | Script .bat automatizado | [04-deploy-script.md](docs/04-deploy-script.md) | 🔄 |
| 5 | Consola de administración 9990 | [05-admin-console.md](docs/05-admin-console.md) | 🔄 |

---

## ⚡ Quickstart — despliegue rápido

```bat
REM 1. Copiar el WAR
copy mi-app.war C:\wildfly-39.0.1.Final\standalone\deployments\

REM 2. Forzar despliegue
echo. > C:\wildfly-39.0.1.Final\standalone\deployments\mi-app.war.dodeploy

REM 3. Verificar log
powershell "Get-Content C:\wildfly-39.0.1.Final\standalone\log\server.log -Wait -Tail 50"
```

---

## 🔖 Marcadores de despliegue

| Archivo | Significado |
|---------|-------------|
| `.war.dodeploy` | Solicita despliegue — WildFly lo procesa y lo borra |
| `.war.deployed` | ✅ Desplegado correctamente |
| `.war.failed` | ❌ Error — abrirlo para leer la causa |
| `.war.undeployed` | Detenido explícitamente |
| `.war.skipdeploy` | Ignorado aunque esté en la carpeta |

---

## 🔀 Fase 3 — Dos instancias en puertos distintos

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
| HTTP   | 8080        | 8180        |
| HTTPS  | 8443        | 8543        |
| Admin  | 9990        | 10090       |
| Management | 9999   | 10099       |

El offset se configura con un único cambio en `standalone.xml`.

---

### Paso 1 — Copiar la carpeta standalone

```bat
xcopy /E /I /H C:\wildfly-39.0.1.Final\standalone C:\wildfly-39.0.1.Final\standalone2
```

---

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

---

### Paso 3 — Crear el script de arranque

Crear el fichero `C:\wildfly-39.0.1.Final\bin\start-instance2-service.bat` desde CMD línea a línea:

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

Debe mostrar:

```bat
@echo off
setlocal
set WILDFLY_HOME=C:\wildfly-39.0.1.Final
call "%WILDFLY_HOME%\bin\standalone.bat" -Djboss.server.base.dir="%WILDFLY_HOME%\standalone2" -Djboss.node.name=wildfly-node2
endlocal
```

---

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

---

### Paso 5 — Registrar el servicio Windows

En **PowerShell como Administrador** usar `sc.exe` (no `sc`, que en PowerShell es alias de `Set-Content`):

```powershell
sc.exe create "WildFly2" binPath= "C:\wildfly-39.0.1.Final\bin\WildFly2Service.exe" DisplayName= "WildFly Application Server_ 2" start= auto

sc.exe description "WildFly2" "Servidor WildFly instancia 2 - puerto 8180"
```

---

### Paso 6 — Reiniciar y verificar

> ⚠️ Si aparece el error "marcado para eliminación" al intentar crear el servicio,
> es necesario reiniciar la máquina para que Windows limpie los servicios pendientes.

Tras el reinicio, verificar que los dos servicios arrancan automáticamente:

```bat
sc query wildfly
sc query WildFly2
```

Ambos deben mostrar `STATE: 4 RUNNING`.

---

### Verificación final en el navegador

```
http://localhost:8080/mi-mini-app/    ← instancia 1 ✅
http://localhost:8180/mi-mini-app/    ← instancia 2 ✅

http://localhost:9990                 ← admin consola instancia 1
http://localhost:10090                ← admin consola instancia 2
```

---

### Desplegar un WAR en la instancia 2

```bat
copy mi-app.war C:\wildfly-39.0.1.Final\standalone2\deployments\
echo. > C:\wildfly-39.0.1.Final\standalone2\deployments\mi-app.war.dodeploy
```

Log de la instancia 2:

```bat
powershell "Get-Content C:\wildfly-39.0.1.Final\standalone2\log\server.log -Wait -Tail 50"
```

---

### 🧠 Lecciones aprendidas — Fase 3

| Problema encontrado | Causa | Solución |
|--------------------|-------|----------|
| `sc` falla en PowerShell | Es alias de `Set-Content` | Usar `sc.exe` explícitamente |
| Servicio no responde con `.bat` | `cmd.exe` no mantiene el proceso vivo | Compilar wrapper `.exe` con `Add-Type` |
| Error "marcado para eliminación" | Servicio pendiente de borrado en memoria | Reiniciar la máquina |
| Conflicto de puertos al arrancar inst. 2 | Dos procesos en el mismo puerto | Configurar `port-offset=100` en `standalone.xml` |
| `.bat` creado en una sola línea | Uso incorrecto de bloque `()` en CMD | Crear fichero con `echo` línea a línea y `>>` |

---

## 🛠️ Entorno

- **WildFly**: 39.0.1.Final
- **Java**: 17+
- **OS**: Windows 10/11
- **Instancia 1**: servicio `wildfly` — puerto 8080 — admin 9990
- **Instancia 2**: servicio `WildFly2` — puerto 8180 — admin 10090

---

## 📚 Documentación detallada

Cada fase tiene su propio documento en [`docs/`](docs/) con pasos, comandos y troubleshooting.

---

## 📝 Licencia

Uso personal / educativo.
