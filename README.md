# Final-Project-DevOps

DevOps course at URJC Teleco Master.

With provisioning, configuration and orchestration focus.

## Data description

Data is obtained from openFDA. FDA its a Food and Drink Administration in EEUU.
The openFDA its a project indexing and formatting FDA data, to make it
accessible to the public.

Our program, analize "Independent Evaluations of COVID-19 Serological Tests". The Serology tests detect presence os antibodies in the blood, when the body is responding to a specific infection, COVID-19.

The data, from openFDA contains several categories, how for example: Manufacturer, device, data_performed, lot_number, etc.

This program, analyze data file and extract important information, to make graphics, analyze information with more details, etc

## Requirements for the application

## The use cases detected

### Bearing in mind that the web design is oriented towards scientific journalism:
- To show information about the percentage of positive results in the COVID tests carried out.
- Show information about the efficiency of the tests carried out.
- Show information about the efficiency of the tests carried out according to the companies.


## Architecture used to implement the use cases

### The provisional architecture for implementing the use cases is as follows:

![Architecture](/images/Diagram_drawio.png)

# Pasos a seguir --> Examen

### Despliegue de la aplicación usando uvicorn en local

Para este caso simplemente se ejecuta el siguiente código:

    python3 main.py

O si queremos usar directamente uvicorn:

    uvicorn main:app --host 0.0.0.0 --port 8000


### Ejecutar test - unitest en local

Igual que en el anterior caso, para ejecutar los tests:
    
    python3 test.py

### Construir imagen y usar docker para correr la aplicación

Creamos un Dockerfile con la versión de python que queremos y los requisitos que se quieran.
A continuación, en el mismo directorio creamos la imagen docker con el comando:

    docker build --tag covid19garin98:v1 .

Una vez creada la imagen, la corremos mediante el uso del siguiente comando:

    docker run -p 10000:8000 covid19garin98:v1

El significado de esos puertos se corresponde con: el primero de ellos es el
puerto externo que es con el que nos podemos conectar a la aplicación; mientras 
que el segundo es el puerto interno que debe coincidir con el establecido en 
el Dockerfile. Solo se puede conectar en el puerto externo ahora.

Podemos observar las imagenes creadas en el entorno mediante:

    docker images

Para borrar imagenes el siguiente código:

    docker rmi --force identificadorimagen

### Extra: docker swarm en mi propia máquina

Para ver lo que se esta ejecutando en cada máquina es con:

    docker ps

Para levantar los servicios establecidos en docker-compose-swarm.yaml se usa:

    docker-compose -f docker-compose-swarm.yaml up

Para destruir contenedores:

    docker-compose -f docker-compose-swarm.yaml down 

(Se necesita modificar la versión de python a 3.3 para que funcione correctamente)

Para crear un swarm en local:

    docker swarm init --advertise-addr 192.168.1.139

COn el siguiente comando vemos si se ha añadido el nodo:

    docker node ls

Donde la ip se escoge de la propia máquina mediante:
    
    ifconfig

Y despleguemos el servicio con el docker-compose-swarm.yaml:

    docker stack deploy --compose-file docker-compose-swarm-yaml covid19

Compruebo que se han asignado los servicios con el siguiente comando:

    docker ps

Para conectarse debe usarse la IP anterior de ifconfig y el puerto del servicio.

Para eliminar servicios y salir del cluster basta con usar:

    docker swarm leave --force

### Docker swarm (ansible-playbook) para crear cluster, despligue y destruir mediante TERRAFORM

Nos creamos un docker-compose-swarm para crear un cluster con nodos.
En este archivo, se añade el nombre de la imagen de docker, el path donde
se va a realizar dicha imagen, los puertos y el número de réplicas que queremos 
en el cluster. Estas réplicas se repartirán entre las instancias y es el propio
swarm el que balanceará la carga. Aqui entra AWS. Los pasos son los siguientes:

Primero exportamos las variables de entorno correspondientes con amazon web services:

    source ../../rootkey.sh

Una vez exportadas, usamos terraform para para desplegar la infraestructura:

    cd terraform/terraform_aws/
    ../../../../Descargas/terraform_1.0.9_linux_amd64/terraform init
    ../../../../Descargas/terraform_1.0.9_linux_amd64/terraform plan
    ../../../../Descargas/terraform_1.0.9_linux_amd64/terraform apply

Con esto ya tenemos desplegada la infraestructura en AWS. A continuación,
copiamos las direcciones que nos devuelve terraform al fichero hosts.ini:

    ../../../../Descargas/terraform_1.0.9_linux_amd64/terraform output | grep '"' | awk -F '"' 'BEGIN {i=1; print "[cluster]"} {print "node0" i++ " ansible_host="$2}' > ../ansible/swarm/hosts.ini

Cambiamos a la carpeta de ansible donde vamos a crear el cluster y desplegar los servicios:

    cd ../..
    cd terraform/ansible/swarm

Y despluegamos ambos:

    ansible-playbook -u ubuntu -i hosts.ini create_swarm.yml 

En caso de que de error debemos abrir un nuevo terminal y conectar a mano por ssh a las
direcciones IP que se encuentran en hosts.ini:

    ssh ubuntu@direccionIP

Ejecutamos de nuevo el create cluster:

    ansible-playbook -u ubuntu -i hosts.ini create_swarm.yml 

Y ahora desplegamos a la aplicación:

    ansible-playbook -u ubuntu -i hosts.ini deploy_app.yml

Una vez funcionando podemos acceder a traves del navegador a cualquiera
de las IP y con los puertos adecuados ver lo que se desea:

    - 40001: Aplicación
    - 9090: Prometheus
    - 3000: Grafana
    - 9100: Node explorer
    - 8080: cAdvisor

Para finalizar el trabajo, eliminamos el cluster en el mismo directorio
en el que nos encontramos ejecutando:

    ansible-playbook -u ubuntu -i hosts.ini destroy_swarm.yml 

Volvemos a terraform AWS (directorio) y eliminamos las instancias y la 
infra generada en AWS.

    cd ../../
    cd terraform_aws/
    ../../../../Descargas/terraform_1.0.9_linux_amd64/terraform destroy


### Acceso a prometheus y cAdvisor

Comentado anteriormente usando los puertos descritos.

### Despligue automático mediante acciones de github (BONUS - low)

Primero se debe crear el backet s3 donde se guardará el estado. Esto
solo se realiza una vez en el proyecto desde el directorio de remote_state
lanzando los debidos terraform init, plan y apply.

Entonces los pasos son los siguientes para el correcto funcionamiento del
pipeline automáticos:

Primero creamos el cluster mediante acciones, el codigo correspondiente
se encuentra en create_cluster.sh. En el se indica el número de nodos (instancias)
que deseamos y a continuación realiza todas las tareas anteriores:
- Instala terraform
- Terraform init 
- Terraform plan 
- Terraform apply
- Copia las IP a hosts.ini
- Entra en cada maquina por ssh desencriptando la vault_pass que es la clave privada
- Y por último, usa ansible playbook para desplegar el cluster

Una vez creado el cluster, se debe realizar un PUSH al repositorio para 
activar la accion que despliga la aplicación en cada nodo del cluster.

Con esto funcionando, podemos acceder de nuevo a las máquinas viendo las 
IP en el propio flujo de trabajo y de nuevo acceder a graphana, prometheus, etc.

Como siempre, se debe eliminar el cluster y destruir las instancias. Para 
ello, activamos la acción de destroy y finalizamos con todo.

### Visualización de métricas con grafana (BONUS - medium)