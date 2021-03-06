
Docker - Tutorial Completo

=======> CONTAINER:
https://docs.docker.com/get-started/part2/#apppy

	1. creamos una app web en python y la dockerizamos con Dockerfile
		
		# Use an official Python runtime as a parent image
		FROM python:2.7-slim

		# Set the working directory to /app
		WORKDIR /app

		# Copy the current directory contents into the container at /app
		ADD . /app

		# Install any needed packages specified in requirements.txt
		RUN pip install --trusted-host pypi.python.org -r requirements.txt

		# Make port 80 available to the world outside this container
		EXPOSE 80

		# Define environment variable
		ENV NAME World

		# Run app.py when the container launches
		CMD ["python", "app.py"]

	2. buildeamos la imagen 
		docker build -t friendlyhello .

	3. verificamos la imagen creada
		docker image ls	

	4. levantamos la imagen en un container => mapping your machine’s port 4000 to the container’s published port 80 using -p
		4.1. docker run -p 4000:80 friendlyhello	

			curl http://localhost:4000

		o bien levantamos en background

		4.2. docker run -d -p 4000:80 friendlyhello 

	5. verificamos los containers corriendo

		docker containers ls

	6. paramos el container
		
		docker container stop <CONTAINER_ID>

	7. subimos la imagen a docker registry para compartirla
	
		7.1 sacamos una cta en docker hub
			Docker Propio (dockerhub)	
				https://hub.docker.com/r/damianciocca/examples/
				ID damianciocca	
				pass racing90	

		7.2 nos logueamos a docker hub		

			docker login

			Nota: The notation for associating a local image with a repository on a registry is username/repository:tag. The tag is optional, but recommended, since it is the mechanism that registries use to give Docker images a version

			Creamos un repository con el nombre "examples"

		7.2 tagueamos la imagen creada 
			Ej: docker tag image username/repository:tag
			
			Verificamos q la imagen este creada con el nombre friendlyhello
			docker images
				REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
				friendlyhello          latest              265482889e93        20 minutes ago      132MB

			docker tag friendlyhello damianciocca/examples:part2	

			docker images
				REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
				damianciocca/examples   part2               265482889e93        24 minutes ago      132MB

		7.3 publicamos la imagen a docker hub

			docker push damianciocca/examples:part2

		7.3 pull & run 
		
			docker run -p 4000:80 damianciocca/examples:part2	

=======> SERVICES (en un nodo):
https://docs.docker.com/get-started/part3/#prerequisites

	Nota: Services are really just “containers in production.” A service only runs one image,

		1. Creamos un stack de servicios: stack.yml (esto es el docker-compose)
			version: "3"
				services:
				  web:
				    # replace username/repo:tag with your name and image details
				    image: damianciocca/examples:part2
				    deploy:
				      replicas: 5
				      resources:
				        limits:
				          cpus: "0.1"
				          memory: 50M
				      restart_policy:
				        condition: on-failure
				    ports:
				      - "80:80"
				    networks:
				      - webnet
				networks:
				  webnet:

			This docker-compose.yml file tells Docker to do the following:

			Pull the image we uploaded in step 2 from the registry.

			Run 5 instances of that image as a service called web, limiting each one to use, at most, 10% of the CPU (across all cores), and 50MB of RAM.

			Immediately restart containers if one fails.

			Map port 80 on the host to web’s port 80.

			Instruct web’s containers to share port 80 via a load-balanced network called webnet. (Internally, the containers themselves publish to web’s port 80 at an ephemeral port.)

			Define the webnet network with the default settings (which is a load-balanced overlay network).

			Importante: Es ver que estas 5 replicas tienen un load balancer interno q distribuye los pedidos aca cada replica

	  	2. Inicializamos docker swarm para poder deployar el docker compose previo (stack.yml)
			
			docker swarm init	

		3. deployamos el stack de servicios (docker compose) en el docker swarm inicializado previamente
		
			docker stack deploy -c stack.yml getstartedlab	

				* Creating network getstartedlab_webnet
				* Creating service getstartedlab_web

				Our single service stack is running 5 container instances of our deployed image on one host

		4. verificamos el stack deployado

			docker stack ls

				NAME                SERVICES
				getstartedlab       1

		4. verificamos los servicios activos de los stacks
		
			docker service ls 

			ID                  NAME                MODE                REPLICAS            IMAGE                         PORTS
			eg8a79wg1gfu        getstartedlab_web   replicated          5/5                 damianciocca/examples:part2   *:80->80/tcp
	
		5. verificamos los logs

			docker service logs getstartedlab_web --follow	

		6. verificamos los procesos/tareas/tasks de un servicio
		
			docker service ps getstartedlab_web	

			ID                  NAME                  IMAGE                         NODE                DESIRED STATE       CURRENT STATE ix9pnwp6lu71        getstartedlab_web.1   damianciocca/examples:part2   moby                Running             Running 6 minutes ago
			4m75crg2cn87        getstartedlab_web.2   damianciocca/examples:part2   moby                Running             Running 6 minutes ago
			qsbyoe7gyg04        getstartedlab_web.3   damianciocca/examples:part2   moby                Running             Running 6 minutes ago
			w1wb0c5ilruy        getstartedlab_web.4   damianciocca/examples:part2   moby                Running             Running 6 minutes ago
			hmllw5tib91n        getstartedlab_web.5   damianciocca/examples:part2   moby                Running             Running 6 minutes ago

		7. 	Verificamos que cada pedido q hagamos una de las replicas va a responder

			Either way, the container ID changes, demonstrating the load-balancing; with each request, one of the 5 tasks is chosen, in a round-robin fashion, to respond. The container IDs match your output from the previous command (docker container ls -q).

		8.  Agregamos mas replicas	

			You can scale the app by changing the replicas value in stack.yml, saving the change, and re-running the docker stack deploy command:

				docker stack deploy -c stack.yml getstartedlab

		9. removemos el stack (swarm) y por ende, se bajaran los servicios (en este caso dos) y de cada servicio de eliminarn los procesos/tasks (en este caso 5 replicas)

				docker stack rm getstartedlab

					Removing service getstartedlab_web
					Removing network getstartedlab_webnet		

				o bien
				
				docker swarm leave --force

					Node left the swarm.

=======> CLUSTER/SWARM (1 servicio en varios nodos - diferentes machines)
https://docs.docker.com/get-started/part4/		

	When Docker is running in swarm mode, you can still run standalone containers on any of the Docker hosts participating in the swarm, as well as swarm services. A key difference between standalone containers and swarm services is that only swarm managers can manage a swarm, while standalone containers can be started on any daemon.

	In the same way that you can use Docker Compose to define and run containers, you can define and run swarm service stacks.

	Docker Machine 

		1. Overview
			https://docs.docker.com/machine/overview/
			
			* Docker Machine is a tool that lets you install Docker Engine on virtual hosts, and manage the hosts with docker-machine commands
			* Using docker-machine commands, you can start, inspect, stop, and restart a managed host
			
			**  Docker Engine accepts docker commands from the CLI, such as docker run <image>, docker ps, etc
			**  Docker Machine is a tool for provisioning and managing your Dockerized hosts (hosts with Docker Engine on them)

			Point the Machine CLI at a running, managed host, and you can run docker commands directly on that host:
			=> docker-machine env default

		2. Instalamos el comando docker-machine	
			https://docs.docker.com/machine/install-machine/#install-machine-directly
			
			* instalamos Virtual Box en Mac para crear virtualizaciones de host

			bash

			base=https://github.com/docker/machine/releases/download/v0.14.0 && curl -L $base/docker-machine-$(uname -s)-$(uname -m) >/usr/local/bin/docker-machine && chmod +x /usr/local/bin/docker-machine	

		3. Instalamos Virtual Box para virtualizacion de host 
			https://www.virtualbox.org/wiki/Downloads

		4. Operamos con docker-machine	
		https://docs.docker.com/machine/get-started/#use-machine-to-run-docker-containers

		4.1. crear una machine (host virtual con docker instalado)	
  			
  			docker-machine create --driver virtualbox default

			Note: pass the appropriate driver to the --driver flag and provide a machine name "default"
		
		4.2. listar las machines availables
			
			docker-machine ls
			
			NAME      ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
			default   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.05.0-ce

		4.3. obtener las variables de entorno de la nueva VM (machine)
			
			docker-machine env default	

				set -gx DOCKER_TLS_VERIFY "1";
				set -gx DOCKER_HOST "tcp://192.168.99.100:2376";
				set -gx DOCKER_CERT_PATH "/Users/damian/.docker/machine/machines/default";
				set -gx DOCKER_MACHINE_NAME "default";
				# Run this command to configure your shell:
				# eval (docker-machine env default)

			o bien si estamos en fish
			docker-machine env --shell fish default	

		4.4. conectar nuestro shell (el ciente de docker) a la nueva VM (machine)			
			
			eval "$(docker-machine env default)"
			
			o bien si estamos en fish
			eval (docker-machine env --shell fish default)

		4.5. obtener la IP de la nueva VM
			
			docker-machine ip default	

		4.6. levantamos un webserver NGINX
		
			docker run -d -p 8000:80 nginx	

			http://192.168.99.100:8000/
			note: accedemos con la IP q se nos asigno cuando levantamos la machine

		4.7. paramos y volvemos a levantar las VM
		
			docker-machine stop		
			NAME      ACTIVE   DRIVER       STATE     URL   SWARM   DOCKER    ERRORS
			default   -        virtualbox   Stopped                 Unknown

			docker-machine start	

		4.8. para desconectar nuestro cliente de docker de la VM

			docker-machine env -u
			docker-machine env -u --shell fish

				set -e DOCKER_TLS_VERIFY;
				set -e DOCKER_HOST;
				set -e DOCKER_CERT_PATH;
				set -e DOCKER_MACHINE_NAME;
				# Run this command to configure your shell:
				# eval (docker-machine env -u --shell fish)	

			eval (docker-machine env -u --shell fish)
			
		4.9. eliminamos la VM
			
			docker-machine rm default		

	SWARM
	
		Multi-container, multi-machine applications are made possible by joining multiple machines into a “Dockerized” cluster called a swarm.					
		A swarm is a group of machines that are running Docker engine and joined into a cluster

		Swarm managers are the only machines in a swarm that can execute your commands, or authorize other machines to join the swarm as workers

		A swarm is made up of multiple nodes, which can be either physical or virtual machines.

		The basic concept is simple enough: 
			run docker swarm init to enable swarm mode and make your current machine a swarm manager, then run docker swarm join on other machines to have them join the swarm as workers


		1. Creando un cluster. create a couple of VMs using docker-machine, using the VirtualBox driver:	

			docker-machine create --driver virtualbox myvm1
			docker-machine create --driver virtualbox myvm2

			Note: Esto es lo q hariamos con AWS creando dos instancias de EC2

		2. Listamos las VMs
		
			docker-machine ls
		
			NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
			myvm1   -        virtualbox   Running   tcp://192.168.99.100:2376           v18.05.0-ce
			myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.05.0-ce	


		3. Inicializamos el SWARM
			The first machine acts as the manager, which executes management commands and authenticates workers to join the swarm, and the second is a worker.

		4. Abrimos en dos consolas diferentes y linkeamos el cliente de docker (docker) a cada VM
			
			docker-machine env --shell fish myvm1
			docker-machine env --shell fish myvm2	

		5. Inicializamos en myvm1 el manager 
			
			docker swarm init --advertise-addr 192.168.99.100

				Swarm initialized: current node (ix6ml4dxsj37ku04w34exojru) is now a manager.

				To add a worker to this swarm, run the following command:

				    docker swarm join --token SWMTKN-1-4ljav6drai86zy0wq4ty3ji1e3b31mvx63krntq1cdj78k4qh6-bk4h356q27il162u8iyqgwitw 192.168.99.100:2377

				To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

		6. Agregamos un worker al manager creado previamente. Vamos a la VM myvm2 y lanzamos el siguiente comando

			docker swarm join --token SWMTKN-1-4ljav6drai86zy0wq4ty3ji1e3b31mvx63krntq1cdj78k4qh6-bk4h356q27il162u8iyqgwitw 192.168.99.100:2377

				This node joined a swarm as a worker.

			Congratulations, you have created your first swarm!
		
		7. Verificamos el estado del Swarm desde el manager ( No desde el worker!)
		
			docker node ls	

				ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
				ix6ml4dxsj37ku04w34exojru *   myvm1               Ready               Active              Leader
				x8vnjtagzdoyq4md464l0p2rf     myvm2               Ready               Active


			Importante: Remember that only swarm managers like myvm1 execute Docker commands; workers are just for capacity.
			
		8. Deploy the app on the swarm manager: desde el manager -> VM "myvm1". Nos paramos en el proyecto que tiene el stack.yml
			
			docker stack deploy -c stack.yml getstartedlab

				Note: If your image is stored on a private registry instead of Docker Hub, you need to be logged in using docker login <your-registry> and then you need to add the --with-registry-auth flag to the above command.

				Esto puede tardar unos minutos.. 

		9. Verificamos el estado de los servicios del deploy previo. Siempre desde el manager (myvm1)
		
			docker service ls				

				ID                  NAME                MODE                REPLICAS            IMAGE                         PORTS
				ujhnr46utaml        getstartedlab_web   replicated          8/8                 damianciocca/examples:part2   *:80->80/tc

				Nota: obsercamos las REPLICAS q estan 8 de 8 running.

			docker stack ps getstartedlab
			
				ID                  NAME                  IMAGE                         NODE                DESIRED STATE       CURRENT STATE           ERROR               PORTS
				xfbmnqskl8jc        getstartedlab_web.1   damianciocca/examples:part2   myvm1               Running             Running 4 minutes ago
				5l4i6syk8hi5        getstartedlab_web.2   damianciocca/examples:part2   myvm2               Running             Running 4 minutes ago
				1b20r4wt58q8        getstartedlab_web.3   damianciocca/examples:part2   myvm1               Running             Running 4 minutes ago
				3quubapa4yo5        getstartedlab_web.4   damianciocca/examples:part2   myvm2               Running             Running 4 minutes ago
				ianomm2p45s6        getstartedlab_web.5   damianciocca/examples:part2   myvm1               Running             Running 4 minutes ago
				32665eid9wwm        getstartedlab_web.6   damianciocca/examples:part2   myvm2               Running             Running 4 minutes ago
				xylhnzvptz0l        getstartedlab_web.7   damianciocca/examples:part2   myvm1               Running             Running 4 minutes ago
				u41l0qfqpr6p        getstartedlab_web.8   damianciocca/examples:part2   myvm2               Running             Running 4 minutes ago	
				
				Note: the services and associated containers have been distributed between both myvm1 and myvm2.

			docker ps

				CONTAINER ID        IMAGE                         COMMAND             CREATED             STATUS              PORTS               NAMES
				a986d84e3029        damianciocca/examples:part2   "python app.py"     2 minutes ago       Up 2 minutes        80/tcp              getstartedlab_web.3.1b20r4wt58q8epzcquiknlkxr
				3f274ef64b04        damianciocca/examples:part2   "python app.py"     2 minutes ago       Up 2 minutes        80/tcp              getstartedlab_web.7.xylhnzvptz0lc2po9d9sk09r3
				dd110605632d        damianciocca/examples:part2   "python app.py"     2 minutes ago       Up 2 minutes        80/tcp              getstartedlab_web.1.xfbmnqskl8jcsnnx5zc3wvf5k
				df1c389e8fa5        damianciocca/examples:part2   "python app.py"     2 minutes ago       Up 2 minutes        80/tcp              getstartedlab_web.5.ianomm2p45s6gtyti5d45looo

		10. Verificamos los containers levantados en el worker (myvm2)

			docker ps

				CONTAINER ID        IMAGE                         COMMAND             CREATED             STATUS              PORTS               NAMES
				b5d943352402        damianciocca/examples:part2   "python app.py"     43 seconds ago      Up 41 seconds       80/tcp              getstartedlab_web.2.5l4i6syk8hi52dj07fnbspk2b
				2fcd04e35baa        damianciocca/examples:part2   "python app.py"     43 seconds ago      Up 41 seconds       80/tcp              getstartedlab_web.6.32665eid9wwm11mwzkyppenxj
				840a149eda33        damianciocca/examples:part2   "python app.py"     43 seconds ago      Up 41 seconds       80/tcp              getstartedlab_web.8.u41l0qfqpr6phodav3daji15d
				ec1151803506        damianciocca/examples:part2   "python app.py"     43 seconds ago      Up 41 seconds       80/tcp              getstartedlab_web.4.3quubapa4yo5hw5usfg77jnkg

			Verificamos q las 8 replicas estan deployadas 4 en el manager y las otras 4 en el worker

			You can access your app from the IP address of either myvm1 or myvm2.

			The network you created in stack.yml is shared between them and load-balancing. 

			Desde afuera le pegamos a una IP o a otra IP e internamente va a redirigir la peticion a una de las 4 replicas q tiene levantada (servicio/container). Internamente va a distribuir la carga a cada app web

		11. jugamos levantando y bajando alguna de las maquinas (en este caso bajamos el worker)
		
				docker-machine stop myvm2	

					nota: al bajar la instancia myvm2 vemos q las 4 replicas se distribuyeron a myvm1. Ahora myvm1 tiene las 8 replicas

					y desde myvm1 ejecutamos: docker stack ps getstartedlab
				
				ID                  NAME                      IMAGE                         NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
				ko7b1leh6640        getstartedlab_web.1       damianciocca/examples:part2   myvm1               Running             Running 8 minutes ago
				xyq7l9b3sfg5         \_ getstartedlab_web.1   damianciocca/examples:part2   myvm2               Shutdown            Shutdown 6 minutes ago
				fpt4btf6oe1i        getstartedlab_web.2       damianciocca/examples:part2   myvm1               Running             Running 11 minutes ago
				8fue6l6zegan        getstartedlab_web.3       damianciocca/examples:part2   myvm1               Running             Running 8 minutes ago
				1m93qpwuioh5         \_ getstartedlab_web.3   damianciocca/examples:part2   myvm2               Shutdown            Shutdown 6 minutes ago
				be0ad3k5cesa        getstartedlab_web.4       damianciocca/examples:part2   myvm1               Running             Running 11 minutes ago
				04zvh6lebmzo        getstartedlab_web.5       damianciocca/examples:part2   myvm1               Running             Running 8 minutes ago
				0i85301xv27v         \_ getstartedlab_web.5   damianciocca/examples:part2   myvm2               Shutdown            Shutdown 6 minutes ago
				vxg8jdbfewrm        getstartedlab_web.6       damianciocca/examples:part2   myvm1               Running             Running 11 minutes ago
				ykxd22r69cqz        getstartedlab_web.7       damianciocca/examples:part2   myvm1               Running             Running 8 minutes ago
				tzikm1bk83kz         \_ getstartedlab_web.7   damianciocca/examples:part2   myvm2               Shutdown            Shutdown 6 minutes ago
				ja1wyfl7day1        getstartedlab_web.8       damianciocca/examples:part2   myvm1               Running             Running 11 minutes ago

				docker-machine start myvm2

					nota: al volver a subir la instancia myvm2, vemos q no se vuelven a re-balancear las replicas en myvm2 

		12. Ahora hacemos lo mismo q el punto 11 pero al reves. Bajamos primero el manager (myvm1)
		
				docker-machine stop myvm1
			
					nota: al bajar la instancia myvm1 (manager) vemos q las 4 replicas NO se distribuyeron a myvm2. Solo nos quedamos con 4 replicas

				docker-machine start myvm1
				
				y desde myvm1 ejecutamos: docker stack ps getstartedlab

				ID                  NAME                      IMAGE                         NODE                DESIRED STATE       CURRENT STATE            ERROR                              PORTS
				9oh15wvl7sm5        getstartedlab_web.1       damianciocca/examples:part2   myvm1               Running             Running 24 seconds ago
				h8ky5j7sl1qy         \_ getstartedlab_web.1   damianciocca/examples:part2   myvm1               Shutdown            Failed 30 seconds ago    "No such container: getstarted…"
				v2b3zen9xw13        getstartedlab_web.2       damianciocca/examples:part2   myvm2               Running             Running 25 seconds ago
				3ipbxfwpl9y7        getstartedlab_web.3       damianciocca/examples:part2   myvm1               Running             Running 24 seconds ago
				ub8b0yxzsaeo         \_ getstartedlab_web.3   damianciocca/examples:part2   myvm1               Shutdown            Failed 30 seconds ago    "No such container: getstarted…"
				ypp0b4vwxkvq        getstartedlab_web.4       damianciocca/examples:part2   myvm2               Running             Running 25 seconds ago
				sf4jsq1bwit0        getstartedlab_web.5       damianciocca/examples:part2   myvm1               Running             Running 24 seconds ago
				snywl85hz87j         \_ getstartedlab_web.5   damianciocca/examples:part2   myvm1               Shutdown            Failed 30 seconds ago    "No such container: getstarted…"
				m5h5z5d2avhd        getstartedlab_web.6       damianciocca/examples:part2   myvm2               Running             Running 25 seconds ago
				faayavhmvi6f        getstartedlab_web.7       damianciocca/examples:part2   myvm1               Running             Running 24 seconds ago
				fs7hpsib9wu7         \_ getstartedlab_web.7   damianciocca/examples:part2   myvm1               Shutdown            Failed 30 seconds ago    "No such container: getstarted…"
				vc8xl4rbac3u        getstartedlab_web.8       damianciocca/examples:part2   myvm2               Running             Running 25 seconds ago
	
				IMPORTANTE. Aca vemos q se volvieron a restaurar las 4 replicas del manager

			CONCLUSION
			
				Si se cae el worker, las replicas se pasan al manager (excepto q se lo especifique q la replica no puede correr en el manager)
				Si se vuelve a levantar el worker, las replicas no se vuelven a restaurar automaticamente. Quedan todas en el manager
					En este escenario queda desbalanceado el swarm.

				Si se cae el manager, las replicas NO del manager NO pasan al worker
				Si se levanta el manager, se vuelven a restauran las replicas, quedando tanto el worker como el manager con sus replicas correspondientes.
					En este caso, queda todo perfectamente balanceado.	

				En el stack.yml tenemos configurado para el servicio web:
					restart_policy: condition: on-failure	

=======> CLUSTER/SWARM/STACK (varios servicios en varios nodos - diferentes machines)
https://docs.docker.com/get-started/part5/#add-a-new-service-and-redeploy

		1. Nos aseguramos q tenemos las dos maquinas levantandas (myvm1 y myvm2)
		
		2. Que las 8 replicas esten running

		3. VISUZALIZER: Agregamos un nuevo servicio al stack. En este ej agregamos ademas del servicio web el servicio visualizer solo en el manager (no 	en el worker)

		4. redeployamos nuevamente el stack
			docker stack deploy -c stack.yml getstartedlab

				Updating service getstartedlab_web (id: bkgfjuzrgx2vm4a5iga5bo11x)
				Creating service getstartedlab_visualizer

		5. verificamos los servicios activos desde el manager
			
			docker service ls
				ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
				46dtqlwuq20c        getstartedlab_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
				bkgfjuzrgx2v        getstartedlab_web          replicated          8/8                 damianciocca/examples:part2       *:80->80/tcp	

		6. abrimos el visualizer desde el browser (con la IP del manager... myvm1)
			
			http://192.168.99.100:8080/	

			The single copy of visualizer is running on the manager as you expect, and the 5 instances of web are spread out across the swarm. You can corroborate this visualization by running docker stack ps getstartedlab

		7. REDIS: Agregamos un nuevo servicio	
			
			para este servicio tenemos q crear un directorio en el manager (myvm1)
			docker-machine ssh myvm1 "mkdir ./data"

		8. redeployamos el stack
			docker stack deploy -c stack.yml getstartedlab

				Updating service getstartedlab_visualizer (id: 46dtqlwuq20c504tnkm8vqpoy)
				Creating service getstartedlab_redis
				Updating service getstartedlab_web (id: bkgfjuzrgx2vm4a5iga5bo11x)	

			docker service ls
			
				ID                  NAME                       MODE                REPLICAS            IMAGE                             PORTS
				46dtqlwuq20c        getstartedlab_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8080->8080/tcp
				bkgfjuzrgx2v        getstartedlab_web          replicated          8/8                 damianciocca/examples:part2       *:80->80/tcp
				n1eyvg3kp7q3        getstartedlab_redis        replicated          1/1                 redis:latest                      *:6379->6379/tcp
	




Probando traefic - stack-2.yaml (3 instancias de traefic y 3 instancias de API web)

	docker stack deploy -c stack-2.yml  webnet

	Traefic -> http://192.168.99.100:8080/dashboard/
	
		Api:
			http://192.168.99.100/core
			http://192.168.99.101/core
			curl http://192.168.99.100/core

	Visualizer -> http://192.168.99.100:8081

	docker service ls

	damian@Damians-MacBook-Pro ~/D/w/docker-web-example> docker service ls
	ID                  NAME                MODE                REPLICAS            IMAGE                             PORTS
	1jie2l4d8olf        webnet_visualizer   replicated          1/1                 dockersamples/visualizer:stable   *:8081->8080/tcp
	eioy6qgox68u        webnet_web          replicated          3/3                 damianciocca/examples:part2
	f4rrhi9psrvz        webnet_traefik      replicated          3/3                 traefik:latest                    *:80->80/tcp,*:443->443/tcp,*:8080->8080/tcp


Probando traefic - saga.yaml (1 instancia de traefic y 1 instancias de API web)

	Ahora la api esta expuesta en el puerto 9090 (ver dockerfile)

	En el stack saga.yml se pone el puerto 9090 como puerto interno de la api

	Ahora tenemos dos endpoints => uno q atiende en / y otra en /hello (ambos get)
		http://192.168.99.100/core
		http://192.168.99.100/core/hello

		Sino ponemos "core" traefik no va a redirijir el request a la api 

	Y como vemos el label "core" se reemplaza por "" segun la regla q pusimos en el stack
	 	- "traefik.frontend.rule=PathPrefixStrip:/core"	

	De esta manera todo lo q pontamos como core va a ir a:
	
		core:
		    image: damianciocca/examples:part4
		    ports:
		    - '9090'
		    networks:
		    - saga
		    deploy:
		      mode: replicated
		      replicas: 1
		      labels:
		        - "traefik.backend=restapi"
		        - "traefik.enable=true"
		        - "traefik.port=9090"
		        - "traefik.docker.network=saga_saga"
		        - "traefik.frontend.rule=PathPrefixStrip:/core"
		        #- "traefik.frontend.rule=Host:api.coolapp.com"
		      update_config:
		        parallelism: 1


=======> ESCALING (3 host. 1 manager y 2 worker)
https://docs.docker.com/engine/swarm/swarm-tutorial/

1. creamos una nueva docker-machine (myvm3)

	docker-machine ls
		NAME    ACTIVE   DRIVER       STATE     URL                         SWARM   DOCKER        ERRORS
		myvm1   *        virtualbox   Running   tcp://192.168.99.100:2376           v18.05.0-ce
		myvm2   -        virtualbox   Running   tcp://192.168.99.101:2376           v18.05.0-ce
		myvm3   -        virtualbox   Running   tcp://192.168.99.102:2376           v18.05.0-ce

2. unimos al swarm existente, un nuevo host (myvm3)

	desde el manager, obtenemos el token
		docker swarm join-token worker

	entramos al nuevo host (worker) myvm3
		    docker swarm join --token SWMTKN-1-52jc4ek41g5d2q6r9yxfe9kbo9zck5vluatr9pvg0p7535wvbc-7uvoniz6nkrsrwkixg6pjmfm1 192.168.99.100:2377

	despues de varios minutos entramos al visualizer
		http://192.168.99.100:8000/

		Nota: vemos q todos los servicios q no tengan alguna restriccion o q NO se haya alcanzado el numero de replicas van a aparecer en el nuevo nodo

	IMPORTANTE
		
		Si queremos unir otro manager en lugar de un worker
			docker swarm join-token manager	

		entramos al nuevo host (manager) myvm4 y copiamos y pegamos el comando de join.		

3. inspeccionamos cada uno de los host (el manager y los dos worker)

	docker info

4. inspeccionamos cada uno de los servicios

	docker service inspect --pretty saga_core

5. Once you have deployed a service to a swarm, you are ready to use the Docker CLI to scale the number of containers in the service. 
	Containers running in a service are called “tasks.”		

	En caso que tengamos solo 1 servicio saga_core => agregamos uno mas y vemos q se deberia deployar en el nuevo host

		docker service scale saga_core=2
			saga_core scaled to 2

	Verificamos los task(containers) del servicio saga_core
	
		docker service ps saga_core

		ID                  NAME                IMAGE                         NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
		7qctrtrt81m6        saga_core.1         damianciocca/examples:part4   myvm2               Running             Running about an hour ago
		tqpbalvlemr0         \_ saga_core.1     damianciocca/examples:part4   myvm1               Shutdown            Shutdown about an hour ago
		ruiuuytys7nt         \_ saga_core.1     damianciocca/examples:part4   myvm2               Shutdown            Shutdown about an hour ago
		sa27t6wkju8u         \_ saga_core.1     damianciocca/examples:part2   myvm2               Shutdown            Shutdown 3 days ago
		5pw1nautw6f7        saga_core.2         damianciocca/examples:part4   myvm3               Running             Running 40 seconds ago		


	Entramos en cada worker (myvm2 y myvm3)
	
		myvm2
		docker logs 6bcd26869084 --follow

		myvm3
		docker logs bfbd9bda6561 --follow	

	Desde el manager o otra terminal
	
		curl http://192.168.99.100/core
		curl http://192.168.99.101/core
		curl http://192.168.99.102/core

		Nota: le pegue a la IP q le pegue, puede responder la ip del myvm2 o myvm3


6. Drain a node on the swarm (inhabilitandolo para que el manager no lo tenga en cuenta) 
	NOTA: Siempre lo ejecutamos desde el manager
	
	docker node ls
		ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
		mpfzcpphfh32yt2unwoar82go     myvm3               Ready               Active
		wrrnqvvi9m7dy8bpl86zo9ob2 *   myvm1               Ready               Active              Leader
		z9drsmeaio3yahvohch6xdjat     myvm2               Ready               Active

	docker node update --availability drain myvm2
		
	docker node ls
		ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
		mpfzcpphfh32yt2unwoar82go     myvm3               Ready               Active
		wrrnqvvi9m7dy8bpl86zo9ob2 *   myvm1               Ready               Active              Leader
		z9drsmeaio3yahvohch6xdjat     myvm2               Ready               Drain

	Por visualizer podemos ver como las replicas que estaban en myvm2 pasaron a myvm3

	Si deployamos nuevamente el stack, no se van a deployar en el nodo myvm2 AVAILABILITY = Drain

7. volvemos a activar el nodo

	docker node update --availability active myvm2	

	docker node ls
		ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS
		mpfzcpphfh32yt2unwoar82go     myvm3               Ready               Active
		wrrnqvvi9m7dy8bpl86zo9ob2 *   myvm1               Ready               Active              Leader
		z9drsmeaio3yahvohch6xdjat     myvm2               Ready               Active


=======> ADMIN SWARM
https://docs.docker.com/engine/swarm/admin_guide/
https://docs.docker.com/engine/swarm/how-swarm-mode-works/nodes/
https://docs.docker.com/engine/swarm/how-swarm-mode-works/services/
https://docs.docker.com/engine/swarm/swarm-mode/

	docker stats

	docker stats <container-id/container-name>

	IMPORTANTE:
	When initiating a swarm, you must specify the --advertise-addr flag to advertise your address to other manager nodes in the swarm. For more information, see Run Docker Engine in swarm mode. Because manager nodes are meant to be a stable component of the infrastructure, you should use a fixed IP address for the advertise address to prevent the swarm from becoming unstable on machine reboot.

	If the whole swarm restarts and every manager node subsequently gets a new IP address, there is no way for any node to contact an existing manager. Therefore the swarm is hung while nodes try to contact one another at their old IP addresses.

	Dynamic IP addresses are OK for worker nodes
