# OpenVPN

El objetivo del trabajo consta de implementar un servicio de OpenVPN con tres arquitecturas distintas y mostrar su funcionamiento.

Las arquitecturas propuestas son:
- Cliente a sitio
- Sitio a sitio
- Multisitio (Full-Mesh)

Todas las implementaciones se realizaron en el entorno de AWS provisto por la cátedra.

# Cliente a sitio
## Instalación de Servidor OpenVPN
Se deben instalar todo lo necesario:
```bash
sudo apt-get update
sudo apt-get upgrade -y
sudo apt-get install openvpn easy-rsa net-tools -y
```

Luego, se crearán los certificados necesarios para autenticar al cliente y servidor. Para esto crearemos un nuevo directorio para configurar la autoridad certificante (CA):
```bash
make-cadir ~/openvpn-ca
```
Se procede a crear las claves privadas y certificados para el servidor y clientes:
* Inicialización del directorio PKI
  ```bash
  ./easyrsa init-pki
  ```
  Se almacenan las claves y certificados.

* Creación de la autoridad certificante
  ```bash
  ./easyrsa build-ca nopass
  ```
  El comando solicitará un common name (en este caso server) y se obtendrá como resultado un par de claves y un certificado de la CA.

* Generacion de una clave privada para el servidor y una solicitud de certificado
  ```bash
  ./easyrsa gen-req server nopass
  ```
* Firma la solicitud de certificado del servidor utilizando la CA
  ```bash
  ./easyrsa sign-req server server
  ```
* Generación de parámetros de Diffie-Hellman para el intercambio de claves de forma segura:
  ```bash
  ./easyrsa gen-dh
  ```
* Generación de una clave que será usada por el servidor y los clientes para prevenir ataques DoS y Man in the Middle
  ```bash
  openvpn --genkey --secret ta.key
  ```
* Generación de una clave privada para el cliente y una solicitud de certificado
  ```bash
  ./easyrsa gen-req client1 nopass
  ```
  Se genera una clave privada para el cliente (client1) y una solicitud de certificado
* Firma de la solicitud de certificado del cliente utilizando la CA
  ```bash
  ./easyrsa sign-req client client1
  ```
  Se firma la solicitud de certificado del cliente (client1) utilizando la CA.

**Archivos generados**
```bash
pki/private/ca.key: Clave privada de la CA.
pki/ca.crt: Certificado de la CA.
```
```bash
pki/private/server.key: Clave privada del servidor.
pki/issued/server.crt: Certificado del servidor.
pki/dh.pem: Parámetros Diffie-Hellman.
```

```bash
pki/private/client1.key: Clave privada de client1.
pki/issued/client1.crt: Certificado de client1.
```

```bash
ta.key: Clave secreta para TLS Auth.
```

### Configuración del servidor
Para el archivo de configuración del servidor (server.conf) se puede partir de un ejemplo que ofrece openVPN. Para eso, lo copiamos a nuestro directorio de configuración:
```bash
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/
```
Luego, se procede a editar el archivo de configuración
```bash
# Puerto en el que el servidor escuchará las conexiones de los clientes
port 1194
# Protocolo que se utilizará para la conexión
proto tcp
# dispositivo de red utilizado para túneles IP
dev tun

# Rutas a certificados y claves
ca /etc/openvpn/ca.crt
cert /etc/openvpn/server.crt
key /etc/openvpn/server.key
dh /etc/openvpn/dh.pem

# Define la red virtual
server 10.8.0.0 255.255.255.0

# Directorio al archivo en el que se guardará la asignación de direcciones IP a los clientes conectados
ifconfig-pool-persist /etc/openvpn/ipp.txt

# Pushea configuraciones a los clientes, en este caso la configuración del servidor DNS
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"

# Define la red propia del servidor para los clientes
push "route 10.0.0.0 255.255.240.0"

# Frecuencia de los paquetes de "keepalive"
keepalive 10 120

# Algoritmo de cifrado
cipher AES-256-CBC

# Algoritmo de autenticación
auth SHA256

# Habilita la autenticación de TLS
tls-auth /etc/openvpn/ta.key 0

Especifica la dirección de la clave. En este caso, la dirección de la clave es 0, lo que indica que la clave se utiliza tanto para la autenticación del servidor como para la autenticación del cliente
key-direction 0

# Usuario y grupo utilizados para limitar privilegios del servidor
user nobody
group nogroup

# Indica que OpenVPN debe intentar mantener abiertos el archivo de clave y el túnel persistente
persist-key
persist-tun

# Archivos de log
status /var/log/openvpn-status.log
log-append /var/log/openvpn.log

# Nivel de loggeo
verb 3

# Indica la topología de la red utilizada
topology subnet

```

Debemos copiar las claves y certificados previamente generados al directorio donde se encuentra la configuración:
```bash
sudo cp ~/openvpn-ca/pki/ca.crt ~/openvpn-ca/pki/private/server.key ~/openvpn-ca/pki/issued/server.crt ~/openvpn-ca/pki/dh.pem ~/openvpn-ca/ta.key /etc/openvpn/
```

Es necesario habilitar el reenvío de IP para permitir que los paquetes de datos se envíen desde una interfaz de red a otra. Para esto, se debe editar el archivo `/etc/sysctl.conf` y descomentar la línea:
```bash
net.ipv4.ip_forward=1
```
Luego, se deben aplicar los cambios:
```bash
sudo sysctl -p
```
Adicionalmente, se debe hacer NAT de los paquetes que lleguen del túnel para que los clientes puedan llegar a  internet:
```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```
Es posible que se deba cambiar la interfaz, consultar con `ifconfig`

Esta configuración no persiste y se debe realizar con cada reinicio

Finalmente se inicia el servicio de OpenVPN:
```bash
sudo systemctl start openvpn@server
```

## Instalación de Cliente OpenVPN
Para la configuración del cliente se debe instalar OpenVPN, tal como se realizó con el servidor. Además, se deben copiar las claves generadas en el servidor (`ca.crt`, `client1.crt`, `client1.key`, `ta.key`) al cliente. Podemos compartir dichos archivos utilizando SCP:
```bash
scp /ruta/del/archivo usuario_remoto@host_remoto:/ruta/destino/
```
Análogamente, copiaremos el template de configuración del cliente al directorio `/etc/openvpn`:
```bash
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /etc/openvpn/
```
Procedemos a editar la configuración
```bash
# Indica que actua como cliente
client

dev tun
proto tcp

#Conexion al servidor
remote 3.230.53.184 1194

# Reintentar infinitamente la resolución de nombres de host
resolv-retry infinite
# El cliente no debe intentar vincularse a una dirección IP o puerto específico
nobind

persist-key
persist-tun

ca "/etc/openvpn/ca.crt"
cert "/etc/openvpn/remote-server.crt"
key "/etc/openvpn/remote-server.key"

# Indica que el cliente debe verificar el certificado del servidor
remote-cert-tls server

tls-auth "/etc/openvpn/ta.key" 1
key-direction 1
cipher AES-256-CBC

verb 3
auth SHA256

```
Finalmente, se debe iniciar el servicio:
```bash
sudo systemctl start openvpn@client
```

# Sitio a sitio
Para la nueva arquitectura, se poseen dos redes distintas que no poseen conexión entre sí, el servidor se encuentra en la red 10.0.0.0/20 y el servidor remoto (un cliente openVPN) se encuentra en la red 192.168.0.0/20.
En este caso, tanto el servidor como el cliente deberán reenviar el tráfico proveniente del túnel VPN a los demás hosts de la red. Para esto, se deben definir reglas de enrutamiento en el cliente, en el servidor y en el default gateway de ambas redes.

## Cambios al servidor
Para que la conexión sitio a sitio funcione correctamente, se debe modificar el archivo de configuración (`server.conf`). Inicialmente. se debe indicar que los paquetes con destino a la red del cliente, se envían por el túnel VPN:
```bash
route 192.168.0.0 255.255.240.0
```
Además, se debe indicar el directorio de configuración de clientes
```bash
client-config-dir ccd
```
Luego, creamos dicho directorio `ccd` y agregamos un archivo con el common name que le dimos a nuestro cliente, en el agregaremos la siguiente configuración:
```bash
iroute 192.168.0.0 255.255.240.0
```
Con esta  instrucción, le indicamos al servidor que los paquetes destinados a esa subred deben ser enviados a través del túnel VPN

## Cambios al cliente
Nuestro cliente ahora actuará como un servidor remoto, por lo que en primer lugar habilitaremos el forwardeo de paquetes como se hizo con el servidor en la explicación de la sección anterior:
* Agregar la linea `net.ipv4.ip_forward=1` al archivo `/etc/sysctl.conf` y aplicar los cambios con:
```bash
sudo sysctl -p
```
* Se debe hacer NAT de los paquetes que lleguen del túnel:
```bash
sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```
Recordar cambiar de interfaz en caso de ser necesario, consultar con `ifconfig`

## Cambios en las tablas de ruteo
Se deben agregar en ambos routers las reglas de ruteo para que los paquetes lleguen correctamente.  
Se debe agregar una regla para los paquetes de la red remota y otra para los paquetes con destino a la red VPN, ambos deben viajar a través del túnel VPN. Para esto, se debe setear como gateway el servidor de cada red.

# Multisitio
Una vez teniendo configurada la conexión sitio a sitio, agregar un nuevo sitio no requiere mayor complejidad. 
A la configuración de nuestro servidor principal se le debe agregar la línea `client-to-client` para permitir la comunicación entre clientes.
Se debe crear un nuevo archivo de configuración en el directorio ccd (análogamente a como se hizo en la configuración anterior), con el common name del nuevo sitio. Además, a todos los archivos de configuración del directorio ccd se deben agregar la instrucción `push "route 172.16.0.0 255.255.240.0"` con la dirección de los otros sitios remotos. Es decir, el archivo de configuración del servidor de la red 192.168.0.0/20 tendra la linea `push "route 172.16.0.0 255.255.240.0"` y el servidor de la red 172.16.0.0/20 tendra la instruccion "route 192.168.0.0 255.255.240.0"`.
Por último, a las tablas de ruteo de los routers de los sitios se le debe agregar el enrutamiento a través del túnel para los paquetes que tengan como destino la nueva red y viceversa.
