# Despliegue GKE barato para proyectos personales
Despliegue GKE con Deployment Manager, traefik y el coste son unos 15 euros con dos nodos de 2C/4G. **Para proyectos personales.**

En vez de balanceador, se usa una IP flotante y un controller (`kubeip`) que va asignando la IP a un nodo siempre. Los nodos son preemtibles, por lo que dura menos de 24 horas en Google Cloud, pero kubernetes hace su magia. Tendras un downtime de un 1 minuto al dia, que es el tiempo que tarda el controller en darse cuenta que el nodo ha caído y reasigna la IP. Google Cloud tiene services con Network Endpoints.

La version de kubernetes engine es la la 1.20.11-gke.1300, se puede cambiar en el archivo gke.yaml linea 13 y 31.
La version de traefik es la traefik:v2.5.6. NOTA: con versiones kubernetes 1.16+ los cdr de traefik con `apiextensions.k8s.io/v1beta1` estan deprecados, pueden saltar errores.
Traefik puede resolver los acme para dominios. Modificar los parametros en el archivo `kubernetes-cluster.jinja` en la linea 773. ` --certificatesresolvers.default.acme.tlschallenge`

Los archivos .yaml son los valores que se utilizaran para el despliegue del network y gke. Osea jinja lee del yaml y va reemplazando los variables con los valores del yaml. Para mayor agilidad, flexibilidad se podria realizar con python, terraform...

El proyecto lleva un iam en la carpeta `/iam` comentado en caso de necesitarlo cicd,etc.... Tambien en el archivo `gke.yaml` lleva un iam comentado.

## Prerequisitos

De inicio tenemos para que configurar una serie de requisitos:

- fijar la ID del projecto
- fijar la region. por defecto esta fijado en europa.
  - region: europe-west1. Para cambiarlo `Makefile` linea 7. El archivo `vpc-network.jinja.schema` linea 37
  - zona: europe-west1-c. Para cambiarlo `kubernetes-cluster.jinja.schema` linea 101


### Fijar ID del proyecto
Para configurar el PROJECT_ID lanzamos y el nombre del proyecto que tengamos o queramos:
`export PROJECT_ID=NOMRE_PROYECTO_QUERAMOS_TENGAMOS`


### Fijar region
Despues de lanzar el comando, lanzamos un `make` y obtendremos un listado parecido a:

```Makefile
gcloud-config-set-base
dm-update-shared/container/gke
dm-update-shared/compute/network
dm-preview-shared/container/gke
dm-preview-shared/compute/network
dm-create-shared/container/gke
dm-create-shared/compute/network
```

El prinicipio se llama `dm` por Deployment Manager. El menu esta estructurado para crear (`..create..`), visualizar (`..preview..`), actualizar (`..update..`)

El menu esta subdividido entre `gke` y `network` osea podemos crear el despliegue de `gke` y `network`, actualizar `gke` y `network`, previsualizar `gke` y `network`

### Conociendo mas el Makefile 

```Makefile
gcloud-config-set-base              Fija region y PROJECT ID desde export

dm-update-shared/container/gke      Actualiza despliegue de GKE
dm-update-shared/compute/network    Actualiza despliegue de network

dm-preview-shared/container/gke     Previsualiza config GKE
dm-preview-shared/compute/network   Previsualiza config network

dm-create-shared/container/gke      Crea despliegue de GKE
dm-create-shared/compute/network    Crea despliegue de network
```

Como utilizaremos el Deployment Manager necesitaremos tener rol de `Owner` para la cuenta [project_number]@cloudservices.gserviceaccount.com

el comando:
```bash
gcloud projects add-iam-policy-binding $(gcloud config get-value project) \
--member serviceAccount:$(gcloud projects describe $(gcloud config get-value project) \
--format="value(projectNumber)")@cloudservices.gserviceaccount.com --role roles/owner
```

## Configuracion

Como primer paso fijamos la region y project_id con el comando:

```bash
make gcloud-config-set-base
```


## Despliegue network


Si queremos cambiar el nombre del network

> gke.yaml.......linea 15....rshared-net

> network.yaml...linea 9.....rshared-net

creamos la network con 

```bash
make dm-create-shared/compute/network
```

## Despliegue GKE

creamos el despliegue con:

```bash
make dm-create-shared/container/gke
```

Tardaran bastantes minutos en desplegars. Si queremos ver la evolucion podemos realizar click en el icono de la hamburguesa y para abajo casi al final encontramos el Deployment Manager.

## Actualizar network

si queremos actualizar el network:

```bash
make dm-update-shared/compute/network
```

Saldra listado de preview:

```
NAME: compute-sn
TYPE: compute.v1.subnetwork
STATE: IN_PREVIEW
ERRORS: []
INTENT: CREATE_OR_ACQUIRE

NAME: google-managed-services-xxxxx
TYPE: compute.v1.globalAddresses
STATE: IN_PREVIEW
ERRORS: []
INTENT: CREATE_OR_ACQUIRE

NAME: LLLLL
TYPE: compute.v1.subnetwork
STATE: IN_PREVIEW
ERRORS: []
INTENT: CREATE_OR_ACQUIRE

NAME: XXXXXXX
TYPE: compute.v1.subnetwork
STATE: IN_PREVIEW
ERRORS: []
INTENT: CREATE_OR_ACQUIRE

NAME: OOOOOOO
TYPE: compute.v1.network
STATE: IN_PREVIEW
ERRORS: []
INTENT: CREATE_OR_ACQUIRE
```

Saldra una pregunta, si queremos cancelar

`Ready to update, are you sure? Ctrl+C to cancel`

Si queremos continuar pulsamos Intro y si queremos cancelar CTRL + C

## Actualizar GKE
