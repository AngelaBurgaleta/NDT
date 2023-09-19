# KNE - vrnetlab

Este documento es una guia de como integrar el proyecto [hellt/vrnetlab](https://github.com/hellt/vrnetlab "hellt/vrnetlab") sobre un arquitectura de kubernetes con el proyecto [KNE](https://github.com/openconfig/kne "KNE"), para el despliegue de topologías de red en un clúster multinodo.

En esta guía se asume que ya se ha creado y configurado un clúster de kubernetes de varios nodos, por lo que se debe verificar que el clúster esté utilizando un complemento de red de pod [compatible with MetalLB](https://metallb.universe.tf/installation/network-addons/).

#### Instalación de KNE en un clúster con 4 nodos.
A continuación se detalla los pasos a seguir para la instalación de KNE  y vrnetlab en un clúster multinodo de kubernetes: 
- #####  Clone el repositorio KNE 
Debe clonarse el repositorio de  [KNE](https://github.com/openconfig/kne "KNE") en la máquina que actua como el plano de control (controller) en el clúster:
  ```bash
git clone https://github.com/openconfig/kne.git
 ```
Dentro de la carpeta kne, instale el binario:
  ```bash
make install
 ```
-  #####  Instalar golang
Se requiere tener instalado golang en la máquina que actua como controller en el clúster, si ya esta instalado verifique la versión actual utilizando el comando *go version*, en caso de no estar instalado:

Instale la nueva versión:

  ```bash
 curl -O https://dl.google.com/go/go1.20.1.linux-amd64.tar.gz
 sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.20.1.linux- amd64.tar.gz
 rm go1.20.1.linux-amd64.tar.gz  
 ```

 Puede considerar añadir las rutas al PATH del sistema operativo en`~/.bashrc` para facilitar el acceso y comandos relacionados con Go.
  ```bash
  export PATH=$PATH:/usr/local/go/bin
  export PATH=$PATH:$(go env GOPATH)/bin
   ```
-  #####  Instalar Docker
Se requiere tener instalado docker en todos los nodos del clúster, en caso de no estar instalado puede hacerlo siguiendo estos [pasos](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository).

Luego de instalarlo, para  configurar y administrar el acceso a docker, ejecute los siguientes comandos:
  ```bash
sudo groupadd docker
sudo usermod -aG docker $USER
   ```
   -  #####  Implementar dependencias de KNE
KNE proporciona el comando deploy para configurar el ingress, cni y los controladores de proveedores necesarios, para instalar estas configuraciones se utilizó el fichero [external-multinode.yaml](https://github.com/openconfig/kne/blob/main/deploy/kne/external-multinode.yaml).

1.  Para instalar estas dependencias en su clúster, ejecute el siguiente comando:
  ```bash
kne  deploy  kne/deploy/kne/external-multinode .yaml
   ```
2. Cree una nueva red docker para usar en el clúster:
  ```bash
 docker network create multinode
   ```
#### Creación de imágenes con vrnetlab
El proceso de creación de las imágenes debe realizarse en las máquinas que actuan como nodos trabajadores (workers) en el clúster.
###### Cisco CSR1000v

- **Clone el repositorio hellt/vrnetlab en todos los workers**:

```shell
git clone https://github.com/hellt/vrnetlab.git
```

- **Modificar fichero launch.py**:

Para las imagenes creada con el proyecto hellt/vrnetlab debe modificar el fichero launch.py de la imagen que desee probar, la modificación del archivo consiste en establecer él parámetro connection-mode como valor por defecto tc , con la finalidad de que este modo de conexión sea el predefinido, para que se creen y conecten las interfaces tap a la interfaces del router como máquina virtual a través de qemu, para así lograr la interconexión entre la máquina virtual y el contenedor que lo envuelve.

> Esto es un fragmento del fichero launch.py del csr1000v:

```shell
if __name__ == "__main__":
    import argparse

    parser.add_argument(
        "--connection-mode",
        default="tc",
        help="Connection mode to use in the datapath",
    )
```
En el mismo archivo, se puede modificar la variable denominada STARTUP_CONFIG_FILE, la cual especifica la ruta dentro del contenedor donde se almacena el archivo de configuración del router. Por lo que, dicha ruta debe ser ajustada debidamente en el descriptor del escenario. 

> En en este caso, se modificó el valor de la variable a “/iosxe_config.txt".

```shell
STARTUP_CONFIG_FILE = "/ iosxe_config .txt"
```
- **Creación de la imagen**:

Una vez se ha modificado el fichero launch.py, coloque el archivo .qcow2 en el directorio csr y fuera del directorio docker y ejecúte el comando para crear la imagen como contenedor del dispositivo:

  ```bash
/vrnetlab/csr$ make docker image csr1000v-universalk9.17.03.06-serial.qcow2
```
Luego de ejecutar este comando se iniciará el proceso de creación de la imagen, una vez se ha completado con éxito la ejecución, debe ver la imagen creada con el nombre de *vrnetlab/vr-csr* en el repositorio local de docker, para comprobarlo ejecutamos el siguiente comando:
```bash
$ docker images
REPOSITORY           TAG            IMAGE  ID             CREATED            SIZE
vrnetlab /vr -csr     17.03.06      b6f24d94df87       2 weeks ago      1.89 GB
```
> En el descriptor utilizado en KNE para definir la topología por ejemplo "10csr.yaml", dentro del parámetro config, es necesario especificar tanto la imagen creada utilizando el proyecto hellt/vrnetlab como la ruta en la que se copiará el archivo de configuración del router con las variables config_path y config_file. De este modo:
```shell
  - name: r1
    vendor: CISCO
    os: "ios-xe"
    config:
      config_path: "/"
      config_file: "iosxe_config.txt"
      file: "r1-config"
      image: "docker.io/vrnetlab/vr-csr:17.03.06"
```

###### Cisco xrv9k 

Para la creación de esta imagen el proceso sería igual que el descrito para el modelo csr1000v.

###### Arista cEOS

Para crear imágenes de contenedores a partir de un archivo en formato .tar como en el caso del contenedor cEOS, es necesario cargar la imagen en los nodos trabajadores del clúster. A continuación, se describe el proceso paso a paso:

- **Cargar la Imagen:**

Primero, asegúrate de que la imagen en formato .tar (por ejemplo, cEOS-lab-4.29.2F.tar) esté presente en el directorio de trabajo de los nodos del clúster donde deseas importarla.

- **Importar la Imagen a Docker:**

Una vez que la imagen en formato .tar esté disponible en el directorio correcto, puedes ejecutar el siguiente comando en el mismo directorio para importarla en docker:

```bash
docker import cEOS-lab-4.29.2F.tar ceos
```

Este comando utiliza docker import para importar la imagen de contenedor desde el archivo .tar. Le asigna un nombre al contenedor importado en Docker, en este caso, "ceos".

Con estos pasos, habrás importado la imagen de contenedor al repositorio local de Docker con el nombre "ceos". Esto te permitirá utilizar la imagen en la creación y ejecución de contenedores Docker en tu clúster.


#### Creación de topologías con  KNE
Para crear una topología, debes agregar el directorio que contiene el archivo YAML de descripción del escenario y sus archivos de configuración correspondientes. En este repositorio, encontrarás ejemplos de topologías y configuraciones que puedes utilizar como punto de partida. Puedes simplemente copiar uno de estos ejemplos y colocarlo en la carpeta "example" del proveedor adecuado dentro de KNE .

>A continuación, se presenta un ejemplo de cómo crear una topología utilizando el archivo "10csr.yaml" de este respositorio en KNE:

1. Cree  la topología de 10 routers CSR1000V
  ```bash
kne create kne/examples/cisco/10csr.yaml
   ```

Para eliminar la topología creada:
  ```bash
kne delete kne/examples/cisco/csr/10csr.yaml
   ```
Para verificar el correcto despliegue de los pods dentro del clúster multinodo podemos utilizar el siguiente comando:
  ```bash
kubectl get pods -A -o wide -w
   ```
Una vez se ha creado la topología con éxito se puede acceder al dispositivo creado con vrnetlab, con las credenciales por defecto, que corresponde a l usuario y contraseña,  "vrnetlab" y“VR-netlab9” respectivamente y su dirección IP external, de esta forma.

> Con el comando *kubectl get svc -n namespace, se puede identificar la dirección IP externa.*

  ```bash
kubectl get svc -n 10-csr
NAME          TYPE                 CLUSTER-IP       EXTERNAL-IP  
service-r1     LoadBalancer   10.107.96.233    172.18.0.58 
service-r11   LoadBalancer   10.102.218.55    172.18.0.52   
service-r12   LoadBalancer   10.103.68.104    172.18.0.59   
service-r13   LoadBalancer   10.111.105.180  172.18.0.54   
service-r2     LoadBalancer   10.99.33.30        172.18.0.55   
service-r3     LoadBalancer   10.97.36.125      172.18.0.57   
service-r4     LoadBalancer   10.109.60.252    172.18.0.51   
service-r5     LoadBalancer   10.108.26.137    172.18.0.50   
service-r6     LoadBalancer   10.99.160.140    172.18.0.53   
service-r7     LoadBalancer   10.97.58.27        172.18.0.56   
 ```
 Pudiendo acceder al router de la siguiente forma
  ```bash
ssh vrnetlab@172.18.0.58
Password: VR-netlab9
   ```











