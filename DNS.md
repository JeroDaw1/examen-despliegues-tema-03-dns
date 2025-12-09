# Configuración de nombres de dominio de una DMZ

## Configuración de DNS

Configurar el archivo /etc/netplan/50-cloud-init.yaml

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      # Usar sólo cuando conexión por IP estática no es accesible
      # dhcp4: true
      addresses: [10.0.2.15/24] # Se obtiene con `ip a`
    routes:
      - to: deault
      via: 10.0.2.2 # Se obtiene con 'iproute' o 'ip route'
    nameservers:
      addresses: [10.0.2.15] # Se obtiene con resolvectl
      search: [jgsdaw.local]
    ens0s8:
      # dhcp4: true
      addresses: [192.168.7.50/24]
    # Usar sólo cuando conexión puente es necesario, sino comentar
    ens0s9:
      dhcp4: true
```

Configurar el archivo /etc/netplan/50-cloud-init.yaml de las máquinas gitlab, web y ftp:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [192.168.7.40/24] # 40 es de webserver o la que de el profesor
    nameservers:
      addresses: [192.168.7.50] # Dirección del DNS
      search: [jgsdaw.local]
    routes:
      - to: deault
      via: 192.168.7.50 # Dirección del DNS
```

Aplicar cambios:

```bash
netplan generate
netplan apply
resolvectl
```

Actualizar archivo de configuraión DNS:

```bash
sudo nano /etc/bind/named.conf.options
```

```bash
options {
    directory "/var/cache/bind";114
    ...
    forwarders {
        // 0.0.0.0
        10.2.1.254 # Se obtiene con 'resolvectl'
        // 8.8.8.8
    };
    ...
};
```

Da esto:

```bash
enp0s3:
...
DNS Servers: 10.0.2.114
DNS Domain: jgsdaw.local
...
```

Copiar el archivo de named.conf.local

```bash
cp /etc/named.conf.local /etc/named.conf.local_091225
```

Crear el archivo /etc/named.conf.local

```bash
cd /etc/bind/
sudo cp named.conf.local named.conf.local_091225
sudo nano named.conf.local
```

```bash
zone "jgsdaw.local" {
    type master;
    file "/etc/bind/zonas/db.jgsdaw.local";
};

zone "7.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zonas/db.7.168.192.in-addr.arpa";
};
```

Crear el directorio y los archivos de zonas:

```bash
mkdir zonas
sudo cp db.local zonas/db.jgsdaw.local
cd zonas
sudo nano db.jgsdaw.local
```

```bash
# Actualizar la dirección principal inicial
@       IN      SOA     ns1.jgsdaw.local. root.jgsdaw.local. (...

# Añadir las zonas
@             IN      NS      ns1
@             IN      A       192.168.7.250
ns1           IN      A       192.168.7.250
servidor      IN      A       192.168.7.250
jgscli        IN      A       192.168.7.XXX # IP obtenido con `ip a` en enp0s3
server        IN      A       192.168.7.250
web           IN      A       192.168.7.40 # Ejemplo, puede ser la 150 o cualquiera en vez de 40
ftp...
gitlab...
```

Copiar un archivo de nombres de zona con la dirección IP:

```bash
sudo cp db.jgs.local db.192.168.7.in-addr-arpa
sudo nano db.192.168.7.in-addr-arpa
```

```bash
# Actualizar direcciones y nombres inferiores
@             IN      NS      ns1.jgsdaw.local.
250           IN      PTR     ns1.jgsdaw.local.
XXX           IN      PTR     jgscli.jgsdaw.local. # Última cifra de la IP obtenida con `ip a` en enp0s3
40            IN      PTR     web.jgsdaw.local. # Ejemplo, puede ser la 150 o cualquiera en vez de 40
XXX           IN      PTR     ftp.jgsdaw.local. # IP de FTP o lo que toque, dado por el profesor
...
```

Ir al root de la consola y configurar las zonas:

```bash
cd ..
cd ..
sudo named-checkzone jgsdaw.local /etc/bind/zonas/db.jgsdaw.local
sudo named-checkzone 7.168.192.in-addr.arpa /etc/bind/zonas/db.7.168.192.in-addr.arpa
```

Reiniciar servicio Bind9:

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

Comprobar la conexión a ns1 con nslookup:

```bash
nslookup ns1
nslookup ns1.jgsdaw.local
nslookup jgscli
nslookup 192.168.7.250
```

Acceder por SSH al servidor comprobando nombre de dominio:

```bash
cat /etc/resolve.conf
ssh ems@server
```

## Configuración de WebServer

Comprobar en el servidor web Apache2:

```bash
sudo systemctl status apache2
```

Ir a Bind9
