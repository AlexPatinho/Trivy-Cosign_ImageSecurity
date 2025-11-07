# 🛡️ Guía de Instalación y Configuración de Connaisseur (Validación de Firmas de Imágenes)

Esta guía detalla la instalación del *Admission Webhook* **Connaisseur** en Kubernetes para forzar la validación de firmas digitales de imágenes (usando **Cosign**) provenientes de un registro privado e inseguro (`192.168.56.114:5000`).

---

## 🧭 Instalación de Helm y Preparación de Repositorios

### 1️⃣ Crear Namespace y preparar Helm

Crear el namespace
```bash
kubectl create ns connaisseur
```

Instalar Helm desde el repositorio oficial (si aún no está instalado)
```bash
curl [https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3](https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3) | bash
```
Verificar instalación
```bash
helm version
```

Agregar el repositorio oficial de Connaisseur
```bash
helm repo add connaisseur [https://sse-secure-systems.github.io/connaisseur/chart](https://sse-secure-systems.github.io/connaisseur/chart)
```

Actualizar repositorios
```bash
helm repo update
```

---

### 2️⃣ Crear archivo values.yaml (configuración mínima)
Este archivo define las políticas de validación, las claves públicas de firma y la excepción para el registro no seguro.

Crear directorio de trabajo
```bash
mkdir -p /root/connaisseur
cd /root/connaisseur
```

Archivo values.yaml
```bash
cat <<'EOF' > /root/connaisseur/values.yaml
#Configuración mínima para Connaisseur + Cosign local
insecureRegistries:
  - "192.168.56.114:5000"

trustRoots:
  - name: cosign-pubkey
    key: |
      -----BEGIN PUBLIC KEY-----
      (AGREGAR AQUÍ TU CLAVE PÚBLICA DE /root/cosign)
      -----END PUBLIC KEY-----

validators:
  - name: cosign-validator
    type: cosign
    trustRoots:
      - cosign-pubkey

policy:
  - pattern: "192.168.56.114:5000/*:*"
    validator: cosign-validator
EOF
```

#### Explicación de parámetros clave
- **insecureRegistries:** Permite usar registros HTTP sin TLS (por ejemplo: :5000).
- **trustRoots:** Contiene las claves públicas confiables, en este caso la de Cosign.
- **validators:** Define el tipo de validador (Cosign).
- **policy:** Determina qué imágenes serán verificadas.

### 3️⃣ Instalar Connaisseur con Helm
Instalar Connaisseur
```bash
helm install connaisseur connaisseur/connaisseur -n connaisseur -f values.yaml
```

Verificar estado de los pods
```bash
kubectl -n connaisseur get pods
```

---

## 🧩 Configuración del Webhook
El Webhook de Connaisseur actúa como un filtro del API Server de Kubernetes.
Antes de crear, actualizar o eliminar un recurso, el API Server solicita permiso a Connaisseur.

1. Excluir el namespace Connaisseur del webhook:
```bash
kubectl label namespace connaisseur \
  securesystemsengineering.connaisseur/webhook=ignore --overwrite
```

2. Verificar la configuración del webhook y los servicios:
```bash
kubectl get mutatingwebhookconfigurations | grep -i connaisseur
kubectl describe mutatingwebhookconfiguration connaisseur-webhook

kubectl -n connaisseur get svc -o wide
kubectl -n connaisseur get endpoints -o wide
```

3. Parchear el Service si no tiene endpoints válidos:
```bash
kubectl -n connaisseur patch svc connaisseur-svc --type='json' \
  -p='[{"op":"replace","path":"/spec/selector","value":{"app.kubernetes.io/instance":"connaisseur","app.kubernetes.io/name":"connaisseur"}}]'
```

4. Actualizar instalación (opcional):
```bash
helm upgrade --install connaisseur connaisseur/connaisseur \
  -n connaisseur -f /root/connaisseur/values.yaml \
  --set webhook.failurePolicy=Fail
```

5. Comprobación final:
```bash
kubectl -n connaisseur logs deploy/connaisseur --tail=50
```

### ESTRUCTURA SUGERIDA DE ARCHIVOS
``
- /root/connaisseur → values.yaml (configuración principal del Helm Chart)
- /root/cosign → cosign.key, cosign.pub (claves de firma Cosign)
- /root/connaisseur/tests → fail-unsigned.yaml, ok-signed.yaml (manifiestos de prueba)
``

