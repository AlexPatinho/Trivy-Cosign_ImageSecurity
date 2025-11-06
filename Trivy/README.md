# 🛡️ Trivy — Escáner de Vulnerabilidades

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



