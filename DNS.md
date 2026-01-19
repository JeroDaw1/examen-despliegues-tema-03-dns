# Configuración de nombres de dominio de una DMZ

## Configuración de DNS

Configurar el archivo /etc/netplan/50-cloud-init.yaml

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.0.2.15/24]
      routes:
        - to: default # Corregido: default
          via: 10.0.2.2
    ens0s8:
      addresses: [192.168.7.50/24] # Usa .50 o .250, pero sé consistente
      nameservers:
        addresses: [127.0.0.1] # El servidor DNS se consulta a sí mismo
        search: [jgsdaw.local]
    ens0s9:
      dhcp4: true
```

Configurar el archivo /etc/netplan/50-cloud-init.yaml de las máquinas gitlab, web y ftp:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [192.168.7.40/24]
      nameservers:
        addresses: [192.168.7.50] # IP del servidor DNS
        search: [jgsdaw.local]
      routes:
        - to: default # Corregido: antes deault
          via: 192.168.7.50
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
    directory "/var/cache/bind";
    ...
    listen-on { any; };
    allow-query { localhost; 192.168.XX.0/24; };

    forwarders {
        // 0.0.0.0
        10.2.1.254; # Se obtiene con 'resolvectl'
        // 8.8.8.8
    };

    dnssec-validation no;
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
// Zona directa
zone "jgsdaw.local" {
    type master;
    file "/etc/bind/zonas/db.jgsdaw.local";
};

// Zona inversa
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
@             IN      A       192.168.XX.50
ns1           IN      A       192.168.XX.50
servidor      IN      A       192.168.XX.50
server        IN      CNAME   servidor
gitlab        IN      A       192.168.XX.100
web           IN      A       192.168.XX.150
ftp           IN      A       192.168.XX.200
jgscli        IN      A       192.168.XX.XXX ; # IP obtenido con `ip a` en enp0s3
```

Copiar un archivo de nombres de zona con la dirección IP:

```bash
sudo cp db.jgs.local db.7.168.192.in-addr.arpa
sudo nano db.7.168.192.in-addr.arpa
```

```bash
# Actualizar direcciones y nombres inferiores
@       IN      SOA     ns1.jgsdaw.local. root.jgsdaw.local. (...

// Añadir las zonas
@             IN      NS      ns1.jgsdaw.local.
50            IN      PTR     ns1.jgsdaw.local.
50            IN      PTR     servidor.jgsdaw.local.
100           IN      PTR     gitlab.jgsdaw.local.
150           IN      PTR     web.jgsdaw.local.
200           IN      PTR     ftp.jgsdaw.local.
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
nslookup 192.168.7.50
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
