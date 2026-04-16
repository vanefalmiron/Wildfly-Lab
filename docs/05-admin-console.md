##WildFly bloquea la consola por defecto. Desde CMD como Administrador:

cd C:\wildfly-39.0.1.Final\bin
add-user.bat

##Seguir el asistente:

What type of user do you wish to add?
 a) Management User  ← elegir esta
Username: admin
Password: admin123*
Groups: (dejar vacío)
Is this correct? yes
Used for AS process interconnection? no

##Instancia2

add-user.bat --user-properties C:\wildfly-39.0.1.Final\standalone2\configuration\mgmt-users.properties --group-properties C:\wildfly-39.0.1.Final\standalone2\configuration\mgmt-groups.properties
