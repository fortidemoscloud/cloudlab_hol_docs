# Protección de entornos DevOps

## Introducción

Esta demo utiliza FortiCNAPP Code Security para implementar protección en la fase de desarrollo mediante un enfoque shift-left security.

Con el entorno desplegado siguiendo las instrucciones de [xs25_hub-spoke-gitops.md](xs25_hub-spoke-gitops.md), los siguientes pasos serán: 

1. Visibilidad de las cargas de trabajo y vulnerabilidades en el cluster.
2. Analisis de código mediante integración en el pipeline de automatización del despliegue.

## 1. FortiCNAPP Cloud Work Load Protection (CWP)

La forma en que FortiCNAPP puede ver las vulnerabilidades es la siguiente:
 
- Mediante agente no intrusivo que corre a nivel de usuario en: 
    - Máquinas Linux o Windows.
    - Contenedor con una apromación de sidecar.  
    - Entornos de Kubernetes como daemonset. 
- Sin agente, esta aproximación es sólo válida para los entornos de public cloud.
- Escaneando las imágenes de repositorios de contenedores: ECR, Dockerhub, locales … donde FortiCNAPP correlará las imagenes que están activas en el entorno y priorizarás las alertas sobre las misma. 
- Escaneando repositorios de código. 

Para la demo vamos a desplegar el agente colector de FortiCNAPP en el nuevo cluster que hemos desplegado, para ello nos ayudaremos de un script que simplifica la instalación usando **HELM**. 

>![NOTE]
> Tipos de instalación del agente, docu [aquí](https://docs.fortinet.com/document/forticnapp/25.4.0/administration-guide/903770/agent-based-workload-security)

```sh
LW_TOKEN=<agent-token>
K8S_CLUSTER=<cluster-kubernetes-name>

helm repo add lacework https://lacework.github.io/helm-charts
helm repo update
helm upgrade --install --namespace lacework --create-namespace \
    --set laceworkConfig.serverUrl=https://api.fra.lacework.net \
    --set laceworkConfig.accessToken=$LW_TOKEN \
    --set laceworkConfig.kubernetesCluster=$K8S_CLUSTER \
    --set laceworkConfig.env=Dev \
    lacework-agent lacework/lacework-agent
```

Los scripts que podemos usar:

1. k8s_get_config.sh - Obtener el contexto de acceso al cluster

```bash
# Script to retrieve the kubeconfig file from a remote server using an SSH private key stored in GCP Secret Manager
# Usage example:
./k8s_get_config.sh user-abcD-k8s 54.217.12.12
```

2. k8s_helm_forticnapp.sh - Lanzar el Helm sobre el cluster

```bash
# k8s_helm_forticnapp.sh - Script to deploy Lacework agent via Helm to Kubernetes cluster
# Arguments:
#  - config_name: Name of the kubeconfig file (default: 54.217.12.12_config)
#  - --insecure: Skip TLS verification (use this if having certificate issues)
# Usage example:
./k8s_helm_forticnapp.sh 54.217.12.12_config --insecure
```

## 2. FortiCNAPP Code Security - IaC, SAST y SCA

IaC Scan, SAST y SCA: Análisis de Seguridad en el Desarrollo

**IaC Scan** (Infrastructure as Code Scanning) analiza el código de infraestructura (como Terraform, CloudFormation, Dockerfiles) para detectar configuraciones inseguras o errores antes del despliegue, identificando problemas como puertos abiertos innecesarios, cifrado deshabilitado o permisos excesivos. 

**SAST** (Static Application Security Testing) examina el código fuente de las aplicaciones sin ejecutarlo, buscando vulnerabilidades como inyecciones SQL, fallos de autenticación o exposición de credenciales mediante el análisis estático del código. 

**SCA** (Software Composition Analysis) inspecciona las dependencias y librerías de terceros utilizadas en el proyecto, detectando componentes con vulnerabilidades conocidas (CVEs), licencias incompatibles o versiones desactualizadas, asegurando que los componentes externos no introduzcan riesgos de seguridad. 

Estas tres técnicas son complementarias y forman parte de una estrategia DevSecOps integral para identificar y mitigar riesgos de seguridad en diferentes capas del desarrollo de software.

El primer paso que vamos a dar es el de integrarnos en el ciclo de integración (CI) para lanzar los escaneos, activandolos con las tareas de commit de código como push o pull request.

En esta demo vamos a ver la integración con Github Actions, pero FortiCNAPP se integra con casi todas las herramientas de CI/CD. [FortiCNAPP CICD integrations](https://docs.fortinet.com/document/forticnapp/25.4.0/administration-guide/526554/integrate-with-a-ci-cd-pipeline)


Estos son los jobs a configurar en Gighub actions workflows:

### IaC scanning:

```yaml
jobs:
  forticnapp-iac:
    runs-on: ubuntu-24.04
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        
      - name: Create environment variables file
        run: |
          env | grep "GITHUB_\|LW_\|CI_" > env.list || true

          echo "LW_ACCOUNT=${{ secrets.LW_ACCOUNT_NAME }}" >> env.list
          echo "LW_API_KEY=${{ secrets.LW_API_KEY }}" >> env.list
          echo "LW_API_SECRET=${{ secrets.LW_API_SECRET }}" >> env.list

          # Fail job if critical findings exist (change to HIGH=1 if desired)
          echo "EXIT_FLAG=CRITICAL=5" >> env.list
          
      - name: check ENV file
        run: |
          echo "--- env.list created ---"
          cat env.list | grep -v "SECRET" || true

      - name: Run IaC scan
        run: |
          echo "Running Lacework/FortiCNAPP IaC scanner..."
          docker run --rm \
            --env-file env.list \
            -v "$(pwd):/app/src" \
            lacework/codesec:stable \
            lacework iac scan --directory=. --noninteractive

      - name: Clean up environment file
        if: always()
        run: rm -f env.list
```

### SCA y SAST scanning:

```yaml
env:
  LW_ACCOUNT_NAME: ${{ secrets.LW_ACCOUNT_NAME }}
  LW_API_KEY: ${{ secrets.LW_API_KEY }}
  LW_API_SECRET: ${{ secrets.LW_API_SECRET }}

jobs:
  forticnapp-sca-sast:
    runs-on: ubuntu-24.04

    name: Run analysis

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Analyze
        uses: lacework/code-security-action@v1
        with:
          target: push
```

Una vez se lancen estas tareas de forma automatizada, en la consola de FortiCNAPP apareran los resultados de los mismos:

* IaC pipelines:

![forticnapp_iac](./images/forticnapp_iac_01.png)

Los escaneos de repositorios que se hayan realizado mediante integración en:

 **`Settings`>`Code Security`** 
 
 apareceran en el tab de `Repositorios` de la sección del menú **`</> Code Security`**

Vamos a lanzar una actualización del código de la aplicación, para añadir intencionadamente librerias con vulnerabilidades críticas y fallos de implementación de código: 

**Código a añadir en routes.py**

```python
import subprocess
import hashlib
import sqlite3

#CWE-78: OS Command Injection
@main.route('/api/ping')
def ping_host():
    host = request.args.get('host', 'localhost')
    
    # Vulnerable: Command injection
    result = subprocess.check_output(f"ping -c 1 {host}", shell=True)
    return result.decode()

#A2: Broken Authentication
@main.route('/api/login', methods=['POST'])
def login():
    username = request.form['username']
    password = request.form['password']
    
    # Vulnerable: Weak hashing algorithm without salt
    hashed_password = hashlib.md5(password.encode()).hexdigest()
    
    conn = sqlite3.connect('database.db')
    cursor = conn.cursor()
    query = f"SELECT * FROM users WHERE username = '{username}' AND password = '{hashed_password}'"
    cursor.execute(query)
    user = cursor.fetchone()
    conn.close()
    
    if user:
        return "Login successful"
    return "Login failed"
```

**Líneas a añadir en requiriments.txt**

```python
SQLAlchemy==1.2.0
pyyaml==3.13
```

## Ejemplos de explotación:

```ssh
# Read sensitive files
curl "http://localhost:5000/api/ping?host=localhost;cat+/etc/passwd"

# Chain multiple commands
curl "http://localhost:5000/api/ping?host=localhost;whoami;pwd"
```

