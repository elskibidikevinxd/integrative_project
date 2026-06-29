# Parte 1 — Distro personalizada con Cubic

## ISO Download
[Descargar ISO aquí] https://drive.google.com/file/d/11Hcagi0Mt5JxRgE-h2XYtCIOiZsWt02b/view?ts=6a420259

## Base utilizada
- **ISO base:** Linux Mint 22.3 Cinnamon 64-bit (Zena)
- **Herramienta:** Cubic (Custom Ubuntu ISO Creator)

---

## Pasos para reproducir

### 1. Arreglar repositorio cdrom roto
```bash
grep -r "cdrom" /etc/apt/
sudo rm /etc/apt/sources.list.d/cdrom.sources
sudo apt update
```

### 2. Instalar Cubic
```bash
sudo apt-add-repository ppa:cubic-wizard/release
sudo apt update
sudo apt install cubic
```

### 3. Crear carpeta del proyecto
```bash
mkdir -p ~/cubic-project/mi-distro
```

### 4. Abrir Cubic
```bash
cubic
```
- Project folder → `~/cubic-project/mi-distro`
- Original ISO → `linuxmint-22.3-cinnamon-64bit.iso`
- Clic en Next → Cubic abre terminal chroot

---

## Modificaciones aplicadas

### Modificación 1 — Reemplazar Transmission por qBittorrent
**Justificación:** qBittorrent es software libre (GPL), más completo y sin publicidad.

```bash
apt remove --purge transmission* -y
apt install qbittorrent -y
```

### Modificación 2 — Instalar Neovim + configurar /etc/skel
**Justificación:** Neovim es un editor potente de software libre. Configurarlo en `/etc/skel` hace que todos los usuarios nuevos hereden la configuración automáticamente.

```bash
apt install neovim -y
mkdir -p /etc/skel/.config/nvim
echo "set number" > /etc/skel/.config/nvim/init.vim
```

### Modificación 3 — Tema oscuro por defecto (gschema)
**Justificación:** Establecer el tema Mint-Y-Dark como predeterminado del sistema para todos los usuarios nuevos, sin configuración manual.

```bash
mkdir -p /usr/share/glib-2.0/schemas/
cat > /usr/share/glib-2.0/schemas/99_custom.gschema.override << 'EOF'
[org.cinnamon.desktop.interface]
gtk-theme='Mint-Y-Dark'
icon-theme='Mint-Y'
EOF
glib-compile-schemas /usr/share/glib-2.0/schemas/
```

---

## Generación del ISO
- Compresión: **XZ**
- Salida: `~/cubic-project/mi-distro/`

### Checksum
```bash
sha256sum ~/cubic-project/mi-distro/*.iso > ~/cubic-project/checksum.sha256
```

---

## Prueba en QEMU
```bash
sudo apt install qemu-system-x86 -y
qemu-system-x86_64 -cdrom ~/cubic-project/mi-distro/*.iso -m 2048 -boot d
```

---

## Comandos que fallaron y por qué

| Comando | Error | Solución |
|---|---|---|
| `ppa:librewolf-dev/librewolf` | No hay PPA para Mint Zena | Se usó qBittorrent como alternativa |
| `deb.librewolf.net` | 404 — no soporta Zena | Descartado |
| `sed -i '/cdrom/d' sources.list` | El cdrom estaba en sources.list.d | Se eliminó `cdrom.sources` directamente |
