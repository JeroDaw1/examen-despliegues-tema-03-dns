# Configuración de nombres de dominio de una DMZ

## 1. Preparando el entorno

### 1.1. Comprobación del acceso a Internet de todos los servidores

Configurar el archivo `/etc/netplan/50-cloud-init.yaml` en el servidor DNS (ns1):

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [10.0.2.15/24] # ip a (NAT) y lo da con todo en DHCP4 en true
      routes:
        - to: default
          via: 10.0.2.2 # ip route y lo da con todo en DHCP4 en true
    ens0s8:
      addresses: [192.168.XX.50/24] # IP estática del servidor DNS (.50)
      nameservers:
        addresses: [127.0.0.1] # El servidor DNS se consulta a sí mismo
        search: [dawjgs.local]
    ens0s9:
      dhcp4: true
```

Luego hacer:

```bash
sudo resolvectl
```

Configurar el archivo `/etc/netplan/50-cloud-init.yaml` de las máquinas gitlab, web y ftp:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      addresses: [192.168.XX.YY/24] # IP variable según servidor
      nameservers:
        addresses: [192.168.XX.50] # Dirección del servidor DNS ns1
        search: [dawjgs.local]
      routes:
        - to: default # Corregido: antes deault
          via: 192.168.XX.50
```

Aplicar cambios y comprobar resolución de nombres externa:

```bash
sudo netplan generate
sudo netplan apply
# Comprobación de acceso a internet mediante resolución (LO 5-f)
nslookup www.google.es
```

## 2. Configurando nuestro servidor DNS

### 2.1. Configuración DNS (Servicio DNS)

Actualizar archivo de configuración `/etc/bind/named.conf.options`:

```bash
options {
    directory "/var/cache/bind";
    ...
    listen-on { any; };
    allow-query { localhost; 192.168.XX.0/24; };

    forwarders {
        // 0.0.0.0
        10.2.1.254; # IP de salida a Internet obtenida con resolvectl
        // 8.8.8.8
    };

    dnssec-validation no;
};
```

### 2.2. Definición de las zonas

Editar el archivo `/etc/bind/named.conf.local`:

```bash
// Zona directa
zone "dawjgs.local" {
    type master;
    file "/etc/bind/zonas/db.dawjgs.local";
};

// Zona inversa
zone "XX.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/zonas/db.XX.168.192.in-addr.arpa";
};
```

### 2.3. Configuración de zonas

#### 2.3.1. Zona directa

Crear el archivo `/etc/bind/zonas/db.dawjgs.local` con registros A y Aliases:

```bash
# Actualizar la dirección principal inicial
@       IN      SOA     ns1.dawjgs.local. root.dawjgs.local. (...

# Añadir las zonas
@             IN      NS      ns1
@             IN      A       192.168.XX.50
ns1           IN      A       192.168.XX.50
dns           IN      A       192.168.XX.50
gitlab        IN      A       192.168.XX.100
web           IN      A       192.168.XX.150
uftp          IN      A       192.168.XX.200
cliente       IN      A       192.168.XX.XXX # IP del cliente según PCNumber
jgscli        IN      A       192.168.XX.XXX ; # IP obtenido con `ip a` en enp0s3
servidor      IN      CNAME   dns
server        IN      CNAME   dns
git           IN      CNAME   gitlab
file          IN      CNAME   uftp
user          IN      CNAME   cliente
```

#### 2.3.2. Zona indirecta (Inversa)

Crear el archivo `/etc/bind/zonas/db.XX.168.192.in-addr.arpa`:

```bash
# Actualizar direcciones y nombres inferiores
@       IN      SOA     ns1.dawjgs.local. root.dawjgs.local. (...

// Añadir las zonas
@             IN      NS      ns1.dawjgs.local.
50            IN      PTR     ns1.dawjgs.local.
50            IN      PTR     dns.dawjgs.local.
100           IN      PTR     gitlab.dawjgs.local.
150           IN      PTR     web.dawjgs.local.
200           IN      PTR     uftp.dawjgs.local.
```

## 3. Comprobación del servidor DNS con nuestra DMZ

Validar sintaxis de los archivos y reiniciar Bind9:

```bash
sudo named-checkzone dawjgs.local /etc/bind/zonas/db.dawjgs.local
sudo named-checkzone XX.168.192.in-addr.arpa /etc/bind/zonas/db.XX.168.192.in-addr.arpa
sudo systemctl restart bind9
```

Realizar pruebas de resolución solicitadas en el escenario:

```bash
# 2.0. Comprobación cliente -> servidor ns1
nslookup ns1.dawjgs.local 192.168.XX.50 # Comprobación del cliente alternativa si devuelve SERVFAIL

# 2.1. Comprobación cliente -> servidor ns1
nslookup ns1.dawjgs.local

# 2.2. Comprobación cliente -> servidor web
nslookup web.dawjgs.local

# 2.3. Comprobación servidor web -> gitlab (usando alias git)
nslookup git

# 2.4. Comprobación cliente -> servidor ftp (alias file)
nslookup file.dawjgs.local

# Comprobación zona inversa
nslookup 192.168.XX.50
```

Acceder por SSH comprobando nombre de dominio:

```bash
cat /etc/resolv.conf
ssh ems@server
```
