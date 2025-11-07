# 🛡️ Troubleshooting y Solución de Problemas – Connaisseur

Este documento recopila los problemas más comunes encontrados durante la implementación y operación de Connaisseur en un clúster de Kubernetes, junto con sus causas, soluciones y recomendaciones preventivas.

---

## 3.1. Error: “failed calling webhook ... x509: certificate signed by unknown authority”

### 📋 Síntoma
Kubernetes no puede validar el webhook de Connaisseur.

### 🔍 Causa Probable
* El `caBundle` en la configuración del `MutatingWebhookConfiguration` no coincide con el certificado TLS usado por el servicio.
* El Secret `connaisseur-tls` fue regenerado sin actualizar el webhook.

### 🛠️ Solución
1.  **Verificar el contenido de la CA actual:**
    ```bash
    kubectl -n connaisseur get secret connaisseur-tls -o jsonpath='{.data.ca.crt}' | base64 -d | openssl x509 -noout -subject
    ```
2.  **Reinyectar el nuevo caBundle en el webhook:**
    ```bash
    CAB=$(kubectl -n connaisseur get secret connaisseur-tls -o jsonpath='{.data.ca.crt}')
    kubectl patch mutatingwebhookconfiguration connaisseur-webhook \
    --type='json' -p='[{"op":"replace","path":"/webhooks/0/clientConfig/caBundle","value":"'"$CAB"'"}]'
    ```
3.  **Esperar a que el webhook reinicie:**
    ```bash
    kubectl rollout status deploy/connaisseur -n connaisseur
    ```

### ✨ Prevención
* Mantener respaldos de la CA (`ca.crt`).
* Actualizar el webhook cada vez que se regeneren certificados TLS.

---

## 3.2. Error: “no matching signatures” o “invalid trust root”

### 📋 Síntoma
El webhook rechaza imágenes que deberían estar correctamente firmadas.

### 🔍 Causa Probable
* La configuración `trustRoot` en `values.yaml` (o `ConfigMap`) no contiene la clave pública correcta.
* El *digest* o *tag* de la imagen no coincide con la firma registrada.

### 🛠️ Solución
1.  **Verificar el valor actual de trustRoot:**
    ```bash
    kubectl -n connaisseur get configmap connaisseur-app-config -o yaml | grep -A3 trustRoot
    ```
2.  **Actualizar la clave pública (si es necesario) y reinstalar/actualizar:**
    ```bash
    helm upgrade connaisseur connaisseur/connaisseur -n connaisseur -f /root/connaisseur/values.yaml
    ```

### ✨ Prevención
* Usar nombres consistentes en las imágenes firmadas.
* Mantener una única clave pública (`cosign.pub`) por entorno para la verificación.

---

## 3.3. Error: “connection refused” o “context deadline exceeded”

### 📋 Síntoma
Los *pods* del clúster no pueden comunicarse con el webhook.

### 🔍 Causa Probable
* El servicio `connaisseur-svc` no tiene *endpoints* válidos (los *pods* no están *Ready*).
* Puerto incorrecto o el *sidecar* `socat` no redirige correctamente al *backend* de Connaisseur.

### 🛠️ Solución
1.  **Verificar *endpoints* activos:**
    ```bash
    kubectl -n connaisseur get endpoints connaisseur-svc -o wide
    ```
2.  **Reiniciar el *deployment* si hay inconsistencias:**
    ```bash
    kubectl -n connaisseur rollout restart deploy connaisseur
    ```

### ✨ Prevención
* Evitar modificar manualmente los puertos del servicio.
* Realizar *snapshots* del entorno antes de aplicar cambios a la infraestructura de red o servicios.

---

## 3.4. Error: “CrashLoopBackOff” en pods de Connaisseur

### 📋 Síntoma
El *pod* entra en un ciclo de reinicios constantes.

### 🔍 Causa Probable
* El contenedor principal no encuentra el binario `uvicorn` o los *scripts* de inicio.
* Los comandos `command` o `args` fueron sobrescritos incorrectamente en el *deployment*.
* Certificados faltantes o Secret eliminado (`connaisseur-tls`).

### 🛠️ Solución
1.  **Eliminar configuraciones manuales de `command/args`:**
    ```bash
    kubectl -n connaisseur patch deploy connaisseur --type=json -p='[
    {"op":"remove","path":"/spec/template/spec/containers/0/command"},
    {"op":"remove","path":"/spec/template/spec/containers/0/args"}
    ]'
    ```
2.  **Reinstalar/Actualizar desde Helm para restaurar la configuración base:**
    ```bash
    helm upgrade --install connaisseur connaisseur/connaisseur -n connaisseur -f /root/connaisseur/values.yaml
    ```

### ✨ Prevención
* No modificar directamente los `command` o `args` del *pod* principal. Si se requiere usar `uvicorn` o similares, hacerlo en una capa de pruebas separada.

---

## 3.5. Error: “unable to verify the first certificate” (en test TLS)

### 📋 Síntoma
La verificación TLS falla al usar `openssl s_client` para probar la conexión al *webhook*.

### 🔍 Causa Probable
* El *pod* de prueba no tiene acceso a la CA (`/tls/ca.crt`).
* El Secret `connaisseur-tls` no fue montado correctamente en el *pod* de prueba.
* El archivo `ca.crt` dentro del Secret está vacío o corrupto.

### 🛠️ Solución
**Ejecutar una prueba temporal montando el Secret (`connaisseur-tls`) correctamente:**

```bash
kubectl -n connaisseur run tlsprobe --image=alpine --restart=Never --overrides='{
"apiVersion": "v1",
"spec": {
"containers": [{
"name": "test",
"image": "alpine",
"command": ["sh","-lc"],
"args": [
"apk add --no-cache openssl >/dev/null 2>&1;
echo | openssl s_client -connect connaisseur-svc.connaisseur.svc:443 \
-CAfile /tls/ca.crt 2>/dev/null | egrep -i \"subject=|issuer=|Verify return code\""
],
"volumeMounts": [{"name":"tls","mountPath":"/tls","readOnly":true}]
}],
"volumes": [{"name":"tls","secret":{"secretName":"connaisseur-tls"}}]
}
}'
```

Verificar que el resultado contenga:
`Verify return code: 0 (ok)`

### ✨ Prevención
* Mantener sincronizados los certificados `CA (ca.crt)` entre el `Secret`, el `webhook` y el servicio.

---

## Comentario general:
La mayoría de los errores en Connaisseur provienen de inconsistencias entre el certificado TLS, el caBundle y el registro de imágenes firmado. Una vez que estos tres elementos están correctamente alineados, Connaisseur funciona de manera estable y confiable, garantizando que solo se desplieguen imágenes firmadas y verificadas.

---

## Buenas prácticas generales:
### 🛡️ Estabilidad y Mantenimiento

* **Realizar snapshots** antes de aplicar cualquier cambio en la configuración de TLS o del webhook.
* **Mantener backups** seguros de las claves Cosign y de la Autoridad Certificadora (CA) TLS.
* **Usar digest SHA256** en todas las políticas de Connaisseur para asegurar una verificación de imagen rigurosa.
* **Revisar los logs de los pods** inmediatamente después de cualquier modificación en la configuración de Cosign o Connaisseur para detectar errores tempranamente.

### 🚀 Despliegue y Desarrollo

* **Desplegar Connaisseur en un namespace dedicado** (por ejemplo, `connaisseur`) para aislarlo de las cargas de trabajo del clúster.
* **Mantener un directorio de pruebas** para las herramientas de seguridad (`Trivy`, `Cosign` y `Connaisseur`) antes de aplicar cambios en producción.
* **Utilizar la versión 2.2.x de Cosign**, ya que se ha observado que versiones posteriores pueden presentar fallas en despliegues que requieren verificación de firmas.
