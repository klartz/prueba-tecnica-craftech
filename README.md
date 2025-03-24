# Diagrama de Red

![diagrama de red](images/network-diagrama.png)

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
