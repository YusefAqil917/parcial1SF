# Parcial1ST

# Primera parte
# Vagrantfile

```bash
# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.configure("2") do |config|
    config.vm.define :maestro do |maestro|
      maestro.vm.box = "bento/ubuntu-22.04"
      maestro.vm.network :private_network, ip: "192.168.50.3"
      maestro.vm.hostname = "maestro"
    end
  
    config.vm.define :esclavo do |esclavo|<
      esclavo.vm.box = "bento/ubuntu-22.04"
      esclavo.vm.network :private_network, ip: "192.168.50.2"
      esclavo.vm.hostname = "esclavo"
    end

    config.vm.define :cliente do |cliente|
        cliente.vm.box = "bento/ubuntu-22.04"
        cliente.vm.network :private_network, ip: "192.168.50.4"
        cliente.vm.hostname = "cliente"
    end
  end
```

# Configuración del entorno

Ejecutar:

```bash
sudo apt-get update
sudo apt-get install bind9 dnsutils
```

# Configuraciones en el Servidor maestro

## Configuración de zona directa 

En el archivo `named.conf.local` que ubicado en `/etc/bind` incluimos el siguiente contenido: 

```bash
// Zona directa (empresa.local)
zone "empresa.local" {
    type master;
    file "/etc/bind/db.empresa.local";
    allow-transfer { 192.168.50.2; }; // Permite la transferencia al esclavo
};

// Zona inversa (192.168.50.x)
zone "50.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192";
    allow-transfer { 192.168.50.2; }; // Permite la transferencia al esclavo
};
```

Ahoa, en el archivo [`db.](http://db.maestro.com)empresa.local` ubicado en `/etc/bind`.

```bash
$TTL 604800
@       IN  SOA     maestro.empresa.local. root.empresa.local. (
        2       ; Serial 
        604800  ; Refresh
        86400   ; Retry
        2419200 ; Expire
        604800 ) ; Negative Cache TTL

; Servidor de nombres
@       IN  NS  maestro.empresa.local.
maestro IN  A   192.168.50.3
esclavo IN  A   192.168.50.2
cliente IN  A   192.168.50.4
www     IN  A   192.168.50.3
mail    IN  CNAME maestro
server  IN  CNAME maestro

```

Para la configuracion de la resolución inversa en el archivo `db.192`  

```bash
$TTL 604800
@       IN  SOA     maestro.empresa.local. root.empresa.local. (
        2       ; Serial
        604800  ; Refresh
        86400   ; Retry
        2419200 ; Expire
        604800 ) ; Negative Cache TTL

; Servidor de nombres
@       IN  NS  maestro.empresa.local.

; Registros PTR (Para la resolución Inversa de la IP)
3       IN  PTR  maestro.empresa.local.
2       IN  PTR  esclavo.empresa.local.
4       IN  PTR  cliente.empresa.local.
```

### Verificación de archivos

`named-checkzone [empresa.local](http://servicios.com/) /etc/bind/db.empresa.local`

`named-checkzone 50.168.192.in-addr.arpa /etc/bind/db.192`

### Desactivamos el firewall para poder hacer las consultas externas

`sudo ufw disable`

# Configuraciones hechas en el servidor esclavo

Para el servidor **esclavo**, necesitamos configurar las zonas en el archivo `named.conf.local` ubicado en `/etc/bind`

```bash
// Zona directa
zone "empresa.local" {
    type slave;
    file "/var/cache/bind/db.empresa.local";
    masters { 192.168.50.3; };
};

// Zona inversa
zone "50.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.192";
    masters { 192.168.50.3; };
};
```
# Archivo resolv.conf
En Esclavo se apunta el resolver hacia la dirección IP del maestro 192.168.50.3

# Comprobaciones

Todas las pruebas se realizan desde el cliente. El objetivo es verificar que, al utilizar el servidor esclavo como DNS, el dominio siempre se resuelva correctamente gracias al caché que obtiene del servidor maestro.

```bash
nslookup maestro.empresa.local 192.168.50.2
nslookup www.empresa.local 192.168.50.2
nslookup esclavo.empresa.local 192.168.50.2
nslookup cliente.empresa.local 192.168.50.2
```

Para resolución inversa es lo mismo, pero en vez de dominios usamos IP’s

```bash
nslookup 192.168.50.3 192.168.50.2
nslookup 192.168.50.2 192.168.50.2
nslookup 192.168.50.4 192.168.50.2
```

# Archivo resolv.conf

En el cliente se apunta el resolver hacia la dirección IP del Esclavo 192.168.50.2


# Parcial - Segunda Parte
Se agrega un nuevo servidor al entorno con la siguiente configuración en Vagrantfile:

```ruby
config.vm.define :servidor do |servidor|
    servidor.vm.box = "bento/ubuntu-22.04"
    servidor.vm.network :private_network, ip: "192.168.50.5"
    servidor.vm.hostname = "servidor"
end
```
Configuración del Servidor Web y DNS
```bash

sudo apt-get update 
sudo apt-get install apache2 bind9 dnsutils
```

Habilitación de los Módulos de Compresión en Apache

```bash
sudo a2enmod deflate
sudo systemctl restart apache2

sudo a2enmod headers
sudo systemctl restart apache2
Configuración de la Zona DNS
En el archivo /etc/bind/named.conf.local:
```

Configuracion del modulo de deflate

```bash
<IfModule mod_deflate.c>
        <IfModule mod_filter.c>
                AddOutputFilterByType DEFLATE text/html text/plain text/xml text/css text/javascript
                AddOutputFilterByType DEFLATE application/x-javascript application/javascript application/ecmascript application/json
                AddOutputFilterByType DEFLATE application/rss+xml
                AddOutputFilterByType DEFLATE application/wasm
                AddOutputFilterByType DEFLATE application/xml
        </IfModule>
        # Regla para exclusión de  imágenes y videos
        SetEnvIfNoCase Request_URI \.(?:gif|jpe?g|png|mp4|avi|webp|mp3)$ no-gzip dont-vary
        Header append Vary Accept-Encoding
</IfModule>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

```bash
zone "apellido.com" {
    type master;
    file "/etc/bind/db.parcial.apellido.com";
};

// Zona inversa
zone "50.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192";
};
En /etc/bind/db.parcial.apellido.com:
```

```bash
$TTL 604800
@       IN  SOA     parcial.apellido.com. root.apellido.com. (
        2       ; Número de serie
        604800  ; Tiempo de actualización
        86400   ; Tiempo de reintento
        2419200 ; Tiempo de expiración
        604800 ) ; TTL negativo

@       IN  NS  parcial.apellido.com.
parcial IN  A   192.168.50.5
www     IN  A   192.168.50.5
Pruebas
Desde la máquina cliente:
```

```bash
curl -H "Accept-Encoding: gzip" -I http://parcial.apellido.com
```
# Parcial - Tercera Parte
## Creación de un Túnel con Ngrok
En la máquina servidor, instalar Ngrok:
```bash
curl -sSL https://ngrok-agent.s3.amazonaws.com/ngrok.asc \
  | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null \
  && echo "deb https://ngrok-agent.s3.amazonaws.com buster main" \
  | sudo tee /etc/apt/sources.list.d/ngrok.list \
  && sudo apt update \
  && sudo apt install ngrok
```

Configurar Ngrok con el token de autenticación:

```bash
ngrok config add-authtoken <token>
```
Iniciar el túnel:

```bash
ngrok http 80
```

Editar el archivo /var/www/html/index.html y personalizar el contenido:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Bienvenido</title>
</head>
<body>
  <h1>Bienvenido a la página personalizada de <b>"Santiago Cortes/Yusef Aqil"</b></h1>
</body>
</html>
```
Con esta configuración, se ha implementado un servicio telemático funcional con DNS, servidor web y un túnel público mediante Ngrok.

