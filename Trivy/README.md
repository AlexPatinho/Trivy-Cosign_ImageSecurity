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
---

### 5️⃣ Script para escanear imágenes manualmente
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

#### Ejemplos
**Ejemplo 1: Escaneo simple**
```bash
sudo /usr/local/bin/scan-image.sh redis:latest
```
![escaneo_simple](Script_escaneo_manual/images/ejemplo1.png)

**Ejemplo 2: Mismo escaneo, pero guardando en otro directorio**
```bash
sudo /usr/local/bin/scan-image.sh nginx:latest /root/trivy-reports
```
![escaneo_en_diferente_directorio](Script_escaneo_manual/images/ejemplo2.png)

**Ejemplo 3: Usando la DB ya descargada (más rápido en repeticiones)**
```bash
sudo /usr/local/bin/scan-image.sh alpine:latest /root/reports --skip-db-update
```
![escaneo_usando_db](Script_escaneo_manual/images/ejemplo3.png)

**Verificaciónd e salidas**
```bash
ls -lh /root/reports/*.html /root/reports/*.json 2>/dev/null || true
ls -lh /root/trivy-reports/*.html /root/trivy-reports/*.json 2>/dev/null || true
```
![salidas](Script_escaneo_manual/images/verificacion_salidas.png)

---

### Ejemplo: Escaneo de Redis
Ejecutando un escaneo con filtros de severidad:
```bash
trivy image redis:latest --scanners vuln --timeout 5m --severity HIGH,CRITICAL
```

Se encontraron 5 vulnerabilidades conocidas en los paquetes del sistema base.
![salidas](Script_escaneo_manual/images/escaneo.png)

---

## TABLA DE NIVELES DE SEVERIDAD

| Nivel | Color en el reporte | Descripción técnica | Recomendación práctica |
| :---: | :---: | :--- | :--- |
| **🟥 CRITICAL** | Rojo | Vulnerabilidades explotables de forma inmediata. Permiten ejecución remota de código, escalamiento de privilegios o fuga crítica de datos. | 🚨 **Debe corregirse de inmediato.** Actualiza la imagen o aplica parches antes de desplegar. |
| **🟧 HIGH** | Naranja | Fallos graves que pueden comprometer la seguridad, aunque requieran condiciones específicas o usuario local. | ⚠️ **Prioridad alta:** Mitigar o reemplazar la imagen tan pronto como sea posible. |
| **🟨 MEDIUM** | Amarillo | Problemas moderados: pueden revelar información o afectar estabilidad si se cumplen ciertas condiciones. | ⚙️ Planifica parcheo o revisión en el siguiente ciclo de mantenimiento. |
| **🟦 LOW** | Azul / gris | Riesgo bajo o difícil de explotar. Generalmente no afecta a la seguridad directamente. | 🕐 Puede posponerse, pero conviene monitorizar. |
| **⚫ UNKNOWN** | Gris claro | Trivy detectó un CVE, pero no hay datos suficientes o el proveedor no asignó severidad. | 🔍 Revisa manualmente el CVE o consulta la base NVD/CVSS. |

### Mapeo de Severidad (CVSS a Trivy)

Trivy utiliza las bases de datos **CVE** y **CVSS** (Common Vulnerability Scoring System). Cada CVE tiene una puntuación del 0 al 10, y Trivy lo mapea de la siguiente manera:

| CVSS Base Score | Severidad Trivy |
| :---: | :---: |
| 9.0 – 10.0 | **CRITICAL** |
| 7.0 – 8.9 | **HIGH** |
| 4.0 – 6.9 | **MEDIUM** |
| 0.1 – 3.9 | **LOW** |
| N/A | **UNKNOWN** |

### Campos principales del reporte

* **Library**: paquete afectado.
* **CVE**: identificador público.
* **Status**: estado del paquete (`affected`, `will_not_fix`).
* **Fixed Version**: versión corregida.
* **Title**: resumen del impacto o ataque posible.

---

## 🧠 Buenas prácticas de mitigación

* **🔄 Actualizar** las imágenes base con frecuencia.
* **🧱 Reconstruir** y reescanear antes de desplegar.
* **🔏 Implementar** políticas de firma (**Cosign** + **Connaisseur**).
* **🧪 Automatizar** el escaneo en pipelines **CI/CD**.

