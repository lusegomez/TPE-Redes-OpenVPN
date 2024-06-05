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
