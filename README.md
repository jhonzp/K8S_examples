# K8S_examples Namespaces, RBAC, Service Account
## 1_NameSpaces
### Namespaces
#### Los namespaces permiten definir un Scope y hacer una separación lógica dentro de nuestro cluster de K8S.
#### Nos facilita y permite aplicar reglas a este nivel sobre el uso de recursos, permisos entre otros.
#### Pero mucho cuidado es solo una separación lógica, los recursos físicos del cluster se comparten entre todos.
#### Como ya sabemos existen objetos básicos en K8s como Pod, Replica Set, Deployment al crear todos estos si no especificamos el namespace 
#### quedaran en el namespace default.

### Crear objetos de k8s sin especificar el ns.
#### Realizar el Kubectl apply -f 
#### de cada una de las carpetas e identificar el namespace con el que queda creado cada tipo de recurso.
#### De esta manera se puede evidenciar que existe un namespace "default" por defecto bajo el que se crean los objetos  cuando no 
#### especificamos el namespace.

## 2_RBAC - Users Groups

## DEV
### Crear una solicitud de creación de usuario dev01 utilizando un (CSR) certificate signing request usando openssl
#### * Generar una key  
            openssl genrsa -out dev01.key 2048
#### * Generar una CSR (usamos la key generada) /CN=usuario  /O=Grupo
            openssl req -new -key dev01.key -out dev01.csr -subj "/CN=dev01/O=devteam"
#### * Codificar en base64 el csr generado. (PowerShell)
            $csrdev01 = get-content 'dev01.csr' -Encoding Byte 
            $base64 = [System.Convert]::ToBase64String($csrdev01)
            Set-Content -Path dev01base64.csr -Value $base64 

## Admin
### Crear un csr (CertificateSigningRequest) y aprobarlo para la conexión.
#### * Apply del yml. para crear un csr (2_RBAC\Users_Groups\filesdev\dev01csr.yaml)
        apiVersion: certificates.k8s.io/v1beta1
        kind: CertificateSigningRequest
        metadata:
            name: mycsr
        spec:
        groups:
        - system:authenticated
        request: (CSR BASE64)
        usages:
        - digital signature
        - key encipherment
        - server auth
        - client auth
#### * Aprobar el csr creado
        kubectl get csr
        kubectl certificate approve mycsr
#### * Obtener el certificado en base64 (powershell)
        $csrdev01_admin = kubectl get csr mycsr -o jsonpath='{.status.certificate}'
        Set-Content -Path dev01.crt -Value $csrdev01_admin
#### * Decodificar el base64 a un crt (En Linux)
        echo 'stringB64' | base64 -d > file.crt
#### * Enviar el crt al DEV.

## DEV
### Usar el certificado para crear un Context y realizar la conexión
#### * En Windows es necesario copiar el .key y el .crt a la carpeta \.Kube 
        copy .\dev01.key C:\Users\zuni230669\.kube
        copy .\dev01.crt C:\Users\zuni230669\.kube
#### * Configurar las credenciales tomando los archivos de la carpeta \.Kube 
        kubectl config set-credentials dev01 --client-certificate=C:\Users\zuni230669\.kube\dev01.crt --client-key=C:\Users\zuni230669\.kube\dev01.key
#### * Configurar el nuevo Context dev01
        kubectl config set-context dev01 --cluster=Devk8sCluster --user=dev0
#### * Hacer switch al nuevo Context dev01 y probar la conexión.
        kubectl config use-context dev01
        kubectl get pods -n dev

## Admin
### Dar permisos haciendo uso de RBAC (roles, clusterRole, RoleBinding, ClusterRoleBinding).
         kubectl apply -f 2_RBAC\1_Users_Groups\2_dev01-pods.yaml

## 2_RBAC - service account
#### * Explorando un service account
        kubectl get sa -n dev
        kubectl get sa -n prod
        kubectl describe sa default -n prod
        kubectl get secrets -n prod
        kubectl describe secret default-token-sspjz -n prod
#### * Relación entre un pod y un service account (seleccionamos un pod)       
        kubectl describe pod deployment-prod-b7c99d94b-7h4sq -n prod 
        kubectl exec -ti deployment-prod-b7c99d94b-7h4sq -n prod -- sh
#### * Peticiones directas al Api de K8S 
####   (https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.19/#list-pod-v1-core)
        Kubectl get svc
        kubectl get svc kubernetes
        kubectl exec -ti deployment-prod-b7c99d94b-7h4sq -n prod -- sh
        (verificar la ruta del mount del volumen)
        apk add curl
        curl https://10.0.0.1/api/v1/namespaces/prod/pods --insecure
        curl https://10.0.0.1/api/v1 --insecure
#### * Hacer peticiones desde el pod usando el token del sa 
        TOKEN=$(cat token)
        curl -H "Authorization: Bearer ${TOKEN}" https://10.0.0.1/api/v1 --insecure
        curl -H "Authorization: Bearer ${TOKEN}" https://10.0.0.1/api/v1/namespaces/prod/pods --insecure
        exit
        kubectl delete -f 1_NameSpaces\5_namespaces\2_env.yaml
#### * Crear un service account y utilizarlo en el deployment
        kubectl apply -f 2_RBAC\2_ServiceAccount\2_env.yaml
        kubectl describe pod deployment-prod-b7c99d94b-7h4sq -n prod 
        kubectl exec -ti deployment-prod-b7c99d94b-7h4sq -n prod -- sh
#### * Aplicar los permisos al service Account.
        kubectl apply -f .\2_RBAC\2_ServiceAccount\3_sa-role.yaml

#### * Hacer peticiones desde el pod usando el token del sa creado
        cd /var/run/secrets/kubernetes.io/serviceaccount
        TOKEN=$(cat token)
        curl -H "Authorization: Bearer ${TOKEN}" https://10.0.0.1/api/v1 --insecure
        curl -H "Authorization: Bearer ${TOKEN}" https://10.0.0.1/api/v1/namespaces/prod/pods --insecure   

        













