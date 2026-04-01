# 🚀 wildfly-lab

Laboratorio personal para practicar despliegues con **WildFly 39** en Windows.  
Cubre instalación como servicio, despliegue de WARs, automatización y consola de administración.

> **Objetivo**: replicar en local el flujo de trabajo de un entorno PRO antes de tocarlo.

---

## 📁 Estructura del repositorio

```
wildfly-lab/
│
├── README.md                          ← este archivo
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
│   ├── start-instance1.bat            ← arrancar instancia 1 (puerto 8080)
│   └── start-instance2.bat            ← arrancar instancia 2 (puerto 8180)
│
├── config/
│   ├── instance1/
│   │   └── standalone.xml             ← configuración WildFly instancia 1
│   └── instance2/
│       └── standalone.xml             ← configuración WildFly instancia 2 (puertos offset +100)
│
├── sample-app/
│   ├── mi-mini-app.war                ← WAR de ejemplo
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
  ┌─────────────────────────────────────────────┐
  │              Windows Host                   │
  │                                             │
  │  ┌──────────────────┐  ┌─────────────────┐  │
  │  │  WildFly inst. 1 │  │ WildFly inst. 2 │  │
  │  │   puerto 8080    │  │  puerto 8180    │  │
  │  │   admin: 9990    │  │  admin: 10090   │  │
  │  └──────────────────┘  └─────────────────┘  │
  │           │                    │             │
  │  ┌────────┴────────────────────┴──────────┐  │
  │  │         deployments/                   │  │
  │  │   mi-mini-app.war  + .deployed         │  │
  │  └────────────────────────────────────────┘  │
  │                                             │
  │  ┌──────────────────────────────────────┐    │
  │  │         deploy.bat                   │    │
  │  │  copia WAR → crea .dodeploy → watch  │    │
  │  └──────────────────────────────────────┘    │
  └─────────────────────────────────────────────┘
```

---

## 🧭 Fases del laboratorio

| Fase | Descripción | Doc | Estado |
|------|-------------|-----|--------|
| 1 | Instalar WildFly como servicio Windows | [01-setup-service.md](docs/01-setup-service.md) | ✅ |
| 2 | Despliegue básico de un WAR | [02-single-deployment.md](docs/02-single-deployment.md) | ✅ |
| 3 | Dos instancias en puertos distintos | [03-multi-instance.md](docs/03-multi-instance.md) | 🔄 |
| 4 | Script .bat automatizado | [04-deploy-script.md](docs/04-deploy-script.md) | 🔄 |
| 5 | Consola de administración 9990 | [05-admin-console.md](docs/05-admin-console.md) | 🔄 |

---

## ⚡ Quickstart — despliegue rápido

```bat
REM 1. Copiar el WAR
copy mi-app.war C:\wildfly-39.0.1.Final\standalone\deployments\

REM 2. Forzar despliegue
echo. > C:\wildfly-39.0.1.Final\standalone\deployments\mi-app.war.dodeploy

REM 3. Verificar
powershell "Get-Content C:\wildfly-39.0.1.Final\standalone\log\server.log -Wait -Tail 50"
```

O usar el script automatizado:
```bat
scripts\deploy.bat mi-app.war
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

## 🛠️ Entorno

- **WildFly**: 39.0.1.Final
- **Java**: 17+
- **OS**: Windows 10/11
- **Servicio**: wildfly-service.exe (inicio automático)

---

## 📚 Documentación

Cada fase tiene su propio documento en [`docs/`](docs/) con pasos detallados, comandos y notas de troubleshooting.

Para el flujo completo de buenas prácticas PRO, ver [`docs/architecture.md`](docs/architecture.md).

---

## 📝 Licencia

Uso personal / educativo.
