[proyecto-laboratorio.zip](https://github.com/user-attachments/files/34598880/proyecto-laboratorio.zip)
[README.md](https://github.com/user-attachments/files/34598895/README.md)
# De cero a una red segura: laboratorio de ciberseguridad

**Autor:** Virgilio Antonio Frías | **Período:** junio – julio 2026

Laboratorio de ciberseguridad construido desde cero en 7 días: un dominio de Active Directory, una estación de trabajo unida al dominio y un SIEM (Wazuh) que monitorea ambos equipos en tiempo real. Todo corre sobre Proxmox y está endurecido siguiendo CIS Benchmark.

> Todas las credenciales y contraseñas se omiten a propósito en este repositorio. Es un laboratorio, pero publicar contraseñas, aunque sean de prueba, es una mala práctica.

## Contenido

1. [Arquitectura](#arquitectura)
2. [Parte 1: Domain Controller](#parte-1--domain-controller)
3. [Parte 2: Usuarios, grupos y OUs](#parte-2--usuarios-grupos-y-ous)
4. [Parte 3: Estación de trabajo en el dominio](#parte-3--estación-de-trabajo-en-el-dominio)
5. [Parte 4: Servidor Wazuh](#parte-4--servidor-wazuh)
6. [Parte 5: Agentes de Wazuh](#parte-5--agentes-de-wazuh)
7. [Parte 6: Incidente, disco lleno](#parte-6--incidente-disco-lleno)
8. [Parte 7: Auditoría de eventos en Windows](#parte-7--auditoría-de-eventos-en-windows)
9. [Parte 8: Reglas de detección personalizadas](#parte-8--reglas-de-detección-personalizadas)
10. [Parte 9: GPOs (SMB y LDAP Signing)](#parte-9--gpos-smb-y-ldap-signing)
11. [Parte 10: Hardening](#parte-10--hardening)
12. [Parte 11: Validación de extremo a extremo](#parte-11--validación-de-extremo-a-extremo)
13. [Resumen y lecciones aprendidas](#resumen-final)

## Arquitectura

Tres máquinas virtuales sobre Proxmox VE 8.4:

| VM | Rol | Sistema | IP |
|---|---|---|---|
| DC01 | Domain Controller, DNS | Windows Server 2022 | 172.16.124.23 |
| CLIENT01 | Estación del empleado | Windows 10 | DHCP |
| vfrias | SIEM (Wazuh all-in-one) | Ubuntu Server 22.04 | 172.16.124.22 |

```
                 Proxmox VE
   ┌──────────────┬──────────────┬──────────────┐
   │ DC01         │ CLIENT01     │ vfrias       │
   │ AD DS + DNS  │ Windows 10   │ Wazuh        │
   │ .124.23      │ (dominio)    │ .124.22      │
   └──────┬───────┴──────┬───────┴──────▲───────┘
          └──── agentes Wazuh ──────────┘
```

![Máquinas virtuales en Proxmox](imagenes/01-proxmox-vms.png)

## Parte 1 — Domain Controller

![Server Manager con AD DS, DNS y File Services](imagenes/02-server-manager.png)

Instalé Windows Server 2022, le asigné IP fija (`172.16.124.23`) y lo promoví a Domain Controller del dominio `Vfrias.local`.

**Actualizaciones primero.** Antes de instalar roles actualicé el sistema por completo: las actualizaciones cierran vulnerabilidades que un atacante podría aprovechar.

**Instalación de AD DS.** Desde Server Manager instalé *Active Directory Domain Services* y luego usé la opción *Promote this server to a domain controller*. Durante la promoción:

- Nombre del dominio: `Vfrias.local`
- El servidor también asume el rol de DNS
- Se definió la contraseña de recuperación DSRM y se guardó en un lugar seguro

## Parte 2 — Usuarios, grupos y OUs

Estructura de Unidades Organizativas:

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

![Estructura de OUs en Active Directory Users and Computers](imagenes/03-estructura-ous.png)

### Grupos de seguridad

| Grupo | Descripción | Nivel de acceso |
|---|---|---|
| GG_IT_Admins | Soporte y administración | Control total |
| GG_Usuarios_IT | Personal de IT | Acceso parcial |
| GG_Usuarios_Operaciones | Empleados comunes | Solo lo que necesitan |
| GG_Acceso_Wazuh | Acceso a monitoreo | Uso futuro |

### Usuarios del dominio

| Usuario | Nombre | Rol |
|---|---|---|
| `cadmin` | Carlos Admin | Administrador de IT |
| `aiti` | Ana IT | Soporte técnico |
| `loperaciones` | Luis Operaciones | Empleado común |
| `mgarcia` | Maria Garcia | Empleado común |

Formato de inicio de sesión: `Vfrias\cadmin` o `cadmin@Vfrias.local`

### Políticas de contraseña diferenciadas

Los administradores tienen más privilegios, así que sus cuentas llevan reglas más estrictas:

| Política | Administradores | Usuarios comunes |
|---|---|---|
| Longitud mínima | 14 caracteres | 10 caracteres |
| Bloqueo tras intentos fallidos | 3 intentos | 5 intentos |
| Duración del bloqueo | 30 minutos | 15 minutos |
| Vigencia máxima | 30 días | 60 días |
| Historial | 10 contraseñas | 5 contraseñas |

## Parte 3 — Estación de trabajo en el dominio

Creé una VM con Windows 10 para representar la computadora de un empleado y la unida al dominio para que las políticas se apliquen automáticamente.

**Problema:** el primer intento falló con:

```
An Active Directory Domain Controller for the domain Vfrias.local could not be contacted
```

**Causa:** el cliente no sabía dónde encontrar el controlador de dominio porque su DNS apuntaba a otra dirección.

**Solución:** configuré el DNS del cliente para que apuntara directamente a DC01 (`172.16.124.23`). Después de ese ajuste, la unión al dominio funcionó de inmediato (con una cuenta administrativa del dominio) y el equipo se reinició.

![Pantalla de login con usuarios del dominio VFRIAS](imagenes/04-login-dominio.png)

En la pantalla de login ya aparecían usuarios de dominio como `VFRIAS\mgarcia`.

## Parte 4 — Servidor Wazuh

Wazuh es un SIEM de código abierto que recolecta eventos de las computadoras conectadas y genera alertas ante actividad sospechosa.

**Sistema base:** Ubuntu Server 22.04 sin interfaz gráfica, porque Wazuh se administra desde el navegador. Consume menos recursos y reduce la superficie de ataque.

- Hostname: `vfrias`
- IP: `172.16.124.22`
- OpenSSH instalado para administración remota

**Instalación:** script oficial en modo all-in-one, que instala en una sola máquina los tres componentes:

- **Manager:** procesa los eventos
- **Indexer:** base de datos donde se almacenan
- **Dashboard:** interfaz web

```bash
curl -sO https://packages.wazuh.com/4.9/wazuh-install.sh
sudo bash wazuh-install.sh -a
```

Al terminar, el instalador muestra las credenciales del panel una sola vez. Hay que guardarlas en ese momento.

![Login de Wazuh](imagenes/05-wazuh-login.png)

**Errores durante la instalación:**

1. Escribí un cero (`0`) en lugar de la letra `O` en el flag de `curl`, así que el archivo nunca se descargó.
2. Intenté usar `vfrias.local` como hostname. El instalador no acepta puntos en el hostname, así que usé solo `vfrias`.

![Dashboard de Wazuh](imagenes/06-wazuh-dashboard.png)

## Parte 5 — Agentes de Wazuh

El agente es un programa pequeño instalado en cada equipo que envía sus eventos al manager.

### DC01 (Windows Server)

Generé el comando de instalación desde el panel de Wazuh y lo ejecuté en PowerShell:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.9.2-1.msi -OutFile $env:tmp\wazuh-agent.msi
msiexec.exe /i $env:tmp\wazuh-agent.msi /q WAZUH_MANAGER='172.16.124.22' WAZUH_AGENT_NAME='DC01'
NET START WazuhSvc
```

![Panel de agentes de Wazuh](imagenes/07-agentes-panel.png)

**Problema:** el agente se instaló pero nunca aparecía como activo. El log del agente mostraba:

```
ERROR: Unable to connect to enrollment service at 'Vfrias.local':1515
```

**Causa:** en la configuración del agente había quedado el nombre del dominio (`Vfrias.local`) como dirección del manager. Ese nombre resuelve al controlador de dominio de Windows, no al servidor Ubuntu.

**Solución:** cambié la configuración para usar la IP directa del manager (`172.16.124.22`) y reinicié el agente. La conexión se estableció en segundos.

**Problema adicional, agente duplicado:** aparecieron dos registros para el mismo equipo: `DC01` (ID 001, "Never connected") y `WIN-LM3LJQOKKTL` (ID 002, "Active"). El primero era un registro huérfano del intento fallido. Lo eliminé desde el manager:

```bash
sudo /var/ossec/bin/manage_agents
# Opción R → ID 001 → confirmar → Q para salir
```

### CLIENT01 (Windows 10)

Mismo proceso, esta vez con `WAZUH_AGENT_NAME='CLIENT01'` y `WAZUH_MANAGER='172.16.124.22'` desde el inicio. Ambos agentes quedaron activos.

## Parte 6 — Incidente: disco lleno

En un momento dado el dashboard dejó de cargar y mostraba solo *"Something went wrong"* y *"Please set up API credentials"*.

![Conexión SSH al servidor Wazuh](imagenes/08-ssh-wazuh.png)

**Diagnóstico:** me conecté por SSH y revisé el disco:

```bash
df -h
# /dev/sda2   32G   32G   0   100%
```

Sin espacio libre el sistema no podía escribir logs, datos ni archivos temporales, y los servicios fallaban en cascada.

**Primer intento (fallido):** en Proxmox añadí un disco nuevo en lugar de ampliar el existente. El disco vacío de 54 GB no ayudaba a la partición llena. Intenté mover los datos de Wazuh a ese disco, pero con 0 bytes libres ni siquiera los comandos de copia podían crear archivos temporales.

**Solución correcta:**

1. Eliminé el disco añadido por error.
2. En Proxmox usé *Resize disk* sobre el disco original, de 32 GB a 59 GB.
3. `growpart` falló porque necesitaba un archivo temporal en el mismo disco lleno. Redirigí el directorio temporal a RAM:

```bash
sudo TMPDIR=/run growpart /dev/sda 2
sudo resize2fs /dev/sda2
```

Resultado: de 0 bytes libres a 24 GB disponibles. Los servicios de Wazuh se recuperaron.

**Lección:** un error en la aplicación no siempre tiene su causa en la aplicación. Hay que diagnosticar por capas, desde lo más básico (espacio en disco, servicios activos) hacia lo más complejo. Aquí la causa estaba en la infraestructura.

## Parte 7 — Auditoría de eventos en Windows

Por defecto Windows no registra todos los intentos de inicio de sesión. Habilité la auditoría en DC01:

```powershell
auditpol /set /category:"Logon/Logoff" /success:enable /failure:enable
auditpol /set /category:"Account Management" /success:enable /failure:enable
auditpol /set /category:"Account Logon" /success:enable /failure:enable
auditpol /set /subcategory:"Kerberos Authentication Service" /success:enable /failure:enable
```

Con esto, cada inicio de sesión exitoso o fallido queda en el Event Log y es visible para el agente de Wazuh.

## Parte 8 — Reglas de detección personalizadas

Además de las reglas predefinidas de Wazuh, creé reglas propias en `/var/ossec/etc/rules/local_rules.xml`:

| Categoría | Evento | Event ID | Nivel |
|---|---|---|---|
| Autenticación | Login fallido | 4625 | 8 |
| Autenticación | Fuerza bruta (5 fallos en 2 min, mismo usuario) | 4625 | 12 |
| Autenticación | Login exitoso | 4624 | 5 |
| Autenticación | Cuenta bloqueada | 4740 | 10 |
| Active Directory | Usuario creado | 4720 | 8 |
| Active Directory | Usuario eliminado | 4726 | 10 |
| Active Directory | Usuario agregado a grupo privilegiado | 4728/4732/4756 | 10 |
| Active Directory | Privilegios especiales asignados | 4672 | 12 |
| Sistema | Política de auditoría modificada | 4719 | 10 |
| Sistema | Tarea programada creada o modificada | 4698/4702 | 12 |
| Sistema | Nuevo servicio instalado | 7045 | 10 |
| Kerberos | Cuenta deshabilitada intentó autenticarse | 4768 | 10 |
| Kerberos | Posible Kerberoasting (cifrado RC4) | 4769 | 12 |
| Sysmon | Ejecución de procesos nuevos | 1 | 8 |
| Sysmon | Conexiones de red iniciadas por procesos | 3 | 8 |
| Sysmon | Modificaciones al Registro | 12/13 | 7 |
| Sysmon | Posible uso de Mimikatz | n/a | 12 |
| Sysmon | PowerShell con comandos en Base64 | n/a | 10 |

Para cargarlas:

```bash
sudo systemctl restart wazuh-manager
```

## Parte 9 — GPOs (SMB y LDAP Signing)

Las GPOs se definen una vez en el servidor y se aplican a todas las computadoras del dominio.

### SMB Signing

Sin firma, alguien dentro de la red podría interceptar y manipular el tráfico de archivos compartidos (ataques tipo NTLM Relay). Con SMB Signing, cada paquete va firmado y cualquier alteración se detecta.

GPO **SMB Signing Policy**, vinculada al dominio:

| Política | Valor |
|---|---|
| Microsoft network client: Digitally sign communications (always) | Enabled |
| Microsoft network server: Digitally sign communications (always) | Enabled |

Se aplicó con `gpupdate /force` y se verificó:

```powershell
Get-SmbServerConfiguration | Select RequireSecuritySignature, EnableSecuritySignature
# RequireSecuritySignature : True
# EnableSecuritySignature  : True
```


### LDAP Signing

LDAP es el protocolo con el que las aplicaciones consultan Active Directory. Sin firma, un atacante podría falsificar o manipular esas consultas.

GPO **LDAP signing policy**:

| Política | Valor |
|---|---|
| Domain controller: LDAP server signing requirements | Require signing |

**Problema:** tras aplicar la GPO el valor del registro no se actualizó de inmediato. Lo forcé directamente:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\NTDS\Parameters" -Name "LDAPServerIntegrity" -Value 2
```

| Valor | Comportamiento |
|---|---|
| 0 | Sin requerimiento: acepta todo |
| 1 | Negocia: prefiere firma pero no la exige |
| 2 | Obligatorio: rechaza conexiones sin firma |

Verificación final: `LDAPServerIntegrity : 2`.


## Parte 10 — Hardening

Apliqué controles basados en CIS Benchmark.

### DC01

**Política de contraseñas del dominio:**

| Parámetro | Valor anterior | Valor nuevo |
|---|---|---|
| Longitud mínima | 7 caracteres | 14 caracteres |
| Bloqueo tras intentos | Sin bloqueo | 5 intentos |
| Duración del bloqueo | Sin límite | 15 minutos |
| Vigencia máxima | 42 días | 60 días |

**Deshabilitar NTLMv1 y LM.** Son protocolos de los años 90 que siguen activos por compatibilidad y cuyas contraseñas se descifran con facilidad. Configuré el servidor para aceptar solo NTLMv2:

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Lsa" -Name "LmCompatibilityLevel" -Value 5
```

**Renombrar la cuenta Administrator**, objetivo habitual de ataques automatizados, a `AdminVfrias`. En un Domain Controller la cuenta es de dominio, por lo que se renombra con el módulo de Active Directory:

```powershell
$admin = Get-ADUser -Identity "Administrator"
Set-ADUser -Identity $admin -SamAccountName "AdminVfrias" -UserPrincipalName "AdminVfrias@Vfrias.local"
Rename-ADObject -Identity $admin.DistinguishedName -NewName "AdminVfrias"
```

![Inicio de sesión con la cuenta renombrada](imagenes/09-login-adminvfrias.png)

**Prueba involuntaria de las políticas:** durante las pruebas de generación de eventos, la cuenta `AdminVfrias` se bloqueó tras 5 intentos fallidos, exactamente como estaba configurado. La contraseña se restableció y el sistema siguió operando con normalidad.

| Medida | Estado |
|---|---|
| Contraseña mínima de 14 caracteres | ✅ |
| Bloqueo tras 5 intentos fallidos | ✅ |
| Duración de bloqueo de 15 minutos | ✅ |
| NTLMv1 y LM deshabilitados | ✅ |
| Cuenta Administrator renombrada | ✅ |

### CLIENT01

Se aplicaron los mismos controles en la estación de trabajo: un solo equipo sin hardening puede ser el punto de entrada que invalide los controles del servidor.

| Medida | Estado |
|---|---|
| NTLMv1 y LM deshabilitados | ✅ |
| Contraseña mínima de 14 caracteres | ✅ |
| Bloqueo tras 5 intentos fallidos | ✅ |
| Duración de bloqueo de 15 minutos | ✅ |
| Cuenta Administrator renombrada a `AdminCliente` | ✅ |
| Cuentas Guest, DefaultAccount y WDAGUtilityAccount deshabilitadas | ✅ |

> Referencia: CIS Microsoft Windows Server 2022 Benchmark y CIS Microsoft Windows 10 Benchmark.

## Parte 11 — Validación de extremo a extremo

Escenario: intentar iniciar sesión varias veces con contraseña incorrecta, lo que haría un atacante adivinando contraseñas.

| Métrica (último día) | Valor |
|---|---|
| Total de eventos | 3,260 |
| Intentos de autenticación fallidos | 4 |
| Autenticaciones exitosas | 1,242 |

Filtrando por Event ID 4625 (login fallido) obtuve el detalle de cada intento:

| Campo | Valor |
|---|---|
| Agente | WIN-LM3LJQOKKTL (DC01) |
| IP del agente | 172.16.124.23 |
| Motivo del fallo | Contraseña incorrecta |
| Tipo de login | 7 (desbloqueo de pantalla) |
| Proceso | svchost.exe |
| Dominio | VFRIAS |

El módulo MITRE ATT&CK del dashboard clasificó automáticamente las tácticas detectadas: *Valid Accounts*, *Domain Policy Modification* y *Account Access Restriction*.


Flujo validado:

```
DC01 (Windows Event Log)
  → Agente Wazuh
    → Manager
      → Indexer
        → Dashboard
          → Alerta visible en menos de 1 minuto
```

## Resumen final

| Componente | Tecnología | Estado |
|---|---|---|
| Servidor principal | Windows Server 2022 (Domain Controller) | ✅ Operativo |
| Dominio | Active Directory `Vfrias.local` con OUs, grupos y usuarios | ✅ Configurado |
| Estación de trabajo | Windows 10 unido al dominio | ✅ Operativo |
| SIEM | Wazuh all-in-one sobre Ubuntu 22.04 | ✅ Operativo |
| Agentes | DC01 + CLIENT01 | ✅ Activos |
| Tráfico de red | SMB Signing (GPO) | ✅ Activo |
| Directorio | LDAP Signing (GPO + registro) | ✅ Activo |
| Endurecimiento | CIS Benchmark (controles priorizados) | ✅ Aplicado |
| Detección | 20+ reglas personalizadas en `local_rules.xml` | ✅ Activas |

### Lecciones aprendidas

Los problemas surgidos durante el proyecto (disco lleno, agente que no conectaba, error de DNS, agente duplicado) se diagnosticaron, se resolvieron y se documentaron. Los sistemas no funcionan perfectos al primer intento, y saber leer un error y encontrar su causa es tan importante como conocer el comando correcto.

### Credenciales

Por seguridad, las credenciales del laboratorio no se publican en este repositorio.
