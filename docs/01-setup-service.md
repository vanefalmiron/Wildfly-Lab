cd C:\wildfly-39.0.1.Final\bin

REM Instalar el servicio
wildfly-service.exe install

REM Configurar inicio automático
sc config wildfly start= auto

REM Arrancar
net start wildfly

REM Verificar estado
sc query wildfly
