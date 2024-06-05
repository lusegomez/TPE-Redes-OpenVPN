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
