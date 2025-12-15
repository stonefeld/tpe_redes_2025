# Arquitectura de Red VPN Site-to-Site con OpenVPN y Kubernetes

Este proyecto implementa una arquitectura de red compleja que combina dos VPCs de AWS conectadas mediante VPN site-to-site usando OpenVPN, junto con un cluster de Kubernetes desplegado en una instancia privada de EC2.

## Descripción General

La arquitectura está compuesta por:

1. **LAN A (Servidor Principal)**:
   - VPC con CIDR `10.0.0.0/16`
   - OpenVPN Server que actúa como gateway VPN y proxy reverso
   - Cluster de Kubernetes (kind) en una instancia EC2 privada
   - Aplicación "The Store" desplegada en el cluster
   - Servidor web Nginx para descarga de certificados y proxy al cluster

2. **LAN B (Cliente Site-to-Site)**:
   - VPC con CIDR `10.2.0.0/16`
   - OpenVPN Client Gateway que establece la conexión site-to-site con LAN A

La conexión VPN permite:
- **Client-to-Site**: Usuarios pueden conectarse desde sus máquinas locales al cluster de Kubernetes y a la aplicación
- **Site-to-Site**: Las dos VPCs pueden comunicarse entre sí a través del túnel VPN

## Prerrequisitos

Antes de comenzar, necesitas:

1. **AWS CLI** configurado con credenciales válidas
2. **Terraform** instalado
3. **Clave SSH** para acceso a las instancias EC2

### Generar Clave SSH

Genera una clave SSH RSA de 4096 bits:

```bash
ssh-keygen -t rsa -b 4096 -f id_rsa
```

Esto creará dos archivos:
- `id_rsa`: Clave privada (mantenerla segura)
- `id_rsa.pub`: Clave pública (se usará en Terraform)

> **Importante**: Asegúrate de que estos archivos estén en el directorio raíz del proyecto, ya que los archivos `.tfvars` hacen referencia a ellos.

## Despliegue

El despliegue se realiza en dos pasos, uno para cada VPC:

### Paso 1: Desplegar LAN A (Servidor Principal con Cluster)

Esta es la VPC principal que contiene el cluster de Kubernetes y la aplicación "The Store".

```bash
terraform init
terraform plan -var-file=lan_a.tfvars
terraform apply -var-file=lan_a.tfvars
```

El archivo `lan_a.tfvars` configura:
- `lan_role = "server"`: Esta VPC actúa como servidor OpenVPN
- `vpc_cidr = "10.0.0.0/16"`: CIDR de esta VPC
- `remote_cidr = "10.2.0.0/16"`: CIDR de la VPC remota (LAN B)
- `peer_gateway_common_name = "lan-b-gw"`: Nombre del certificado para el gateway de LAN B
- `enable_site_to_site = true`: Habilita el routing site-to-site

Este despliegue creará:
- VPC con subnets públicas y privadas
- OpenVPN Server en subnet pública
- Instancia EC2 privada con el cluster de Kubernetes
- Rutas de red para site-to-site y client-to-site

### Paso 2: Desplegar LAN B (Cliente Site-to-Site)

Esta es la VPC secundaria que se conecta a LAN A mediante VPN site-to-site.

```bash
terraform init  # Solo necesario si es en un directorio diferente
terraform plan -var-file=lan_b.tfvars
terraform apply -var-file=lan_b.tfvars
```

El archivo `lan_b.tfvars` configura:
- `lan_role = "client"`: Esta VPC actúa como cliente OpenVPN
- `vpc_cidr = "10.2.0.0/16"`: CIDR de esta VPC
- `remote_cidr = "10.0.0.0/16"`: CIDR de la VPC remota (LAN A)
- `peer_gateway_common_name = "lan-a-gw"`: Nombre del certificado para el gateway de LAN A
- `enable_site_to_site = true`: Habilita el routing site-to-site

> **Nota**: Si despliegas ambas VPCs en el mismo directorio de Terraform, necesitarás usar [workspaces](https://www.terraform.io/docs/language/state/workspaces.html) o directorios separados para evitar conflictos de estado.

## Scripts de User Data

Durante el despliegue, Terraform inyecta scripts de user data en las instancias EC2 que automatizan la configuración. Estos scripts se ejecutan automáticamente al iniciar las instancias.

### Scripts en LAN A (Servidor)

#### `openvpn-server.sh`

Este script configura el servidor OpenVPN en LAN A:

1. **Instalación de dependencias**:
   - OpenVPN
   - Easy-RSA (para gestión de certificados)
   - Nginx (para web server y proxy reverso)
   - Python3 e iptables

2. **Configuración de red**:
   - Habilita IP forwarding
   - Configura NAT para el pool de VPN (10.8.0.0/24)
   - Configura reglas de iptables para forwarding

3. **Configuración de OpenVPN Server**:
   - Crea la infraestructura de certificados (CA, servidor, DH)
   - Configura el servidor en el puerto 1194 UDP
   - Configura el túnel TUN con pool 10.8.0.0/24
   - Habilita site-to-site routing con CCD (Client Config Directory)

4. **Configuración de Nginx**:
   - **Ruta protegida `/download/<usuario>/<archivo>.ovpn`**:
     - Autenticación HTTP Basic
     - Cada usuario solo puede descargar sus propios certificados
     - Headers anti-cache para seguridad
   - **Proxy reverso al cluster**:
     - Todas las demás rutas se proxean a la IP privada del cluster
     - Soporte para WebSockets

5. **Herramientas de gestión**:
   - `/root/ovpn-export`: Script para exportar certificados de clientes a la web
   - `/root/client-setup-auto.sh`: Script para crear certificados de clientes client-to-site
   - `/root/vpn-management.sh`: Script para gestionar el servicio OpenVPN

6. **Creación automática del certificado del gateway**:
   - Crea el certificado para el gateway de LAN B (`lan-b-gw`)
   - Genera el archivo `.ovpn` con toda la configuración
   - Lo exporta automáticamente a la web para descarga

#### `setup-the-store.sh`

Este script configura la instancia EC2 privada que alojará el cluster de Kubernetes:

1. **Instalación de Docker**:
   - Instala Docker desde el repositorio oficial
   - Configura el usuario `ubuntu` en el grupo `docker`
   - Inicia el servicio Docker

2. **Instalación de kind**:
   - Descarga e instala kind (Kubernetes in Docker)
   - Soporta arquitecturas x86_64 y ARM64

3. **Instalación de kubectl**:
   - Descarga e instala kubectl
   - Verifica la firma SHA256

4. **Clonación del repositorio**:
   - Clona el repositorio `the-store` en `/home/ubuntu/the-store`

### Scripts en LAN B (Cliente)

#### `openvpn-client.sh`

Este script configura el gateway cliente OpenVPN en LAN B:

1. **Instalación de dependencias**:
   - OpenVPN
   - Python3 e iptables

2. **Configuración de red**:
   - Habilita IP forwarding (para actuar como gateway)
   - Configura reglas de iptables para forwarding del túnel VPN

3. **Script de instalación del cliente**:
   - Crea `/root/install-gateway-client.sh`:
     - Acepta un archivo `.ovpn` como parámetro
     - Añade automáticamente la ruta al CIDR remoto (LAN A) si no está presente
     - Copia la configuración a `/etc/openvpn/`
     - Inicia el servicio OpenVPN cliente

> **Nota**: Este script solo prepara el gateway. El certificado debe descargarse desde el servidor de LAN A y luego instalarse manualmente usando el script `install-gateway-client.sh`.

## Descarga de Certificados

Los certificados OpenVPN se descargan desde el servidor de LAN A mediante una página web protegida con autenticación HTTP Basic.

### Obtener la IP del Servidor

Después del despliegue, obtén la IP pública del servidor OpenVPN:

```bash
terraform output -var-file=lan_a.tfvars openvpn_public_ip
```

### Descargar Certificado del Gateway (Site-to-Site)

Para la conexión site-to-site, el certificado del gateway de LAN B ya está creado automáticamente. Para descargarlo:

1. **Obtener credenciales**:
   - SSH al servidor OpenVPN de LAN A
   - Ver el mensaje de bienvenida (MOTD) o ejecutar:
   ```bash
   ssh ubuntu@<IP_SERVIDOR>
   cat /etc/motd
   ```

2. **Descargar desde el navegador**:
   - Abre: `http://<IP_SERVIDOR>/download/lan-b-gw/lan-b-gw.ovpn`
   - Usa las credenciales mostradas en el MOTD:
     - Usuario: `lan-b-gw`
     - Password: (generado aleatoriamente, mostrado en el MOTD)

3. **Instalar en el gateway de LAN B**:
   ```bash
   # En el servidor de LAN B
   scp lan-b-gw.ovpn ubuntu@<IP_LAN_B>:/tmp/
   ssh ubuntu@<IP_LAN_B>
   sudo /root/install-gateway-client.sh /tmp/lan-b-gw.ovpn 10.0.0.0/16
   ```

### Crear y Descargar Certificado de Cliente (Client-to-Site)

Para crear un certificado de cliente que permita conectarse desde tu máquina local:

1. **SSH al servidor OpenVPN**:
   ```bash
   ssh ubuntu@<IP_SERVIDOR>
   ```

2. **Crear el certificado**:
   ```bash
   sudo /root/client-setup-auto.sh mi-usuario
   ```

   Esto creará el certificado y lo exportará automáticamente a la web, mostrándote:
   - Usuario
   - Password
   - URL de descarga

3. **Descargar el certificado**:
   - Abre la URL proporcionada en tu navegador
   - Ingresa las credenciales mostradas
   - Descarga el archivo `.ovpn`

## Conexión a la VPN

Una vez descargado el certificado, puedes conectarte a la VPN desde tu máquina local.

### Opción 1: OpenVPN desde la Línea de Comandos (Linux/macOS)

1. **Instalar OpenVPN**:
   ```bash
   # Ubuntu/Debian
   sudo apt-get install openvpn

   # macOS
   brew install openvpn
   ```

2. **Conectar**:
   ```bash
   sudo openvpn --config mi-usuario.ovpn
   ```

3. **Verificar la conexión**:
   ```bash
   # Deberías poder hacer ping a recursos en la VPC
   ping 10.0.2.X  # IP del cluster privado
   ```

### Opción 2: OpenVPN GUI (Windows)

1. **Descargar OpenVPN GUI**:
   - Descarga desde: https://openvpn.net/community-downloads/

2. **Importar el certificado**:
   - Copia el archivo `.ovpn` a `C:\Users\<Usuario>\OpenVPN\config\`

3. **Conectar**:
   - Abre OpenVPN GUI
   - Selecciona tu perfil
   - Haz clic en "Connect"

4. **Verificar**:
   - Una vez conectado, deberías ver un ícono de VPN activo
   - Puedes acceder a `http://<IP_SERVIDOR>/` en tu navegador

## Levantar el Cluster de Kubernetes

Para levantar el cluster de Kubernetes en la instancia EC2 privada:

1. **Conectarte a la instancia del cluster**:
   ```bash
   # Obtener la IP del servidor OpenVPN y del cluster
   OPENVPN_IP=$(terraform output -var-file=lan_a.tfvars -raw openvpn_public_ip)
   CLUSTER_IP=$(terraform output -var-file=lan_a.tfvars -raw cluster_private_ip)

   # SSH usando el OpenVPN como jump host
   ssh -J ubuntu@$OPENVPN_IP ubuntu@$CLUSTER_IP
   ```

2. **Navegar al directorio del proyecto**:
   ```bash
   cd tpe_redes_2025/the-store
   ```

3. **Crear el cluster**:
   ```bash
   ./local.sh create-cluster --skip-tests
   ```

   Este comando:
   - Crea un cluster de Kubernetes usando kind
   - Despliega la aplicación "The Store" en el cluster
   - Configura todos los servicios necesarios

## Acceso a la Aplicación

Una vez levantado el cluster y conectado a la VPN, puedes acceder a la aplicación "The Store":

1. **Abrir en el navegador**:
   ```
   http://<IP_SERVIDOR>/
   ```

## Comandos Útiles

### Verificar Estado del Servidor OpenVPN (LAN A)

```bash
ssh ubuntu@<IP_SERVIDOR>
sudo /root/vpn-management.sh status
sudo /root/vpn-management.sh clients  # Ver clientes conectados
sudo /root/vpn-management.sh logs     # Ver logs en tiempo real
```

### Verificar Estado del Cliente OpenVPN (LAN B)

```bash
ssh ubuntu@<IP_LAN_B>
sudo systemctl status openvpn-client@lan-b-gw
sudo journalctl -u openvpn-client@lan-b-gw -f
```

### Probar Conectividad Site-to-Site

```bash
# Desde LAN B
ping 10.0.2.X  # IP del cluster en LAN A

# Desde LAN A
ping 10.2.2.X  # IP de recursos en LAN B
```

### Acceder al Cluster de Kubernetes

```bash
# SSH al servidor privado usando el OpenVPN como jump host
ssh -J ubuntu@<IP_SERVIDOR> ubuntu@<IP_CLUSTER_PRIVADO>

# O si estás conectado a la VPN desde tu máquina local
ssh ubuntu@<IP_CLUSTER_PRIVADO>
```

## Usar kubectl de Manera Remota

Para ejecutar comandos de Kubernetes desde tu máquina local (sin necesidad de SSH al servidor):

1. **Exportar el kubeconfig desde el servidor del cluster**:
   ```bash
   # SSH al servidor del cluster
   ssh -J ubuntu@<IP_SERVIDOR> ubuntu@<IP_CLUSTER_PRIVADO>

   # Exportar el kubeconfig
   kind get kubeconfig --name the-store > kubeconfig.yaml
   ```

2. **Copiar el kubeconfig a tu máquina local**:
   ```bash
   # Desde tu máquina local
   scp -J ubuntu@<IP_SERVIDOR> ubuntu@<IP_CLUSTER_PRIVADO>:~/kubeconfig.yaml ./
   ```

3. **Modificar la IP en el kubeconfig**:
   ```bash
   # Obtener la IP privada del cluster
   CLUSTER_IP=$(terraform output -raw cluster_private_ip)

   # Reemplazar la IP en el kubeconfig
   sed -i "s|server:.*|server: https://$CLUSTER_IP:6443|" kubeconfig.yaml
   ```

4. **Usar kubectl con el kubeconfig**:
   ```bash
   # Ejecutar comandos de kubectl especificando el kubeconfig y el namespace
   KUBECONFIG=./kubeconfig.yaml kubectl -n the-store --insecure-skip-tls-verify=true get pods
   KUBECONFIG=./kubeconfig.yaml kubectl -n the-store --insecure-skip-tls-verify=true get services
   KUBECONFIG=./kubeconfig.yaml kubectl -n the-store --insecure-skip-tls-verify=true logs <pod-name>
   ```

   > **Nota**: El flag `--insecure-skip-tls-verify=true` es necesario porque kind usa certificados autofirmados.

## Integración de CI/CD Pipeline

Si se quisiera integrar un pipeline de CI/CD para automatizar el despliegue de actualizaciones al cluster de Kubernetes remoto a través de la conexión VPN, se debería realizar lo siguiente:

### Configuración Inicial

Para configurar el pipeline CI/CD, necesitas agregar los siguientes secretos en GitHub (Settings → Secrets and variables → Actions):

1. **`OPENVPN_CLIENT_CERT`**: Contenido completo del archivo `.ovpn` que descargaste del servidor OpenVPN
2. **`KUBECONFIG_CONTENT`**: Contenido completo del archivo `kubeconfig.yaml` del cluster
3. **`KUBERNETES_CLUSTER_IP`**: IP privada del cluster (ej: `10.0.2.153`)

> **Nota**: Asegúrate de que en el `kubeconfig.yaml` el campo `server:` apunte a la IP privada del cluster (ej: `http://10.0.2.153:6443`).

### Cómo Funciona el Pipeline

El pipeline se ejecutaría automáticamente cuando se hace push a la rama `main` con cambios en los servicios. El flujo sería el siguiente:

1. **Checkout del código**: Descarga el código del repositorio
2. **Instalación de OpenVPN**: Instala el cliente OpenVPN en el runner
3. **Conexión VPN**: Usa el certificado almacenado en los secretos para conectarse al servidor OpenVPN y establecer el túnel VPN
4. **Configuración de kubectl**: Instala kubectl y configura el kubeconfig para acceder al cluster remoto
5. **Construcción de imágenes**: Construye las imágenes Docker de todos los servicios (catalog, cart, checkout, orders, ui) con un tag basado en el commit SHA
6. **Actualización de manifiestos**: Actualiza los tags de las imágenes en el archivo `kubernetes.yaml` con el nuevo tag del commit
7. **Despliegue**: Aplica los manifiestos de Kubernetes al cluster remoto
8. **Verificación**: Espera a que todos los deployments se completen correctamente
9. **Limpieza**: Cierra la conexión VPN

### Ejemplo de Pipeline

El archivo `.github/workflows/deploy.yml` debería contener la siguiente configuración:

```yaml
name: Deploy to Remote Cluster

on:
  push:
    branches: [main]
    paths:
      - 'the-store/src/**'
      - 'the-store/dist/**'

env:
  NAMESPACE: the-store
  IMAGE_TAG: ${{ github.sha }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install OpenVPN
        run: |
          sudo apt-get update
          sudo apt-get install -y openvpn

      - name: Connect via OpenVPN
        run: |
          echo "${{ secrets.OPENVPN_CLIENT_CERT }}" > /tmp/client.ovpn
          chmod 600 /tmp/client.ovpn
          sudo openvpn --config /tmp/client.ovpn --daemon
          
          # Wait for VPN connection
          for i in {1..20}; do
            if ip link show tun0 &>/dev/null; then
              echo "VPN connected"
              break
            fi
            sleep 2
          done

      - name: Setup kubectl
        uses: azure/setup-kubectl@v3

      - name: Configure kubeconfig
        run: |
          mkdir -p ~/.kube
          echo "${{ secrets.KUBECONFIG_CONTENT }}" > ~/.kube/config

      - name: Build Docker images
        working-directory: ./the-store
        run: |
          for service in catalog cart checkout orders ui; do
            docker build -t the-store-$service:${{ env.IMAGE_TAG }} ./src/$service
          done

      - name: Update image tags in manifests
        working-directory: ./the-store
        run: |
          sed -i "s|image: \"the-store-\(.*\):.*\"|image: \"the-store-\1:${{ env.IMAGE_TAG }}\"|g" dist/kubernetes.yaml

      - name: Apply Kubernetes manifests
        working-directory: ./the-store
        run: |
          kubectl --insecure-skip-tls-verify=true create namespace ${{ env.NAMESPACE }} --dry-run=client -o yaml | kubectl --insecure-skip-tls-verify=true apply -f -
          kubectl --insecure-skip-tls-verify=true apply -f dist/kubernetes.yaml -n ${{ env.NAMESPACE }}

      - name: Wait for deployment
        run: |
          kubectl --insecure-skip-tls-verify=true rollout status deployment/catalog -n ${{ env.NAMESPACE }} --timeout=300s || true
          kubectl --insecure-skip-tls-verify=true rollout status deployment/cart -n ${{ env.NAMESPACE }} --timeout=300s || true
          kubectl --insecure-skip-tls-verify=true rollout status deployment/checkout -n ${{ env.NAMESPACE }} --timeout=300s || true
          kubectl --insecure-skip-tls-verify=true rollout status deployment/orders -n ${{ env.NAMESPACE }} --timeout=300s || true
          kubectl --insecure-skip-tls-verify=true rollout status deployment/ui -n ${{ env.NAMESPACE }} --timeout=300s || true

      - name: Cleanup
        if: always()
        run: sudo pkill openvpn || true
```

### Requisitos

Para que el pipeline funcione correctamente:

- El cluster de Kubernetes debe estar corriendo y accesible a través de la VPN
- Las imágenes Docker deben estar cargadas en el cluster Kind (usando `local.sh` o un registry de contenedores)
- El certificado OpenVPN debe ser válido y tener permisos para conectarse al servidor

## Limpieza

Para destruir los recursos:

```bash
# Destruir LAN B primero
terraform destroy -var-file=lan_b.tfvars

# Luego destruir LAN A
terraform destroy -var-file=lan_a.tfvars
```


