# Diagrama de Red

![diagrama de red](https://github.com/klartz/prueba-tecnica-craftech/blob/main/images/network-diagram.png)

## Componentes/servicios utilizados

1. **Frontend**
    1. **Route 53**: Gestiona la resolución de nombres de dominio.
    2. **S3 Bucket**: Almacena los archivos estáticos del frontend.
    3. **CloudFront (CDN)**: Almacena en caché y distribuye los assets estáticos del frontend.
2. **Backend**
    1. **Internet Gateway**: Permite la comunicación entre la VPC e internet.
    2. **Application Load Balancer (ALB)**: Recibe solicitudes entrantes y distribuye la carga entre las instancias, garantizando balanceo de tráfico y alta disponibilidad.
    3. **EC2 Auto Scaling Group**: Contiene y escala instancias automáticamente según la demanda.
    4. **NAT Gateway**: Permite que las instancias privadas accedan a internet de forma segura.
3. **Bases de Datos**
    1. **Amazon RDS**: Base de datos relacional.
    2. **Amazon DynamoDB**: Base de datos no relacional.
    3. **Réplicas**: Se mantienen réplicas tanto para ambas bases de datos, asegurando alta disponibilidad y redundancia.

---

El diagrama representa una arquitectura de tres capas:

- **Frontend**: ubicado en una red pública.
- **Backend**: alojado en una red privada.
- **Bases de datos**: también en una red privada.

La aplicación está desplegada en una única región de AWS, pero distribuida en múltiples *Availability Zones* (AZ) para garantizar alta disponibilidad. Esto permite que si una AZ sufre una falla crítica, la aplicación siga funcionando.
Opté por utilizar una única región por simplicidad, pero es posible extender la arquitectura a múltiples regiones. En tal caso, CloudFront se encargaría de enrutar el tráfico entre ellas.

Las peticiones de los clientes son recibidas y gestionadas por Amazon Route 53. CloudFront se encarga de distribuir los assets desde S3 según la ubicación geográfica del cliente, mejorando los tiempos de respuesta.

El frontend se encuentra en una red pública para poder comunicarse con internet, mientras que tanto el backend como las bases de datos se encuentran en redes privadas para evitar la comunicación directa con internet.
Por esta razón el backend interactúa con los microservicios externos a través de un NAT Gateway en la subred pública, asegurando una comunicación segura.

Dentro de la VPC, las peticiones pasan por un Elastic Load Balancer (ELB), que distribuye la carga entre los distintos contenedores de la primera capa. La comunicación del frontend con el backend se maneja de la misma manera.

# Deployment

# Local

El repositorio incluye un Dockerfile para el backend (`backend/Dockerfile`) y otro para el frontend (`frontend/Dockerfile`). Ambos utilizan Alpine Linux, una distribución liviana optimizada para contenedores. Cada Dockerfile define las instrucciones necesarias para instalar las dependencias del servicio y ejecutarlo.

El backend consume un archivo de variables de entorno ubicado en `backend/.env` con el siguiente formato:

```bash
SECRET_KEY=<secret-key>
DEBUG=1
SQL_ENGINE=django.db.backends.postgresql
SQL_DATABASE=postgres
SQL_USER=<user>
SQL_PASSWORD=<password>
SQL_HOST=database
SQL_PORT=4321
```

En el docker-compose se declaran 3 servicios: database, backend y frontend y se configuran para que sean capaces de comunicarse entre ellos, obtengan las variables de entorno necesarias y se reinicien en caso de algún tipo de fallo.

El projecto se levanta ejecutando:

```bash
docker compose up --build
```
![deployment](https://github.com/klartz/prueba-tecnica-craftech/blob/main/images/deployment.png)

## AWS

Para deployar la aplicación en AWS seguí los siguientes pasos

1. En AWS acceder al servicio EC2 y seleccionar la opción “Launch an instance”
    1. Selecciono “Ubuntu” como la imagen a utilizar para el sistema
    2. Creo un par de claves para poder conectarme via ssh
    3. Configuro la red para que acepte conexiones HTTPS y HTTP desde internet
2. Conectarse via ssh a la instancia

    ```bash
    scp -i "aws.pem" -o IdentitiesOnly=yes ubuntu@<ec2-ip>
    ```

3. Preparar docker
    1. Instalo docker siguiendo la guía de instalación de la documentación ( https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository)
    2. Configuro un grupo para poder correr comandos de docker sin sudo (https://docs.docker.com/engine/install/linux-postinstall/#manage-docker-as-a-non-root-user)
    3. Configuro el servicio de docker para que inicie automáticamente al bootear en el sistema (https://docs.docker.com/engine/install/linux-postinstall/#configure-docker-to-start-on-boot-with-systemd)
4. Preparar el proyecto

    ```bash
    git clone https://github.com/klartz/prueba-tecnica-craftech.git
    cd prueba-tecnica-craftech
    # transfiero las variables de entorno del backend a la instancia
    scp -i "aws.pem" -o IdentitiesOnly=yes ./backend/.env ubuntu@<ec2-ip>:/home/ubuntu/prueba-tecnica-craftech/backend/
    ```

5. Configurar los puertos en el panel de administración de Security Groups
    1. Configuro el puerto de acceso para el frontend como 3000, y el del panel de administración como el 8000.
6. Correr la aplicación

    ```bash
    docker compose up --build -d
    ```

    Nota: al intentar construir la imagen del frontend la instancia se quedaba sin memoria (pues tiene 1GB de RAM) y se desconectaba, por lo que opté por construir la imagen localmente y transferirla:

    ```bash
    docker build -t frontend .
    docker save -o frontend.tar frontend
    scp -i "aws.pem" -o IdentitiesOnly=yes frontend.tar ubuntu@<ec2-ip>:/home/ubuntu/prueba-tecnica-craftech/
    ```

    Luego para utilizar esta imagen al levantar la aplicación, es necesario modificar el `docker-compose.yml` para que utilice la imagen local:

    ```yaml
      frontend:
        image: frontend
        ...
    ```

    Y ejecutar:

    ```bash
    docker load -i frontend.tar
    docker compose up -d
    ```

# CI/CD pipeline
Los archivos correspondientes a la prueba 3 se encuentran en los directorios `cicd-pipeline/` y `.github/` dentro de este repositorio, con la siguiente estructura:
```sh
prueba-tecnica-craftech
├── .github
│   └── workflows
│		    └── deploy-nginx.yml
└── cicd-pipeline
		└── docker
		    └── nginx
		        ├── Dockerfile
		        └── index.html
```
El pipeline está definido en `.github/workflows/deploy-nginx.yml` y se activa únicamente ante cambios en `cicd-pipeline/docker/nginx/index.html`.
Cuenta con dos trabajos: `build`, que construye y sube la imagen al GitHub Container Registry, y `deploy`, que simula su despliegue ejecutando el contenedor en el runner, verificando que funcione correctamente.

En un entorno real, el job de despliegue consistiría en acceder al entorno de producción, obtener la imagen desde el registro y levantar el contenedor actualizado.

El resultado de cada ejecución puede consultarse desde la pestaña **Actions** del repositorio:
