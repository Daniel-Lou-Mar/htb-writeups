# 🔐 HackTheBox Writeups

Repositorio con los writeups de las máquinas que voy resolviendo en [HackTheBox](https://www.hackthebox.com).

Cada writeup detalla el proceso completo: reconocimiento, enumeración, explotación y escalada de privilegios.

---

## 📊 Estadísticas

| Total resueltas | Fáciles | Medias | Difíciles |
|:-:|:-:|:-:|:-:|
| 2 | 2 | 0 | 0 |

---

## 🖥️ Máquinas

| Máquina | OS | Dificultad | Técnicas | Writeup |
|---------|-----|------------|----------|---------|
| [Cap](#) | Linux | 🟢 Fácil | IDOR, FTP credentials, Linux Capabilities | [📄 Ver](machines/Cap/writeup.md) |
| [Reactor](#) | Linux | 🟢 Fácil | CVE-2025-66478 (Next.js RCE), SQLite credentials, Node.js Inspector | [📄 Ver](machines/Reactor/writeup.md) |

---

## 🛠️ Técnicas utilizadas

- **IDOR** (Insecure Direct Object Reference)
- **Análisis de tráfico de red** (PCAP)
- **Credenciales en texto claro** (FTP)
- **Linux Capabilities** (`cap_setuid`)
- **CVE-2025-66478** (Next.js RCE — Prototype Pollution en RSC)
- **Node.js Inspector** (ejecución de código como root vía `--inspect`)
- **Cracking de hashes MD5**

---

## ⚙️ Herramientas

- `nmap` — escaneo de puertos y servicios
- `searchsploit` — búsqueda de exploits
- `tshark` / `wireshark` — análisis de tráfico
- `whatweb` / `wappalyzer` — fingerprinting web
- `getcap` — enumeración de capabilities
- `sqlite3` — análisis de bases de datos
- `john` — cracking de hashes
- `python3` — desarrollo de exploits personalizados

---

> ⚠️ Este repositorio es solo con fines educativos. Todas las máquinas pertenecen a HackTheBox y se resuelven en entornos controlados y legales.
