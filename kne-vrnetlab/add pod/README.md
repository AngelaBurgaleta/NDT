###  Add a new pod to a topology

Si se desea realizar cambios dinámicos en una topología existente ya creada con KNE, como la creación de un nuevo nodo en una topología ya establecida, se necesita crear un archivo YAML en el que se defina el pod (router) que se desea reutilizar del escenario para conectar el nuevo pod. Debes especificar el namespace donde esta creada la topología y la interfaz punto a punto junto con la subred que se utilizará para la conexión. Además, deberás especificar los detalles del nuevo pod que deseas desplegar, como la imagen, el archivo de configuración y los servicios que se expondrán. Un ejemplo de esto se puede encontrar en los archivos "new.yaml" y "new-csr.yaml".

Una vez que se haya creado el archivo para desplegarlo en el clúster, se ejecuta el siguiente comando:

> El archivo new.yaml crea un nuevo contenedor basado en Alpine, mientras que el archivo new-csr.yaml crea un contenedor utilizando el router CSR1000v en un topología ya creada con el nombre 2ceos.

```shell
kubectl apply -f new.yaml
```

Si se quiere eliminar el nuevo pod creado:
```shell
kubectl delete -f new.yaml
```

![2ceosnew](../../images/2ceosNew.png)