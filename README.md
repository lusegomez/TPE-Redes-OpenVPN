# OpenVPN

El objetivo del trabajo consta de implementar un servicio de OpenVPN con tres arquitecturas distintas y mostrar su funcionamiento.

Las arquitecturas propuestas son:
- Cliente a sitio
- Sitio a sitio
- Multisitio (Full-Mesh)

Todas las implementaciones se realizaron en el entorno de AWS provisto por la cátedra.

## Crear VPC
## Crear instancia EC2

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

# Indica que el trafico destinado a esta red viajara a traves de la VPN
route 192.168.0.0 255.255.240.0

# Indica el directorio donde se encuentran los archivos de configuración  de cada cliente
client-config-dir ccd

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
