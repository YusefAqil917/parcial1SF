# parcial1ST
# Parcial 1 - Primera parte

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

# Servidor maestro

## Configuración zona directa

En el servidor **maestro**, y en el archivo `named.conf.local` ubicado en `/etc/bind`

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

En el archivo [`db.](http://db.maestro.com)empresa.local` ubicado en `/etc/bind`.

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

En el archivo `db.192` 

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

; Registros PTR (Resolución Inversa)
3       IN  PTR  maestro.empresa.local.
2       IN  PTR  esclavo.empresa.local.
4       IN  PTR  cliente.empresa.local.
```

### Verificación de archivos

`named-checkzone [empresa.local](http://servicios.com/) /etc/bind/db.empresa.local`

`named-checkzone 50.168.192.in-addr.arpa /etc/bind/db.192`

### Se desactiva el firewall

Para consultas externas

`sudo ufw disable`

# Servidor esclavo

En el servidor **esclavo**, y en el archivo `named.conf.local` ubicado en `/etc/bind`

```bash
// Zona directa (empresa.local)
zone "empresa.local" {
    type slave;
    file "/var/cache/bind/db.empresa.local";
    masters { 192.168.50.3; };
};

// Zona inversa (192.168.50.x)
zone "50.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.192";
    masters { 192.168.50.3; };
};
```

## Refrescar el caché del servidor esclavo

```bash
sudo rndc retransfer empresa.local
sudo rndc retransfer 50.168.192.in-addr.arpa
```

# Pruebas

Todas las pruebas se hacen desde el **cliente.** Con estas pruebas buscamos comprobar que si usamos como DNS el servidor **esclavo**, siempre se va a resolver el dominio gracias al cache que trae del servidor **maestro**.

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

## Configurar archivo resolv.conf

En el Esclavo se apunta el resolver hacia la dirección IP del maestro 192.168.50.3

En el cliente se apunta el resolver hacia la dirección IP del Esclavo 192.168.50.2

## Para parar el bind

sudo systemctl stop bind9
