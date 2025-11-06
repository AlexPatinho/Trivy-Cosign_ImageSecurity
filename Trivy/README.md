# 🛡️ Trivy — Escáner de Vulnerabilidades

![escaneo_trivy](/images/trivy.jpg)

## ⚙️ Guía de Instalación de Trivy

---

### 1️⃣ Instalación (método oficial - local)

#### Prerrequisitos (ejemplo en Rocky/CentOS/Fedora)
```bash
sudo dnf -y install wget git tar curl
```

Instalar y habilitar Docker o Podman
```bash
sudo dnf install -y docker
sudo systemctl enable --now docker
trivy --version
```

💡 Si Trivy no aparece en el PATH:
```bash
echo 'export PATH=$PATH:/usr/local/bin' | sudo tee -a /etc/profile
source /etc/profile
```

Actualiza la base de datos antes del primer uso:
```bash
trivy --download-db-only
```

---

### 2️⃣ Escaneo básico de prueba
Escanear la imagen alpine:latest en busca de vulnerabilidades:
```bash
trivy image alpine:latest
```

**Explicación:**
- image: indica a Trivy que escanee una imagen de contenedor (Docker o Podman).
- alpine:latest: nombre y etiqueta de la imagen a analizar.

---

### 3️⃣ Instalar plantilla HTML para reportes
```bash
sudo mkdir -p /root/reports
sudo mkdir -p /usr/local/share/trivy
sudo curl -fsSL -o /usr/local/share/trivy/html.tpl \
  https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl
```

---

### 4️⃣ Generar reportes profesionales (JSON + HTML)
Ejemplo con tres imágenes: nginx, alpine y redis.
**NGINX**
```bash
trivy image nginx:latest \
  --scanners vuln --timeout 10m \
  --format template --template /usr/local/share/trivy/html.tpl -o /root/trivy-report-nginx.html
```
```bash
trivy image nginx:latest \
  --scanners vuln --timeout 10m \
  --format json -o /root/trivy-report-nginx.json
```

**ALPINE**
```bash
trivy image alpine:latest \
  --scanners vuln --timeout 10m \
  --format template --template /usr/local/share/trivy/html.tpl -o /root/trivy-report-alpine.html
```
```bash
trivy image alpine:latest \
  --scanners vuln --timeout 10m \
  --format json -o /root/trivy-report-alpine.json
```

**REDIS**
```bash
trivy image redis:latest \
  --scanners vuln --timeout 10m \
  --format template --template /usr/local/share/trivy/html.tpl -o /root/trivy-report-redis.html
```
```bash
trivy image redis:latest \
  --scanners vuln --timeout 10m \
  --format json -o /root/trivy-report-redis.json
```

### 4️⃣ Generar reportes profesionales (JSON + HTML)
Guarda el siguiente script en **/usr/local/bin/scan-image.sh**
```bash
sudo tee /usr/local/bin/scan-image.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

# Uso: scan-image.sh <imagen[:tag]> [directorio_salida] [opciones_trivy...]
# Ejemplo: scan-image.sh nginx:latest /root/reports --skip-db-update
img="${1:-}"
outdir="${2:-/root/reports}"
shift || true
shift || true
extra_args=("$@")

if [[ -z "${img}" ]]; then
  echo "Uso: $(basename "$0") <imagen[:tag]> [directorio_salida] [opciones_trivy...]"
  exit 1
fi

tpl="/usr/local/share/trivy/html.tpl"
if [[ ! -f "$tpl" ]]; then
  echo "Template HTML no encontrado. Descargando..."
  sudo mkdir -p /usr/local/share/trivy
  sudo curl -fsSL -o "$tpl" \
    https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl
fi

mkdir -p "$outdir"
ts="$(date +%Y%m%d-%H%M%S)"
base="$(echo "$img" | tr '/:' '__')"
html="$outdir/${base}_${ts}.html"
json="$outdir/${base}_${ts}.json"

trivy image "$img" --scanners vuln --timeout 10m \
  --format template --template "$tpl" -o "$html" "${extra_args[@]}"

trivy image "$img" --scanners vuln --timeout 10m \
  --format json -o "$json" "${extra_args[@]}"

echo "Listo:"
echo "  HTML: $html"
echo "  JSON: $json"
EOF
```
```
sudo chmod +x /usr/local/bin/scan-image.sh
```



