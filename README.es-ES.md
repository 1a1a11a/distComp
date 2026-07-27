# DistComp: una plataforma de computación distribuida
DistComp es una herramienta sencilla que utiliza el paralelismo de datos para acelerar la computación. Está diseñada para ser fácil de usar. 
La he estado utilizando en Cloudlab para emplear hasta 100 nodos y así acelerar mis experimentos.

![DistComp diagram](/diagram/diagram.svg)

## Arquitectura
DistComp utiliza una arquitectura de gestor-trabajador (manager-worker). El nodo gestor es responsable de la administración, por ejemplo, el envío de nuevas tareas y el monitoreo de las tareas y el estado de los trabajadores. Los nodos trabajadores realizan la computación. 
El diseño actual desacopla el envío de tareas y la ejecución de las mismas. Un gestor envía tareas a la cola de Redis. Los nodos trabajadores consultarán la cola y ejecutarán las tareas.

## Características
* **DistComp maximiza el uso de recursos**: ejecuta tantos trabajos como sea posible en cada nodo. Cuando el uso de memoria de algunas tareas crece con el tiempo y el trabajador está a punto de quedarse sin memoria, la tarea más reciente se devolverá a la cola de tareas pendientes.
* **Envío de tareas bajo demanda**: se pueden enviar nuevas tareas en cualquier momento.
* **Tolerancia a fallos**: los trabajadores reiniciados pueden recuperar sus tareas anteriores para continuar y, si algunos trabajadores fallan, las tareas fallidas pueden moverse a la cola de pendientes.
* **Diferentes tipos de tareas**: DistComp admite tareas de bash y Python.


## Cómo usar
### 0. Preparar los nodos trabajadores
Es necesario preparar los nodos trabajadores para ejecutar las tareas, por ejemplo, instalando las dependencias. 
Además de las dependencias para ejecutar la tarea, los nodos trabajador y gestor deben instalar los paquetes `redis` y `psutil`. 
```bash
pip install redis psutil
```
Y el gestor necesita `parallel-ssh` para lanzar los trabajos. 

Si su tarea requiere datos, los datos deben ser accesibles en los nodos trabajadores.
Puede considerar el uso de un sistema de archivos en clúster, por ejemplo, [MooseFS](https://moosefs.com/), para agregar la capacidad de disco de todos los trabajadores y proporcionar un acceso de alto ancho de banda a los datos. 

### 1. Configurar Redis en el nodo gestor
```bash

bash ./redis.sh
```

### 2. Crear las tareas a ejecutar
Cree un archivo que contenga todas las tareas. Cada línea del archivo consiste en una tarea en el siguiente formato:

```bash
# task type:priority:min_dram:min_cpu:task_params
# echo 'shell:4:2:2:./cachesim PARAM1 PARAM2' >> task

# submit the tasks to the Redis
python3 redisManager.py --task 'initRedis&loadTask' --taskfile task

```

### 3. Iniciar los nodos trabajadores
Utilizamos la herramienta `parallel-ssh` para iniciar los scripts en los nodos trabajadores. La lista de trabajadores (IP o nombre de host) se almacena en el archivo `host`.

```bash

parallel-ssh -h host -i -t 0 '''
    cd /PATH/TO/DistComp;
    screen -S worker -L -Logfile workerScreen/$(hostname) -dm python3 redisWorker.py
'''
```

### 4. Monitorear el progreso
```bash
# check the task status
python3 redisManager.py --task checkTask --finished false --todo false --in_progress false --failed false

# check the worker status
python3 redisManager.py --task checkWorker

# monitor the task progress
watch "python3 redisManager.py --task 'checkTask&checkWorker' --finished false --print_result false --in_progress false"

```
