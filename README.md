# elastic-k8s

Procedimiento para la instalación de un clúster de Elasticsearch, Kibana, y Filebeat
utilizando Ubuntu 20.04, Kubernetes, y OpenEBS.

## Consejos

Antes de empezar, es conveniente realizar los siguientes pasos:

### 1. Configurar los nombres de los servidores

Para simplificar la gestión de los mismos es conveniente que cada servidore cuente
con un nombre que los identifique, y que siga cierta logica para indicar que pertenece
a un único grupo de servidores con un mismo proposito. Por ejemplo, compartiendo un
mismo prefijo que identifique su rol. En este ejemplo, utilizaremos el prefijo
`elastic-k8s` y un número al final separado por un guion (`-`).

- `elastic-k8s-01`
- `elastic-k8s-02`
- `elastic-k8s-03`

El proceso para configurar el `hostname` en cada servidor Ubuntu es el siguiente:

```bash
# 1.
# Veríficar el nombre actual
hostnamectl
# 2.
# Cambiar el nombre del servidor de forma apropiada
sudo hostnamectl set-hostname elastic-k8s-01
# 3.
# Editar el archivo `/etc/hosts` para que resuelva el nombre
sudo vim /etc/hosts
# 4.
# Verificar la existencia del archivo `/etc/cloud/cloud.cfg`.
cat /etc/cloud/cloud.cfg
# 5.
# Si existe, modificar la variable `preserve_hostname` a false.
sudo vim /etc/cloud/cloud.cfg
```

En el paso `3` al momento de modificar el archivo `/etc/hosts`, se recomienda agregar
entradas para resolver a los demás servidores del clúster por nombre. Por ejemplo,
para el servidor `elastic-k8s-01` el archivo quedaría parecido a este:

```bash
127.0.0.1 localhost
127.0.0.1 elastic-k8s-01
172.31.62.238 elastic-k8s-01
172.31.49.145 elastic-k8s-02
172.31.58.59  elastic-k8s-03
```

### 2. Modificar la zona horaria

Durante el proceso de instalación del servidor es posible configurar la zona horaria
correspondiente. Sin embargo, esto rara vez queda correctamente configurado. Es
conveniente todos los servidores estén configurados con la zona horaria que corresponde.

```bash
# 1.
# Veríficar la zona horaria actual
timedatectl
# 2.
# Aplicar la zona horaria de Montevideo
sudo timedatectl set-timezone America/Montevideo
```

### 3. Actualizar y aplicar todos los parches de seguridad al momento

Antes de comenzar la instalación, es reocomendable instalar todos los parches de
seguridad existentes al momento, y actualizar todas las dependencias existentes dentro
del sistema operativo.

```bash
# 1.
# Actualizar repositorios
sudo apt-get update
# 2.
# Actualizar dependencias
sudo apt-get upgrade -y
# 3.
# Instalar parches de seguridad
sudo apt-get full-upgrade -y
# 4.
# Reiniciar el servidor
sudo reboot
```

### 4. Instalar o verificar la instalación de `snap`

```bash
# 1.
# Instalar `snap`
sudo snap install core
# 2.
# Actualizar `snap`
sudo snap refresh core
```

## Instalación de `microk8s` (Kubernetes)

`microk8s` es una distribución de Kubernetes desarrollada por Canonical (creadores
de Ubuntu) que simplifica la puesta en marcha de clústers pequeños de Kubernetes.
Su proceso de instalación es muy sencillo, ya que no necesita de la instalación de
`etcd` para funcionar, y, puede ser instalado a través de `snap` de forma automatica.

El instalador no solamente instala todos los componentes de Kubernetes, sino el
motor de contenedores `containerd`. Por lo que no es necesario contar con níngun
otro motor de contenedores (como `docker`).

Los siguientes comandos deben ser ejecutados en todos los nodos.

```bash
# 1.
# Instalar microk8s
sudo snap install microk8s --classic
# 2.
# Agregamos permisos al usuario actual para manipular `microk8s`
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
su - $USER
# 3.
# Esperar a que la instalación finalice
microk8s status --wait-ready
```

Una vez completada la instalación en todos los equipos, debemos realizar el siguiente
procedimiento para que se descubran entre sí y levanten el clúster.

```bash
# 1.
# En un nodo ejecutar el siguiente comando
microk8s add-node
# 2.
# La salida de este comando debe ser copiada y pegada como entrada
# en otro de los nodos. Por ejemplo:
microk8s join 172.31.62.238:25000/be2169e5c9cd07677c665bfb064b62d2
# 3.
# Una vez terminado el proceso en el segundo servidor, repetir el proceso
# con los demás servidores.
microk8s join 172.31.62.238:25000/2c89cae107f61b2cf5360769d17196f6
# 4.
# Una vez agregados todos los servidores, veríficamos el estado del clúster
→ microk8s status
microk8s is running
high-availability: yes
  datastore master nodes: 172.31.62.238:19001 172.31.49.145:19001 172.31.58.59:19001
  datastore standby nodes: none
# 5.
# Listamos los nodos del clúste y veríficamos que todos hayan sido agregados
# correctamente.
→ microk8s kubectl get nodes
NAME             STATUS   ROLES    AGE    VERSION
elastic-k8s-02   Ready    <none>   28m    v1.20.1-34+e7db93d188d0d1
elastic-k8s-03   Ready    <none>   2m8s   v1.20.1-34+e7db93d188d0d1
elastic-k8s-01   Ready    <none>   36m    v1.20.1-34+e7db93d188d0d1
```

Si todo funciono correctament deberíamos ver a todos los servidores en la lista.

Es recomendable crear una serie de aliases (en `./bashrc` o `./bash_aliases`) para
simplificar la interacción con `microk8s kubectl`. Por ejemplo:

```bash
# ~/.bash_aliases
alias kubectl='microk8s kubectl'
alias k='kubectl'
```

Ahora tenemos que habilitar una serie de `add-ons` para completar la instalación:

- `dns`: Servicio de `CoreDNS` necesario para el `discovery` de servicios.
- `rbac`: Role Bases Authentication Control.

Se pueden instalar todos ellos así:

```bash
microk8s enable dns rbac
```

Al finalizar la instalación contaremos con un clúster configurado para resolver el
nombre de los distintos recursos de Kubernetes y la capacidad de exponer de forma
segura y con alta disponibilidad los servicios del clúster.

## Instalación de OpenEBS

Solución Open Source para el manejo de `storage` dentro de un clúster de Kubernetes.

OpenEBS soporta una serie de motores que permiten dotar a los volumenes de múltiples
cualidades. Para este proyecto, el motor que se utilizará es: Local PV (hostpath).

[Documentación de OpenEBS](https://docs.openebs.io/)

![OpenEBS Architecture](https://docs.openebs.io/docs/assets/svg/openebs-arch.svg)
![OpenEBS Configuration Sequence](https://docs.openebs.io/docs/assets/svg/1-config-sequence.svg)

### Pre-requisitos

Es fundamental que el cliente de `iSCSI` de los nodos este correctamente configurado
previo a levantar OpenEBS.

Los servidores de Ubuntu pueden ya tener instalado el iniciador de `iSCSI`. Esto se
puede veríficar de la siguiente manera:

```bash
→ sudo cat /etc/iscsi/initiatorname.iscsi
GenerateName=yes

→ systemctl status iscsid
● iscsid.service - iSCSI initiator daemon (iscsid)
     Loaded: loaded (/lib/systemd/system/iscsid.service; disabled; vendor preset: enabled)
     Active: inactive (dead)
TriggeredBy: ● iscsid.socket
       Docs: man:iscsid(8)
```

Si esta inactivo sera necesario activarlo:

```bash
sudo systemctl enable --now iscsid
```

En el caso de que no este instalado, podemos hacerlo con los siguientes comandos:

```bash
sudo apt-get update
sudo apt-get install open-iscsi
sudo systemctl enable --now iscsid
sudo systemctl start --now iscsid
```

Además, es necesario que este levantado el módulo del `kernel` `iscsi_tcp`. Podemos
veríficar su funcionamiento corriendo el siguiente comando:

```bash
→ lsmod | grep iscsi
iscsi_tcp              24576  0
libiscsi_tcp           32768  1 iscsi_tcp
libiscsi               57344  2 libiscsi_tcp,iscsi_tcp
scsi_transport_iscsi   110592  4 libiscsi_tcp,iscsi_tcp,libiscsi
```

Si la salida no es equivalente a la mostrada en el ejemplo, significa que el módulo
no esta siendo ejecutado. Utilizamos el comando `modprobe` para levantarlo.

```bash
sudo modprobe iscsi_tcp
```

Ahora si la salida debería ser similar a la expuesta anteriormente.

Queda todavía un paso más. El último comando solo hace que el sistema operativo inicie
el módulo una vez. Si se reinicia el nodo, el módulo no se volverá a levantar de forma
automática. Debemos indicarle al sistema operativo que debe levantar el mismo automáticamente
cuando inicie. Esto lo hacemos agregando la línea `iscsi_tcp` al archivo `/etc/modules`

```bash
→ cat /etc/modules
# /etc/modules: kernel modules to load at boot time.
#
# This file contains the names of kernel modules that should be loaded
# at boot time, one per line. Lines beginning with "#" are ignored.
iscsi_tcp
```

### Instalación

Para instalar OpenEBS es necesario aplicar los recursos listados en el archivo
`./openebs-operator.yaml`.

```bash
k apply -f openebs/operator.yaml
```

Si todo sale bien podemos veríficar la instalación listando los `pods` correspondientes
al nuevo `namespace` `openebs`:

```bash
k get pods -n openebs
```

También necesitamos veríficar que todas las clases de almacenamiento por defecto
(`StorageClasses`) hayan sido creadas con éxito:

```bash
k get storageclass
```

En este proyecto, estamos interesados en utilizar la `StorageClass = openebs-hostpath`.

OpenEBS ofrece una utilidad que simplifica la administración de los volumenes dentro
del cluster: `mayactl`. La misma puede accederse dentro del `pod` que contiene el
servicio `maya-apiserver`. Para simplificar su uso, se sugiere utilizar la siguiente
función, la cual puede cargarse en `~/.bashrc` o `~/.bash_profile`.

```bash
# Permite acceder de forma dínamica a la utilidad `mayactl` dentro del
# pod `maya-apiserver`.
#
# @example
# mayactl list volumes
function mayactl {
  apiserver=`microk8s kubectl get pod -n openebs | grep -i api | awk '{print $1}'`

  microk8s kubectl -n openebs exec $apiserver -- /usr/local/bin/mayactl $@
}
```

Una vez cargada esta función (`source ~/.bashrc`) se puede utilizar de la siguiente
manera:

```bash
# Mensaje de ayuda
mayactl help

# Listar volumenes
mayactl volume list

# Describir un volumen
mayactl volume describe --volname pvc-a57c55ab-5773-4f44-b70b-5f17d0cee28e -n openebs

# Obtener las estádisticas del volumen
mayactl volume stats --volname pvc-a57c55ab-5773-4f44-b70b-5f17d0cee28e -n openebs
```

## ECK

ECK o Elastic Cloud on Kubernetes es un operador que simplifica las tareas de gestión
de un clúster de Elasticsearch sobre Kubernetes.

[Documentación](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html)

El primer paso es instalar el operador. A diferencia del caso anterior, vamos a
descargar la última versión del operador directamente desde Internet:

```bash
k apply -f https://download.elastic.co/downloads/eck/1.3.1/all-in-one.yaml
```

Podemos monitorear el proceso de instalación de la siguiente manera:

```bash
k -n elastic-system logs -f statefulset.apps/elastic-operator
```

**Puede demorar unos segundos en estar disponible.**

Luego de finalizada la instalación del operador, procederemos a levantar un clúster
de Elasticsearch con 3 nodos. Los archivos mencionados en el siguiente procedimiento
se pueden encontrar en la raíz del proyecto.

```bash
# 1.
# Creamos el clúster de Elasticsearch
k apply -f elasticsearch/deployment.yaml
# 2.
# Monitoreamos el estado del clúster
k get elasticsearch
# Puede demorar unos minutos finalizar el proceso.
# 3.
# Listamos los pods asociados al cluster
k get pods --selector='elasticsearch.k8s.elastic.co/cluster-name=elastic-k8s'
```

El clúster estará listo para utilizar cuando indique que esta `green`:

```bash
→ k get elasticsearch
NAME          HEALTH   NODES   VERSION   PHASE   AGE
elastic-k8s   green    3       7.10.2    Ready   13m
```

Además de los `pods` que corresponden al clúster, el operador configura un servicio
para poder acceder al mismo, y las credenciales del usuario administrador llamado
`elastic`. Para obtener el valor de la contraseña almacenada como un secreto utilizamos
el siguiente comando:

```bash
k get secret elastic-k8s-es-elastic-user -o go-template='{{.data.elastic | base64decode}}'
```

Para veríficar el acceso al clúster podemos utilizar la función `port-forward`de
`kubectl`. Como los certificados autogenerados por el operador solo responden a
consultas desde dentro del clúster o `localhost` solo podemos probar el acceso desde
el mismo servidor donde se exponen los puertos. Por lo tanto, correremos la función
`port-forward` en segundo plano.


```bash
# 1.
# Abrimos el puerto escuchando en `localhost:9200` en segundo plano
k port-forward service/elastic-k8s-es-http 9200 &
# 2.
# Veríficamos que el proceso se encuentra levantado
→ jobs -lr
[1]+ 225330 Running   microk8s kubectl port-forward service/elastic-k8s-es-http 9200 &
# 3.
# Obtenemosla contraseña y la almacenamos en una variable de entorno.
export PASSWORD=$(k get secret elastic-k8s-es-elastic-user -o go-template='{{.data.elastic | base64decode}}'
)
# 4.
# Probamos el acceso
→ curl -u "elastic:$PASSWORD" -k "https://localhost:9200"
Handling connection for 9200
{
  "name" : "elastic-k8s-es-masters-2",
  "cluster_name" : "elastic-k8s",
  "cluster_uuid" : "PDuVxYEbRtKKb8Rias19lg",
  "version" : {
    "number" : "7.10.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "747e1cc71def077253878a59143c1f785afa92b9",
    "build_date" : "2021-01-13T00:42:12.435326Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
# 5.
# Detenemos el proceso que se encuentra ejecutando en segundo plano.
pkill -9 -f "port-forward"
```

Ahora que tenemos el clúster de Elasticsearch levantado, el siguiente paso es
levantar el dashboard de Kibana.

```bash
# 1.
# Hacemos el deploy del recurso Kibana
k apply -f kibana/deployment.yaml
# 2.
# Obtenemos el estado de los `pod`
k get pod --selector='kibana.k8s.elastic.co/name=elastic-k8s'
# 3.
# Monitoreamos el estado del recurso
k get kibana
```

Para probar el acceso a Kibana, podemos utilizar nuevamente la función `port-forward`
de `kubectl`.

```bash
k port-forward service/elastic-k8s-kb-http 5601
```

Si vamos a la IP del servidor `elastic-k8s-01` en el puerto `5601` desde un explorador
podremos acceder al dashboard de Kibana. El usuario por defecto es también `elastic`
y la contraseña se puede obtener de la siguiente manera:

```
k get secret elastic-k8s-es-elastic-user -o=jsonpath='{.data.elastic}' | base64 --decode;echo
```

<details>
  <summary>¿Como puedo obtener certificados para hacer pruebas?</summary>
<p>
Para conseguir un certificado con el que probar antes de hacer la implementación
sigua estas instrucciones:
</p>

<pre># 1.
# Instale certbot
sudo snap install --classic certbot
# 2.
# Preparar el comando `certbot`
sudo ln -s /snap/bin/certbot /usr/bin/certbot
# 3.
# Con el servidor apagado y los registros DNS apuntando al mismo
# ejecutar el siguiente comando. Esto levantará un servidor web en
# el servidor para validar los certificados.
sudo certbot certonly \
 --manual \
 --preferred-challenges=dns \
 --email gmonne@conatel.com.uy \
 --server https://acme-v02.api.letsencrypt.org/directory \
 --agree-tos \
 -d *.k8s.conatest.click
# OBS1: Es importante sustituir el dominio `kibana.k8s.conatest.click`
# por el que realmente corresponda.
# OBS2: Durante el proceso se le pedira que ingrese la dirección de correo
# del administrador del dominio que piensa utilizar.
# 4.
# Obtenemos el certificado y la llave.
sudo cp /etc/letsencrypt/live/k8s.conatest.click/cert.pem ./tls.crt
sudo cp /etc/letsencrypt/live/k8s.conatest.click/privkey.pem ./tls.key
# 5.
# Modificamos los permisos para evitar problemas
sudo chown $USER tls.*
</pre>
</details>
<br />
<details>
  <summary>¿Como puedo crear mis propios certificados?</summary>
  Podemos generar nuestros propios certificados utilizando openssl. Es importante
  notar que no serán detectados por los exploradores debido a que no han sido firmados
  por una CA conocida.

  <pre>
# 1.
# Definimos algunas variables de entornos de interes
export KEY_FILE=tls.key
export CERT_FILE=tls.crt
export HOST=*.k8s.conatest.click
# 2.
# Creamos el certificado y su llave
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout ${KEY_FILE} \
  -out ${CERT_FILE} \
  -subj "/CN=${HOST}/O=${HOST}"
  </pre>
</details>

<br/>

Vamos a suponer que el certificado y la clave TLS se encuentran en la raiz del
servidor donde estamos trabajando, llamadas `tls.crt` y `tls.key`. Para que nuestro
`Ingress` sepa que certificados vamos a utilizar, debemos almacenarlos en un `Secret`.

```bash
# 1.
# Configuramos los nombres de los archivos a utilizar
export KEY_FILE=tls.key
export CERT_FILE=tls.crt
# 2.
# Creamos el secreto
k create secret tls ssl-certificate --key ${KEY_FILE} --cert ${CERT_FILE}
```

Si describimos el resultado de este secreto, veremos que nuestros archivos fueron
almacenados encodeados en base64.

```bash
k edit secret ssl-certificate
```

Ahora lo que queda es levantar el `Ingress` para acceder a Kibana. Este es un recurso
utilizado para publicar servicios fuera del clúster.

```bash
# 1.
# Habilitar el ingress en el clúster
microk8s enable ingress:default-ssl-certificate=default/ssl-certificate
# 2.
# Levantar el ingress de Kibana
k apply -f kibana/ingress.yaml
```

Al finalizar de aplicar los cambios vamos a poder acceder al dashboard de Kibana
a través del dominio configurado.

```
https://kibana.k8s.conatest.click
```

## CISCO con Filebeat

Filebeat es un producto desarrollado por Elastic que simplifica el proceso de ingesta
de datos de múltiples fuentes. El mismo, cuenta con dos mecanismos para recopilar
información: `inputs` y `modules`. Las `inputs` son la versión anterior de los
`modules` y siguen soportados por continuidad. Sin embargo, se recomienda utilizar
`modules` en nuevas implementaciones.

En este caso, utilizaremos Filebeat como servidor de Syslog. Utilizaremos un `DaemonSet`
para levantar un `pod` de Filebeat en cada servidor, y un `Service` para llegar hasta el.
También configuraremos el `Ingress` para escuchar en los puertos del Syslog.

Filebeat ya cuenta con un modulo diseñado para recibir y parsear `logs` de dispositivos
Cisco. [Este modulo](https://www.elastic.co/guide/en/beats/filebeat/7.10/filebeat-module-cisco.html) simplifica el parseo de mensajes de Syslog, así como la creación de indices
para su almacenamiento. Lamentablemente, solo incluye un `dashboard` predefinido
para trabajar con `logs` de dispositivos `ASA`. El módulo es capaz de manejar `logs`
de: `asa`, `firepower`, `ios`, `nexus`, `meraki`, y `umbrella`. Configuraremos todos
menos el de `umbrella` dado que requiere de acceso a un `Bucket` de AWS S3, en donde
`umbrella` deja disponible los `logs`.

Todos los archivos disponibles a esta configuración se encuentran en la carpeta `filebeat`.

El primer paso es levantar los recursos en Kubernetes y correr algunos comandos
necesarios para configurar Elasticsearch y Kibana.

```bash
# 1.
# Aplicamos los recursos
k apply -f filebeat/filebeat-cisco.yaml
# 2.
# Esperamos unos segundos a que terminen de levantarse los contenedores
# Puede ser que no sea suficiente para que los pods estén listos. En ese caso,
# verificar su estado, y repetir los pasos 3 y 4.
sleep 30
# 3.
# Obtenemos el id de alguno de los pods de FILEBEAT
export FILEBEAT_POD=$(k get all | grep pod/filebeat | awk '{print $1}' | head -n 1)
# 4.
# Configuramos los recursos en Elasticsearh y Kibana
k exec $FILEBEAT_POD -- ./filebeat setup --dashboards -c /etc/filebeat.yml
k exec $FILEBEAT_POD -- ./filebeat setup --index-management -c /etc/filebeat.yml
k exec $FILEBEAT_POD -- ./filebeat setup --pipelines -c /etc/filebeat.yml
```

La configuración de Filebeat utilizada se puede encontrar dentro del `ConfigMap`
llamado `filebeat-cisco-config`. Este es un ejemplo de como se ve:

```yaml
# k describe configmap filebeat-cisco-config
Name:         filebeat-cisco-config
Namespace:    default
Labels:       app=filebeat
              ingest=cisco
Annotations:  <none>

Data
====
filebeat.yml:
----
filebeat.modules:
  - module: cisco
    asa:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9001
      var.log_level: 5
    ftd:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9003
      var.log_level: 5
    ios:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9002
    nexus:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9506
      var.tz_offset: -03:00
    meraki:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9525
      var.tz_offset: -03:00
    umbrella:
      enabled: false

setup.template.settings:
  index.number_of_shards: 2
  index.number_of_replicas: 2

setup.kibana.host: "http://${KIBANA_HOST}:${KIBANA_PORT}"
setup.kibana.ssl.enabled: true

output.elasticsearch.hosts: ['https://${ELASTICSEARCH_HOST}:${ELASTICSEARCH_PORT}']
output.elasticsearch.username: ${ELASTICSEARCH_USERNAME}
output.elasticsearch.password: ${ELASTICSEARCH_PASSWORD}
output.elasticsearch.ssl.certificate_authorities: ["/etc/pki/root/ca.crt"]
output.elasticsearch.ssl.certificate: "/etc/pki/root/tls.crt"
output.elasticsearch.ssl.key: "/etc/pki/root/tls.key"
Events:  <none>
```

Se puede apreciar de la salida los puertos en donde estamos escuchando por los `logs`
de cada familia. Por ahora estos puertos solo están disponibles dentro del clúster
de Kubernetes. El siguiente paso es publicarlos.

<details>
<summary><b>Es necesario haber cargado el add-on de <code>ingress</code>.</summary>
<pre>
microk8s enable ingress:default-ssl-certificate=default/ssl-certificate
<pre>
</details>
<br/>

Por defecto, los `Ingress` en Kubernetes solamente permiten la publicación de servicios
HTTP. Esto es debeido a que es díficil identificar a que servicio se debería rutear
el tráfico cuando se trabaja con otros protocolos de red. El `Ingress` de NGINX evita
esta limitiación al permitir la utilización de un `ConfigMap` en donde se indique
a que servicio se debe rutear el tráfico recibido en otros puertos.

Por lo tanto, para poder publicar los puertos de Syslog en nuestro cluster vamos a
tener que:

1. Abrir los puertos en los pods de Filebeat.
2. Abrir los puertos en el servicio de Filebeat.
2. Abrir los puertos en los pods del `nginx-ingress`.
3. Actualizar el `ConfigMap` llamado `nginx-ingress-udp-microk8s-conf` con la lista
de puertos que deben ser públicados.

Los primeros dos pasos ya fueron configurados al aplicar el archivo
`filebeat/filebeat-cisco.yaml`.

La información dentro del `ConfigMap nginx-ingress-udp-microk8s-conf` debe estar compuesta por un mapa en donde las
llaves correspondan a los puertos a públicar, y los valores indiquen el `Service`,
`Namespace` y el puerto donde se debe rutear el tráfico. En nuestro caso, esto sería
algo así:

```yaml
data:
  9001: "default/filebeat-cisco:9001"
  9003: "default/filebeat-cisco:9003"
  9002: "default/filebeat-cisco:9002"
  9506: "default/filebeat-cisco:9506"
  9525: "default/filebeat-cisco:9525"
```

Para realizar estas modificaciones vamos a utilizar el cómando `patch` de `kubectl`
que permite combinar la configuración existente de un recurso con nuevas opciones.

```bash
# 1.
# Patcheamos los contenedores de `nginx` para que escuchen el puerto de netflow
k -n ingress patch daemonset.apps/nginx-ingress-microk8s-controller \
  --patch "$(cat filebeat/nginx-ingress-microk8s-controller-cisco-patch.yaml)"
# 2.
# Patcheamos la configuración del `Ingress` de UDP
k -n ingress patch configmap/nginx-ingress-udp-microk8s-conf \
  --patch "$(cat filebeat/nginx-ingress-udp-microk8s-conf-cisco-patch.yaml)"
```

Este proceso hara que se vuelvan a crear todos los `pods` asociados al `nginx-ingress`.
Una vez que todos esten nuevamente en estado `Ready` podremos comenzar a enviar
`logs` a Elasticsearch a través de Filebeat.

## Netflow

Filebeat tambien permite la configuración de un modulo para recibir Netflow. El
proceso para dejar funcionando este módulo es equivalente al anterior pero abriendo
un puerto distinto. Sin embargo, si ya contamos con Filebeat operando dentro del
clúster, no tiene sentido que levantemos `pods` adicionales para recibir el tráfico
de Netflow. En cambio, lo mejor es utilizar los `pods` existentes, simplemente
cambiando la configuración.

Realizaremos exactamente esto para dejar Netflow funcionando en el clúster.

Primero debemos agregar la siguiente configuración al `ConfigMap` que contiene la
configuración de Filebeat. La actualización de un `ConfigMap` no genera la actualización
de los recursos que estén utilizando del mismo. Solamente nuevos recursos que apunten
a el utilizarán las nuevas configuraciones. Esto es porque se entiende que los `ConfigMap`
son entidades inmutables. Osea, que no cambian.

¿Como actualizamos un `ConfigMap` entonces?

No lo hacemos. En cambio, creamos un nuevo `ConfigMap` y apuntamos los recursos que
lo deben consumir al mismo. Esto tiene la ventaja que si el `ConfigMap` contiene algún
error, los recursos fallarán al momento de actulizarse, y volverán a su estado anterior
sin necesidad de interacción por el administrador, y evitando caídas de sistema.

Entonces, debemos crear un nuevo `ConfigMap` igual al anterior, agregando la configuración
del módulo de Netflow. Llamaremos este configmap como `filebeat-config-YYYY-MM-DD.yaml` para
indicar su versión.

**Modificar el sufijo de acuerdo a la fecha en que se realicen estos pasos.**

```yaml
- module: netflow
  log:
    enabled: true
    var:
      netflow_host: 0.0.0.0
      netflow_port: 2055
```

Por ahora nos limitaremos a crear el nuevo `ConfigMap`. Más adelante, mostraremos
como se puede modificar el `DaemonSet` para que consuma esta nueva configuración.

**Antes de copiar y pegar estos comandos, verifique que la configuración utilizada
dentro del `ConfigMap` se encuentra correctamente configurada:**

```bash
# 1.
# Creamos el nuevo archivo que filebeat.
# Las comillas en 'EOF' indican que no queremos expansión de variables.
cat >> filebeat.yml << 'EOF'
filebeat.modules:
  - module: netflow
    log:
      enabled: true
      var:
        netflow_host: 0.0.0.0
        netflow_port: 2055
  - module: cisco
    asa:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9001
      var.log_level: 5
    ftd:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9003
      var.log_level: 5
    ios:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9002
    nexus:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9506
      var.tz_offset: -03:00
    meraki:
      enabled: true
      var.syslog_host: 0.0.0.0
      var.syslog_port: 9525
      var.tz_offset: -03:00
    umbrella:
      enabled: false

setup.template.settings:
  index.number_of_shards: 2
  index.number_of_replicas: 2

setup.kibana.host: "http://${KIBANA_HOST}:${KIBANA_PORT}"
setup.kibana.ssl.enabled: true

output.elasticsearch.hosts: ['https://${ELASTICSEARCH_HOST}:${ELASTICSEARCH_PORT}']
output.elasticsearch.username: ${ELASTICSEARCH_USERNAME}
output.elasticsearch.password: ${ELASTICSEARCH_PASSWORD}
output.elasticsearch.ssl.certificate_authorities: ["/etc/pki/root/ca.crt"]
output.elasticsearch.ssl.certificate: "/etc/pki/root/tls.crt"
output.elasticsearch.ssl.key: "/etc/pki/root/tls.key"
EOF
# 2.
# Creamos el nuevo `ConfigMap`
k create configmap filebeat-config-2021-01-30 --from-file=filebeat.yml
```

Se puede apreciar que los `pods` de Filebeat escucharán por los mensajes de Netflow
en el puerto `2055/udp`. Por lo tanto, vamos a tener que abrir este puerto en:

1. Los `pods` de Filebeat.
2. Los `pods` del `nginx-ingress`.

Además, vamos a tener que modificar el `Service` de Filbeat para incluir este puerto.
Y modificar el `ConfigMap` llamado `nginx-ingres-udp-microk8s-conf` para que le indique
al `nginx-ingress` a donde mandar el tráfico de Netflow.

Vamos a utilizar el comando `patch` de `kubectl` para aplicar estos cambios en los
recursos mencionados.

```bash
# 1.
# Abrimos el puerto 2055 en los `pods` de Filebeat y modificamos la configuración
# para que consuma el nuevo `ConfigMap`.
# OBS: Verifique el nombre del `ConfigMap` antes de realizar estos cambios.
k patch daemonset filebeat --patch "$(cat <<- EOF
spec:
  template:
    spec:
      containers:
        - name: filebeat
          ports:
            - containerPort: 2055
              protocol: UDP
      volumes:
        - name: config
          configMap:
            defaultMode: 0640
            name: filebeat-config-2021-01-30
EOF
)"
# 2.
# Abrimos el puerto 2055 en los `pods` de `ingress-nginx`.
k -n ingress patch daemonset.apps/nginx-ingress-microk8s-controller --patch "$(cat <<- EOF
spec:
  template:
    spec:
      containers:
        - name: nginx-ingress-microk8s
          ports:
            - containerPort: 2055
              hostPort: 2055
              name: netflow
              protocol: UDP
EOF
)"
# 3.
# Agregamos el puerto 2055 al `Service` filebeat-cisco
k patch service filebeat-cisco --patch "$(cat <<- EOF
spec:
  ports:
    - name: netflow
      port: 2055
      targetPort: 2055
      protocol: UDP
EOF
)"
# 4.
# Indicamos a `ingress-nginx` el puerto donde debe enviarnos el tráfico de Netflow.
k -n ingress patch configmap/nginx-ingress-udp-microk8s-conf --patch "$(cat <<- EOF
data:
  2055: "default/filebeat-cisco:2055"
EOF
)"
```

Listo. Ahora el clúster esta escuchando por Netflow en el puerto 2055.