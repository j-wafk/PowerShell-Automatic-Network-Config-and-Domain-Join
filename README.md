# PowerShell-Automatic-Network-Config-and-Domain-Join
# Script de configuración de red y unión al dominio

Este script de PowerShell está diseñado para cambiar la configuración de red de un equipo cliente, unirlo a un dominio y realizar un reinicio. Además, el servidor recibe un mensaje desde el cliente para realizar el cambio de contraseña de un usuario. A continuación se explica cómo funciona el script y los pasos que realiza.

## Descripción del script

### Cliente

 **Importante:**  
   Este script debe ejecutarse **como administrador** para que funcione correctamente, ya que realiza cambios en la configuración de red y en el dominio.

1. **Solicita al usuario la nueva dirección IP** que se asignará al equipo.
2. **Configura el servidor DNS** de la interfaz de red a una dirección específica.
3. **Verifica la conectividad** con el servidor DNS mediante un ping.
4. Si el ping es exitoso:
   - **Une el equipo al dominio** `servidor.local` usando credenciales predefinidas.
   - **Envía un mensaje UDP** al servidor para indicar que se debe cambiar la contraseña del usuario.
   - **Reinicia el equipo** para aplicar los cambios de dominio.
5. Si el ping falla, **registra un error** en un archivo de log.
   
  


### Servidor

1. El servidor escucha en el puerto UDP 2020 para recibir el mensaje del cliente.
2. Cuando recibe el mensaje "Cambiar", **cambia la contraseña** de la cuenta `creador` en el servidor esta cuenta debe tener permisos para añadir usuarios al dominio.
3. Luego, cierra la conexión UDP.

## Código del Script

### Cliente

```powershell
# Script para cambiar la configuración de red, añadir el equipo a un dominio y realizar un reinicio.

# Solicitar al usuario la nueva IP que se asignará al equipo cliente.
$nuevaIP = Read-Host "¿Cuál es la nueva IP del ordenador cliente?"

# Definir la dirección del servidor DNS.
$DNS_Server = "10.10.1.2" #Cambiar por el DNS del AD

# Cambiar la configuración de la dirección IP y el servidor DNS en la interfaz "Ethernet".
netsh interface ip set address name="Ethernet" static $nuevaIP
netsh interface ip set dns name="Ethernet" static $DNS_Server

# Esperar 30 segundos para que el sistema pueda aplicar la nueva configuración.
Start-Sleep -Seconds 30

# Verificar la conectividad con el servidor DNS configurado a través de un ping.
$ping = Test-Connection -ComputerName $DNS_Server -Count 1 -Quiet 
$ping
# Si el ping es exitoso, realizar las acciones de dominio.
if ($ping) {
    # Credenciales para unir el equipo al dominio.
    $Pass = ConvertTo-SecureString "Windows1234" -AsPlainText -Force
    $cred = New-Object System.Management.Automation.PSCredential ("creador", $Pass)
    
    # Unir el equipo al dominio "servidor.local".
	#Cambiar "servidor.local" por el nombre de tu AD
    Add-Computer -DomainName "servidor.local" -Credential $cred

    # Enviar un mensaje UDP al servidor para indicar que se debe cambiar la contraseña del usuario.
    $ip = New-Object System.Net.IPEndPoint ([IPAddress]$DNS_Server, 2020)
    $udp = New-Object System.Net.Sockets.UdpClient
    # Mensaje que se enviará al servidor
    #Se puede usar cualquier palabra siempre y cuando se ponga la misma en el cliente y en el servidor.
    $mensaje = [Text.Encoding]::ASCII.GetBytes('Cambiar')  
    $udp.Send($mensaje, $mensaje.Length, $ip) | Out-Null  # Enviar el mensaje.
    $udp.Close()  # Cerrar la conexión UDP.

    # Reiniciar el equipo para aplicar los cambios de dominio.
    Restart-Computer
} else {
    # Si el ping falla, registrar el error en un archivo de log.
    Write-Host ("No se pudo hacer ping a " + $DNS_Server) -ForegroundColor Red
    ("No se pudo hacer ping a " + $DNS_Server) >> FalloPingLog.txt
}
```
### Servidor

```powershell
# Script para recibir un mensaje UDP del cliente y cambiar la contraseña de un usuario.

# Configurar la escucha UDP en el servidor.
$ip = New-Object System.Net.IPEndPoint ([IPAddress]::Any, 0)
$udp = New-Object System.Net.Sockets.UdpClient 2020

# Esperar a recibir el mensaje "Cambiar" desde el cliente.
if("Cambiar" -eq [Text.Encoding]::ASCII.GetString($udp.Receive([ref]$ip))){

    # Cambiar la contraseña de la cuenta "creador" a una nueva contraseña.
    Set-ADAccountPassword -Identity "creador" -NewPassword (ConvertTo-SecureString -AsPlainText "Windows123456789" -Force) -Reset
    Write-host ("La contraseña se cambió de forma exitosa")
}

# Cerrar la conexión UDP.
$udp.Close()
```
