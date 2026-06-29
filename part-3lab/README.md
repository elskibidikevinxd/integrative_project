# Parte 3 — Black Hat Bash Lab

## 3.A — Lab up and running

### Requisitos
- Docker + Docker Compose
- WSL2 con Ubuntu
- make

### Pasos para reproducir

sudo apt install make -y
git clone https://github.com/dolevf/Black-Hat-Bash.git
cd Black-Hat-Bash/lab
sudo make deploy
sudo make test
docker ps --format "{{.Names}}"
docker network inspect public | grep Subnet
docker network inspect corporate | grep Subnet

### Verificación
- make test → Lab is up.
- 8 contenedores corriendo
- Red pública: 172.16.10.0/24
- Red corporativa: 10.1.0.0/24

> **Nota:** En WSL2 con Docker Desktop, el comando `ip addr | grep "br_"` no muestra las interfaces `br_public` y `br_corporate` porque Docker Desktop las gestiona internamente en Windows y no las expone al subsistema Linux. Las redes fueron verificadas correctamente mediante `docker network inspect`:
> - `docker network inspect public` → Subnet: 172.16.10.0/24
> - `docker network inspect corporate` → Subnet: 10.1.0.0/24
> Ambas redes funcionan correctamente como lo confirma `make test = Lab is up` y los 8 contenedores corriendo.

### Arquitectura del lab

| Máquina | IP Pública | IP Corporativa |
|---|---|---|
| p-web-01 | 172.16.10.10 | — |
| p-web-02 | 172.16.10.12 | 10.1.0.11 |
| p-ftp-01 | 172.16.10.11 | — |
| p-jumpbox-01 | 172.16.10.13 | 10.1.0.12 |
| c-backup-01 | — | 10.1.0.13 |
| c-redis-01 | — | 10.1.0.14 |
| c-db-01 | — | 10.1.0.15 |
| c-db-02 | — | 10.1.0.16 |

### Diagrama de redes

Internet
    |
[ Red Pública: 172.16.10.0/24 ]
    |--- p-web-01     (172.16.10.10)
    |--- p-ftp-01     (172.16.10.11)
    |--- p-web-02     (172.16.10.12)
    |--- p-jumpbox-01 (172.16.10.13) ---+
                                        |
                        [ Red Corporativa: 10.1.0.0/24 ]
                            |--- p-web-02     (10.1.0.11)
                            |--- p-jumpbox-01 (10.1.0.12)
                            |--- c-backup-01  (10.1.0.13)
                            |--- c-redis-01   (10.1.0.14)
                            |--- c-db-01      (10.1.0.15)
                            |--- c-db-02      (10.1.0.16)

### Acceso demostrado

docker exec -it p-web-01 bash
root@p-web-01:/app#

---

## 3.B — Técnica de hacking (Nivel Advanced)

### Técnica 1 — Port scan con nmap

Herramienta: nmap
Objetivo: p-web-01 (172.16.10.10)
Comando:

docker exec -it p-jumpbox-01 bash
nmap -sV -sC 172.16.10.10

Resultado:
- Puerto 8081/tcp abierto
- Servicio: Werkzeug 3.0.1 / Python 3.12.3
- Aplicación web Flask corriendo

Interpretación: El servidor expone únicamente el puerto 8081 con una aplicación web Python/Flask. Werkzeug es un framework de desarrollo, su presencia indica posibles malas prácticas de configuración. Los 999 puertos restantes están cerrados pero el servicio web puede contener vulnerabilidades a nivel de aplicación.

---

### Técnica 2 — Login FTP anónimo

Herramienta: ftp client
Objetivo: p-ftp-01 (172.16.10.11)
Comando:

ftp 172.16.10.11
usuario: anonymous
password: (vacío)

Resultado:
- Login exitoso sin contraseña
- Directorio backup/ con carpetas: acme-hyper-branding, acme-impact-alliance
- Archivo app.py (código fuente) expuesto públicamente

Interpretación: El servidor vsFTPd 3.0.5 tiene habilitado el acceso anónimo, cualquier persona en la red puede conectarse sin credenciales. Se encontraron backups con código fuente, lo que representa una fuga de información crítica. Un atacante podría analizar el código para encontrar vulnerabilidades adicionales.

---

### Técnica 3 — Nuclei vulnerability scanning (Advanced)

Herramienta: Nuclei v3.9.0
Objetivo: http://172.16.10.10:8081
Comando:

nuclei -u http://172.16.10.10:8081 -tags exposure,misconfig,disclosure

Resultado — 11 vulnerabilidades encontradas:

| Vulnerabilidad | Severidad | Descripción |
|---|---|---|
| strict-transport-security | info | Sin HTTPS forzado |
| x-content-type-options | info | Vulnerable a MIME sniffing |
| cross-origin-embedder-policy | info | Sin aislamiento de origen |
| cross-origin-opener-policy | info | Vulnerable a ataques cross-window |
| cross-origin-resource-policy | info | Recursos accesibles desde otros orígenes |
| content-security-policy | info | Sin protección contra XSS |
| permissions-policy | info | Sin control de permisos |
| x-frame-options | info | Vulnerable a clickjacking |
| x-permitted-cross-domain-policies | info | Sin restricción de dominios |
| referrer-policy | info | Filtra URLs internas |
| missing-sri | info | Recursos externos sin verificación de integridad |

Interpretación: Nuclei ejecutó 2058 templates automáticamente y detectó 11 misconfigurations. La ausencia de headers de seguridad HTTP expone a los usuarios a ataques como XSS, clickjacking y MIME sniffing. El missing-sri indica que recursos externos se cargan sin verificar su integridad, permitiendo ataques de supply chain si esos CDNs son comprometidos.