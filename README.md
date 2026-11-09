# Proyecto-de-Ciberseguridad-Laboratorio-de-Active-Directory-SIEM-y-Hardening
Proyecto de Ciberseguridad — Laboratorio de Active Directory, SIEM y Hardening

# De cero a una red segura — Historia del laboratorio de ciberseguridad
Virgilio Antonio Frías | Junio – Julio 2026

## 

!image.png

## Imagina que te piden construir la infraestructura de tecnología de una empresa desde cero, en 7 días, y al final tienes que demostrar en vivo que todo funciona y que es seguro. Eso fue este proyecto.
Todo corrió sobre Proxmox, una plataforma que permite crear computadoras virtuales dentro de una sola máquina física. Construí tres: un servidor Windows que controla la red, una computadora de empleado, y un servidor Linux que vigila todo lo que pasa en tiempo real.

## Parte 1 — Levantando el servidor que manda a todos
[Foto: pantalla de instalación de Windows Server 2022 en Proxmox]

!image.png

Lo primero fue instalar el servidor principal, conocido en el mundo empresarial como Domain Controller. Es exactamente lo que suena: el servidor que controla el dominio, es decir, toda la red de la empresa.
Lo instalé con Windows Server 2022, le asigné una dirección fija en la red (`172.16.124.23`) y lo convertí en la autoridad del dominio Vfrias.local.
Actualizaciones antes de tocar nada
Antes de instalar cualquier rol o configurar nada, actualicé el sistema operativo completamente. Esto es lo primero que siempre debe hacerse — las actualizaciones cierran puertas de seguridad que los atacantes podrían usar. Una vez todo al día, procedí con la instalación del rol de Active Directory.
Instalación del rol de Active Directory
📸 [Foto: Server Manager con el rol AD DS instalado y la opción "Promote this server to a domain controller"]
Desde el Server Manager instalé el rol de Active Directory Domain Services (AD DS), que es el sistema de Microsoft para gestionar usuarios, grupos y políticas en toda la red. Al terminar la instalación, el sistema me ofreció la opción de promover el servidor a Domain Controller — ese fue el momento clave, cuando el servidor pasó de ser "una computadora con Windows" a ser la autoridad de la red.
Durante la promoción configuré:
Nombre del dominio: Vfrias.local
El servidor también se encargó del DNS (el "directorio telefónico" de la red)
Se definió una contraseña de recuperación de emergencia (DSRM) y se guardó en lugar seguro
Después del reinicio automático, el servidor arrancó como Domain Controller del dominio Vfrias.local.

Parte 2 — Organizando la empresa: usuarios, grupos y jerarquías

Con el servidor listo, llegó el momento de organizar la "empresa virtual". Creé lo que se conoce como Unidades Organizativas (OUs) — carpetas dentro del directorio que agrupan a los usuarios según su rol, permitiendo aplicar reglas de seguridad distintas a cada grupo.
La estructura quedó así:

!image.png

```
Vfrias.local
├── OU_Admin          → cuentas administrativas
├── OU_Servers        → servidores de la red
├── OU_Workstations   → computadoras de empleados
├── OU_Usuarios
│   ├── OU_IT         → equipo de tecnología
│   └── OU_Operaciones → empleados comunes
└── OU_Grupos         → grupos de seguridad
```

Grupos de seguridad
Grupo	Descripción	Nivel de acceso
GG_IT_Admins	Soporte y administración	Control total
GG_Usuarios_IT	Personal de IT	Acceso parcial
GG_Usuarios_Operaciones	Empleados comunes	Solo lo que necesitan
GG_Acceso_Wazuh	Acceso a monitoreo	Uso futuro
Usuarios del dominio
📸 [Foto: PowerShell mostrando la creación exitosa de usuarios]
Usuario	Nombre	Contraseña	Rol
`cadmin`	Carlos Admin	AdminLab2025!Seguro	Administrador de IT
`aiti`	Ana IT	ITLab2025!Seguro	Soporte técnico
`loperaciones`	Luis Operaciones	UserLab2025!Seguro	Empleado común
`mgarcia`	Maria Garcia	UserLab2025!Seguro	Empleado común
Para iniciar sesión en cualquier máquina del dominio, el formato es:

```
Vfrias\cadmin    o bien    cadmin@Vfrias.local
```

## Políticas de contraseña diferenciadas
Aquí viene algo que marca la diferencia entre una red segura y una insegura: no todos los usuarios tienen las mismas reglas de contraseña. Los administradores tienen más poder, por lo que sus cuentas necesitan más protección.
Política	Administradores	Usuarios comunes
Longitud mínima	14 caracteres	10 caracteres
Bloqueo tras intentos fallidos	3 intentos	5 intentos
Duración del bloqueo	30 minutos	15 minutos
Vigencia máxima	30 días	60 días
Historial	10 contraseñas	5 contraseñas

Parte 3 — La computadora del empleado entra a la red
📸 [Foto: ventana "Computer Name/Domain Changes" con el campo de dominio Vfrias.local]
Creé una segunda máquina virtual con Windows 10 — la computadora del empleado típico. El objetivo era conectarla al dominio para que las reglas de la empresa aplicaran automáticamente en ella.
⚠️ Problema: el primer intento falló con el error:

> *"An Active Directory Domain Controller for the domain Vfrias.local could not be contacted"*
Causa: la computadora no sabía dónde encontrar al servidor porque le faltaba la dirección DNS correcta. Estaba buscando en la dirección equivocada.
Solución: configuré el DNS del equipo cliente para que apuntara directamente al servidor (`172.16.124.23`). Después de ese ajuste, la conexión funcionó de inmediato.
Para unirse al dominio se usaron las credenciales del administrador (`cadmin / AdminLab2025!Seguro`), y el equipo se reinició automáticamente.
📸 [Foto: pantalla de login de Windows mostrando "VFRIAS\mgarcia" como usuario de dominio disponible]
> 

!image.png

El resultado fue el más satisfactorio hasta ese momento: en la pantalla de login ya no solo aparecía la cuenta local — ahora había usuarios de dominio reales disponibles, incluyendo `VFRIAS\mgarcia`. La empresa virtual tenía su primera computadora conectada al sistema.

---

Parte 4 — Instalando las cámaras de seguridad de la red
Una red sin monitoreo es como una tienda sin cámaras — si algo pasa, te enteras demasiado tarde. Para resolver eso instalé Wazuh, un sistema de monitoreo de seguridad que observa todo lo que ocurre en las computadoras conectadas y lanza alertas cuando detecta algo sospechoso.
Instalación del servidor Ubuntu
📸 [Foto: pantalla de "Profile configuration" de Ubuntu con usuario vfrias]
Instalé Ubuntu Server 22.04 (Linux) como sistema base para Wazuh. Elegí la versión sin interfaz gráfica porque Wazuh se administra desde el navegador, no desde el escritorio — lo que consume menos recursos y tiene menos puntos vulnerables.
Configuración del servidor:
Hostname: `vfrias`
Usuario: `vfrias`
IP asignada: `172.16.124.22`
OpenSSH instalado para administración remota
Instalación de Wazuh

!image.png

La instalación se hizo con el script oficial en modo all-in-one, que instala en una sola máquina los tres componentes:
Manager: el cerebro que procesa todo
Indexer: la base de datos donde se almacenan los eventos
Dashboard: la pantalla donde se ve todo en tiempo real

```bash
curl -O <https://packages.wazuh.com/4.9/wazuh-install.sh>
sudo bash wazuh-install.sh -a
```

## Al terminar, el sistema mostró las credenciales de acceso al panel — hay que guardarlas en ese momento porque no se vuelven a mostrar.
⚠️ Errores encontrados durante la instalación:
Primer intento fallido: se usó `0` (cero) en lugar de `O` (letra O mayúscula) en el comando curl. El archivo nunca se descargó.
Segundo intento: se intentó poner `vfrias.local` como nombre del servidor durante la configuración — el instalador no acepta puntos en el hostname. Se usó solo `vfrias`.

!Screenshot 2026-06-29 194750.png

Una vez resueltos esos detalles, el dashboard cargó correctamente y el sistema de monitoreo quedó operativo.

Parte 5 — Conectando las cámaras a las computadoras
Para que Wazuh vigilara las máquinas de la red, había que instalar un pequeño programa en cada una — el agente de Wazuh. Es como colocar una cámara pequeña en cada habitación: el agente es la cámara, y el servidor de Wazuh es la sala de monitoreo.
Instalación del agente en DC01 (Windows Server)

!Screenshot 2026-06-29 114413.png

Desde el panel de Wazuh, generé el comando automático de instalación para Windows, lo copié en PowerShell de DC01 y lo ejecuté:

```powershell
Invoke-WebRequest -Uri <https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi> -OutFile $env:tmp\wazuh-agent
msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='172.16.124.22' WAZUH_AGENT_NAME='DC01'
NET START WazuhSvc
```

⚠️ Problema: el agente instaló pero nunca apareció como activo en el dashboard. Al revisar el log del agente se encontró el error:

```
ERROR: Unable to connect to enrollment service at 'Vfrias.local':1515
```

Causa: por error se usó el nombre del dominio (`Vfrias.local`) como dirección del servidor de Wazuh. Ese nombre no resuelve hacia el servidor Ubuntu — resuelve dentro del contexto del dominio de Windows, que es otro sistema completamente diferente.
Solución: corregí la configuración del agente para que usara la IP directa del servidor Wazuh (`172.16.124.22`) en lugar del nombre de dominio. Después de reiniciar el agente, la conexión se estableció en segundos.
⚠️ Problema adicional — agente duplicado:
En el dashboard aparecieron dos registros para el mismo equipo: uno llamado `DC01` (ID 001, estado "Never connected") y otro llamado `WIN-LM3LJQOKKTL` (ID 002, estado "Active"). El primero era un registro huérfano de un intento anterior que nunca conectó. Se eliminó por línea de comandos:

```bash
sudo /var/ossec/bin/manage_agents
# Opción R → ID 001 → confirmar → Q para salir
```

## Tras la limpieza, el dashboard mostró un único agente activo para el servidor.
Instalación del agente en CLIENT01 (Windows 10)
Mismo proceso que con DC01, pero esta vez con el nombre correcto desde el inicio (`WAZUH_AGENT_NAME='CLIENT01'` y `WAZUH_MANAGER='172.16.124.22'`).
📸 [Foto: dashboard de Wazuh mostrando DC01 y CLIENT01 ambos con estado "Active" y punto verde]
Con ambos agentes activos, el sistema tenía ojos en las dos máquinas de la red.

Parte 6 — Crisis a las 5 de la madrugada: el disco lleno
Esta fue la situación más intensa del proyecto. En un momento dado, el dashboard de Wazuh dejó de cargar y mostraba solo: "Something went wrong", con un mensaje adicional de "Please set up API credentials".

!Screenshot 2026-06-28 233202.png

Me conecté al servidor por SSH y revisé el estado de los servicios. El diagnóstico llegó rápido con un solo comando:

```bash
df -h
```

📸 [Foto: resultado de df -h mostrando /dev/sda2 32G 32G 0 100%]

```
/dev/sda2    32G    32G    0    100%
```

El disco estaba completamente lleno. Sin espacio libre, el sistema no podía escribir nada — ni logs, ni datos nuevos, ni archivos temporales. Todos los servicios fallaban en cascada por eso.
⚠️ Primer intento de solución (fallido): fui a Proxmox y por error usé la opción de añadir un disco nuevo en lugar de ampliar el disco existente. Resultado: apareció un segundo disco vacío de 54 GB que no ayudaba en nada a la partición original que estaba llena.
Intenté mover los datos de Wazuh al disco nuevo, pero tampoco funcionó — con el disco a 0 bytes, ni siquiera había espacio para que los comandos de copia crearan sus archivos temporales.
✅ Solución correcta: eliminé el disco añadido por error y esta vez usé la opción correcta en Proxmox: "Redimensionar disco" sobre el disco original, ampliándolo de 32 GB a 59 GB.
⚠️ Segundo obstáculo: incluso con el disco ampliado, el comando para extender la partición (`growpart`) fallaba porque necesitaba crear un archivo temporal en el mismo disco que seguía lleno a nivel de sistema de archivos. La solución fue redirigir ese temporal a la memoria RAM del sistema:

```bash
sudo TMPDIR=/run growpart /dev/sda 2
sudo resize2fs /dev/sda2
```

## 📸 [Foto: resultado de df -h mostrando /dev/sda2 58G 32G 24G 57% — después de la solución]
El disco pasó de 0 bytes libres a 24 GB disponibles. Los servicios de Wazuh volvieron a funcionar y el dashboard recuperó el acceso normal.
Lo que aprendí: un error en la aplicación ("Something went wrong") no siempre tiene su causa en la aplicación. Hay que diagnosticar por capas — desde lo más básico (¿hay espacio en disco? ¿están los servicios corriendo?) hacia lo más complejo. En este caso, la causa estaba en la infraestructura, no en el software.

Parte 7 — Habilitando el registro de eventos en Windows
Para que Wazuh pudiera detectar intentos de acceso no autorizado, primero había que decirle a Windows que registrara esos eventos. Por defecto, Windows no anota todos los intentos de login — hay que activarlo explícitamente.
Se habilitó la auditoría en DC01:

```powershell
auditpol /set /category:"Logon/Logoff" /success:enable /failure:enable
auditpol /set /category:"Account Management" /success:enable /failure:enable
auditpol /set /category:"Account Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
```

## Con esto, cada intento de login (exitoso o fallido) quedó registrado en el Event Log de Windows, visible para el agente de Wazuh.

Parte 8 — Reglas de detección personalizadas para el laboratorio
Wazuh ya trae reglas predefinidas para detectar muchos tipos de amenazas. Sin embargo, también se crearon reglas propias adaptadas al entorno del laboratorio, guardadas en `/var/ossec/etc/rules/local_rules.xml`.
Las reglas cubren los eventos más relevantes:
Autenticación y acceso:
Login fallido (Event ID 4625) — nivel de alerta: 8
Fuerza bruta: 5 fallos en 2 minutos para el mismo usuario — nivel: 12
Login exitoso (4624) — nivel: 5
Cuenta bloqueada (4740) — nivel: 10
Active Directory:
Usuario creado en AD (4720) — nivel: 8
Usuario eliminado en AD (4726) — nivel: 10
Usuario agregado a grupo privilegiado (4728/4732/4756) — nivel: 10
Privilegios especiales asignados (4672) — nivel: 12
Seguridad del sistema:
Política de auditoría modificada (4719) — nivel: 10
Tarea programada creada o modificada (4698/4702) — nivel: 12
Nuevo servicio instalado (7045) — nivel: 10
Kerberos:
Cuenta deshabilitada intentó autenticación Kerberos (4768) — nivel: 10
Posible Kerberoasting — cifrado débil RC4 detectado (4769) — nivel: 12
Detección avanzada con Sysmon:
Ejecución de procesos nuevos — nivel: 8
Conexiones de red iniciadas por procesos — nivel: 8
Modificaciones en el Registro de Windows — nivel: 7
Posible uso de Mimikatz (herramienta de robo de credenciales) — nivel: 12
PowerShell con comandos codificados en Base64 — nivel: 10
Después de guardar el archivo de reglas, se reinició el manager para que las cargara:

```bash
sudo systemctl restart wazuh-manager
```

---

Parte 9 — Las reglas que protegen toda la red (GPOs)
Con el monitoreo funcionando, llegó el turno de configurar las reglas de seguridad que aplican a toda la red automáticamente. Estas se llaman GPOs (Group Policy Objects) y son una de las herramientas más poderosas de Active Directory: se definen una vez en el servidor y se aplican automáticamente en todas las computadoras del dominio.
SMB Signing — firmando las conversaciones de red
📸 [Foto: Group Policy Management Editor mostrando las políticas de SMB Signing como "Enabled"]
Sin esta protección, alguien dentro de la red podría interceptar el tráfico de archivos compartidos entre computadoras y manipularlo sin que nadie se diera cuenta — un ataque conocido como NTLM Relay. Con SMB Signing activado, cada paquete de datos va firmado digitalmente. Si alguien lo modifica en el camino, la firma se rompe y el sistema lo detecta.
Se creó la GPO "SMB Signing Policy" con dos políticas:
Política	Valor
Microsoft network client: Digitally sign communications (always)	Enabled
Microsoft network server: Digitally sign communications (always)	Enabled
La GPO se vinculó al dominio completo y se aplicó con `gpupdate /force`. Verificación:

```powershell
Get-SmbServerConfiguration | Select RequireSecuritySignature, EnableSecuritySignature
```

```
RequireSecuritySignature: True
EnableSecuritySignature:  True
```

LDAP Signing — protegiendo las consultas al directorio
📸 [Foto: resultado del registro de Windows mostrando LDAPServerIntegrity : 2]
LDAP es el protocolo que usan las aplicaciones para consultar el Active Directory. Sin firma en esas consultas, un atacante podría falsificarlas y extraer información del directorio.
Se creó la GPO "LDAP signing policy" y se configuró:
Política	Valor
Domain controller: LDAP server signing requirements	Require signing
⚠️ Problema: después de aplicar la GPO, el valor del registro no se actualizó de forma inmediata. Se forzó directamente:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "LDAPServerIntegrity" -Value 2
```

## Verificación final: `ldapserverintegrity : 2` — el servidor ahora rechaza cualquier conexión LDAP que no llegue firmada.
Valor	Comportamiento
0	Sin requerimiento — acepta todo
1	Negocia — prefiere firma pero no la exige
2	Obligatorio — rechaza conexiones sin firma ✅

Parte 10 — Endureciendo las defensas (Hardening)
Instalar y configurar es solo la mitad del trabajo. La otra mitad es endurecer los sistemas: revisar todas las configuraciones predeterminadas de Windows que son inseguras por defecto y corregirlas. Seguí las guías del CIS Benchmark, una lista de controles recomendados por expertos internacionales.
Hardening del servidor (DC01)
📸 [Foto: PowerShell mostrando LmCompatibilityLevel : 5 después de la configuración]
Política de contraseñas del dominio mejorada:
Parámetro	Valor anterior	Valor nuevo
Longitud mínima	7 caracteres	14 caracteres
Bloqueo tras intentos	Sin bloqueo	5 intentos
Duración del bloqueo	Sin límite	15 minutos
Vigencia máxima	42 días	60 días
Deshabilitar protocolos de autenticación obsoletos (NTLMv1 y LM):
NTLMv1 y LM son protocolos de los años 90 que siguen activos por compatibilidad pero son vulnerables con hardware moderno. Sus contraseñas se pueden descifrar en minutos con las herramientas correctas. Se configuró el servidor para rechazar cualquier autenticación que no sea NTLMv2:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "LmCompatibilityLevel" -Value 5
```

Verificado: `LmCompatibilityLevel : 5`
Renombrar la cuenta Administrator:
La cuenta `Administrator` es el objetivo número uno de cualquier ataque automatizado. Se renombró a `AdminVfrias` para dificultar los ataques dirigidos a esa cuenta específica.

```powershell
Rename-LocalUser -Name "Administrator" -NewName "AdminVfrias"
```

## ⚠️ Momento que confirmó que todo funciona: durante las pruebas de generación de eventos para el SIEM, la cuenta `AdminVfrias` se bloqueó después de 5 intentos fallidos — exactamente como estaba configurado. Lejos de ser un problema, fue la mejor demostración posible de que las políticas funcionan en la práctica. Se restableció la contraseña a `AdminFuerte2026!!` y el sistema siguió operando normalmente.
Resumen hardening DC01:
Medida	Control CIS	Estado
Contraseña mínima 14 caracteres	CIS 5.2	✅
Bloqueo tras 5 intentos fallidos	CIS 5.2	✅
Duración de bloqueo 15 minutos	CIS 5.2	✅
NTLMv1 y LM deshabilitados	CIS 9.2	✅
Cuenta Administrator renombrada a AdminVfrias	CIS 5.1	✅
Hardening del equipo cliente (CLIENT01)
Se aplicaron los mismos controles en la computadora del empleado. Un solo equipo sin hardening puede ser el punto de entrada que invalide todos los controles del servidor.

!image.png

en el cliente mostrando los valores de política de contraseñas]
Medida	Control CIS	Estado
NTLMv1 y LM deshabilitados	CIS 9.2	✅
Contraseña mínima 14 caracteres	CIS 5.2	✅
Bloqueo tras 5 intentos fallidos	CIS 5.2	✅
Duración de bloqueo 15 minutos	CIS 5.2	✅
Cuenta Administrator renombrada a AdminCliente	CIS 5.1	✅
Cuentas Guest/DefaultAccount/WDAGUtilityAccount deshabilitadas	CIS 5.1	✅

Parte 11 — La alerta que cierra el círculo
📸 [Foto: dashboard de Wazuh — sección Threat Hunting mostrando los eventos de authentication failure]
Con todo configurado, llegó el momento de demostrar que el sistema detecta amenazas de extremo a extremo. El escenario fue simple: desde la computadora del empleado, se bloqueó la sesión y se intentó iniciar sesión con contraseña incorrecta varias veces seguidas — lo mismo que haría un atacante intentando adivinar contraseñas.
Resultado en el dashboard de Wazuh:
Métrica	Valor
Total de eventos (último día)	3,260
Intentos de autenticación fallidos	4
Autenticaciones exitosas	1,242
Filtrando por el Event ID 4625 (login fallido), se obtuvieron los detalles completos de cada intento:
Campo	Valor
Agente	WIN-LM3LJQOKKTL (DC01)
IP del agente	172.16.124.23
Motivo del fallo	Contraseña incorrecta
Tipo de login	7 (desbloqueo de pantalla)
Proceso	svchost.exe
Dominio	VFRIAS
El módulo MITRE ATT&CK del dashboard clasificó automáticamente las tácticas detectadas, incluyendo: Valid Accounts, Domain Policy Modification, Account Access Restriction.
📸 [Foto: dashboard de Wazuh mostrando la alerta de fuerza bruta activa con clasificación MITRE ATT&CK]
El flujo completo quedó validado de extremo a extremo:

```
DC01 (Windows Event Log)
    → Agente Wazuh
        → Manager
            → Indexer
                → Dashboard
                    → Alerta visible en menos de 1 minuto
```

## Ese es el resultado final: desde el intento de acceso no autorizado hasta la alerta en el sistema de monitoreo, todo conectado y funcionando en tiempo real.

## Resumen final del entorno
Componente	Tecnología	Estado
Servidor principal (Domain Controller)	Windows Server 2022	✅ Operativo
Dominio Vfrias.local	Active Directory con OUs, grupos y usuarios	✅ Configurado
Computadora del empleado	Windows 10 unido al dominio	✅ Operativo
Sistema de monitoreo (SIEM)	Wazuh all-in-one sobre Ubuntu 22.04	✅ Operativo
Agentes de monitoreo	DC01 + CLIENT01	✅ Activos
Protección del tráfico de red	SMB Signing (GPO)	✅ Activo
Protección del directorio	LDAP Signing (GPO + registro)	✅ Activo
Endurecimiento de sistemas	CIS Benchmark (controles priorizados)	✅ Aplicado
Reglas de detección personalizadas	20+ reglas en local_rules.xml	✅ Activas
Alertas en tiempo real	Fuerza bruta, cuentas bloqueadas, cambios en AD	✅ Funcionando

Credenciales del laboratorio

> ⚠️ Estas credenciales son exclusivas del entorno de laboratorio. No se deben reutilizar en producción.
Sistema	Usuario	Contraseña	Notas
Ubuntu (Wazuh)	vfrias	(definida en instalación)	Acceso SSH
Wazuh Dashboard	admin	(generada por instalador)	https://172.16.124.22
Dominio — Administrador	AdminVfrias	AdminFuerte2026!!	Restablecida tras bloqueo durante pruebas
Dominio — IT Admin	cadmin	AdminLab2025!Seguro	Miembro de GG_IT_Admins
Dominio — IT	aiti	ITLab2025!Seguro	Usuario estándar
Dominio — Operaciones	loperaciones	UserLab2025!Seguro	Usuario estándar
Dominio — Operaciones	mgarcia	UserLab2025!Seguro	Usuario estándar
> 

---

Los problemas que surgieron durante el proceso — el disco lleno, el agente que no conectaba, el error de DNS, el agente duplicado, el timeout del indexer — no se ocultaron. Se diagnosticaron, se resolvieron y se documentaron. Eso también es parte del trabajo real: los sistemas no funcionan perfectos desde el primer intento, y saber leer un error y encontrar su causa es tan importante como saber ejecutar el comando correcto.
