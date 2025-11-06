# 🧾 Cosign
Cosign es una herramienta de **Sigstore** que simplifica la firma y verificación de contenedores y otros artefactos de software. Es esencial para la implementación de políticas de seguridad en pipelines CI/CD y entornos de Kubernetes.

---

## ⚙️ Guía de Instalación de Cosign

Antes de instalar, asegúrate de tener privilegios de `root` y las siguientes dependencias:
```bash
sudo dnf install -y curl wget tar podman containerd jq
```

---

### 🧩 Método 1: Instalación desde repositorio (recomendada)

Instala Cosign usando el gestor de paquetes DNF:

```bash
sudo dnf install -y cosign
```

Verificar versión instalada
```bash
cosign version
```

Mostrar información binaria (ruta y hash)
```bash
which cosign
sha256sum $(which cosign)
```

**✅ Si obtienes un número de versión (por ejemplo cosign: 2.2.1), la instalación fue exitosa.
**

### 🧩 Método 2: Instalación binaria desde GitHub
Instala la versión más reciente directamente desde los releases oficiales de Sigstore:
```bash
sudo curl -sSL -o /usr/local/bin/cosind \
https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64

sudo chmod +x /usr/local/bin/cosind
sudo install -m 0755 /usr/local/bin/cosind /usr/local/bin/cosign
```
