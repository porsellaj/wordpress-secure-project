WordPress Secure Project
Proyecto académico de despliegue y seguridad de aplicaciones web realizado con Docker Compose sobre una instancia Ubuntu en AWS EC2.

1. Objetivo
Desplegar WordPress con una base de datos MariaDB, caché Redis, dos instancias de WordPress detrás de Nginx, y acceso al sitio mediante HTTPS usando un certificado autofirmado. También se contemplan plugins de seguridad para WordPress.

2. Arquitectura
Navegador
   | HTTPS :443
   v
 Nginx (proxy inverso / balanceador)
   |----------------------|
   v                      v
WordPress              WordPress2
   |                      |
   |------ MariaDB -------|  Base de datos
   |                      |
   |-------- Redis -------|  Caché de objetos
Componente	Función
Nginx	Recibe las solicitudes web, redirige HTTP a HTTPS y reparte las solicitudes entre las dos instancias de WordPress.
WordPress / WordPress2	Ejecutan el CMS. Se usa una imagen personalizada con la extensión PHP Redis instalada.
MariaDB 11	Guarda las publicaciones, usuarios, opciones y demás datos del sitio.
Redis 7	Servicio de caché en memoria. Para que WordPress lo utilice, también debe estar instalada y habilitada la extensión PHP Redis y activado el plugin Redis Object Cache.
Docker Compose	Declara y ejecuta los servicios como una aplicación.
Certificado TLS autofirmado	Permite cifrar la conexión HTTPS en el entorno académico. Los navegadores mostrarán una advertencia porque el certificado no está firmado por una autoridad de confianza.
Las dos instancias comparten la base de datos y el volumen de archivos de WordPress para que tengan disponibles los mismos temas, plugins y archivos subidos. Todo se ejecuta en una sola instancia EC2: la prueba demuestra tolerancia a la caída de un contenedor WordPress, no alta disponibilidad ante la caída del servidor EC2.

3. Estructura del proyecto
wordpress-secure-project/
├── README.md
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── .gitignore
├── nginx/
│   └── nginx.conf
└── ssl/
    └── .gitkeep
Dockerfile: construye la imagen personalizada de WordPress e instala la extensión PHP Redis.
docker-compose.yml: declara MariaDB, Redis, las dos instancias WordPress y Nginx.
.env.example: plantilla de variables de entorno. Se copia como .env y se completan los secretos localmente.
nginx/nginx.conf: configuración del proxy inverso, el balanceo y HTTPS.
ssl/: contiene localmente el certificado público (server.crt) y la clave privada (server.key). No se suben al repositorio.
4. Requisitos previos
Una instancia EC2 con Ubuntu y una Elastic IP asociada.
Reglas de entrada del Security Group: SSH (22) solo desde la IP administrativa y HTTPS (443) para los visitantes. El puerto 80 debe quedar abierto si se quiere que Nginx pueda responder a HTTP y redirigirlo a HTTPS; si se bloquea en AWS, las solicitudes HTTP externas no alcanzarán la redirección.
Docker Engine y Docker Compose v2.
Un nombre DNS o una IP pública para acceder al sitio.
5. Instalación y despliegue
Paso 1. Instalar Docker
Si Docker aún no está instalado en Ubuntu:

sudo apt update
sudo apt upgrade -y
sudo apt install -y docker.io docker-compose-v2 git
sudo systemctl enable docker
sudo systemctl start docker

docker --version
sudo docker compose version
El nombre correcto del servicio de systemd es docker en minúsculas. Se usa sudo docker ... para ejecutar comandos de Docker con privilegios; no hace falta añadir el usuario al grupo docker para esta práctica.

Paso 2. Preparar las variables de entorno
Desde la raíz del repositorio:

cp .env.example .env
nano .env
Cambiar los valores de ejemplo por contraseñas nuevas y fuertes. Los valores del usuario y la contraseña de MariaDB deben coincidir con los que WordPress utiliza en el mismo archivo Compose.

El archivo .env no cifra las contraseñas: solo evita escribirlas directamente en docker-compose.yml. Por esa razón está excluido de Git.

Paso 3. Crear el certificado TLS autofirmado
Desde la raíz del proyecto:

mkdir -p ssl nginx
sudo openssl req -x509 -nodes -days 365 -newkey rsa:4096 \
  -keyout ssl/server.key \
  -out ssl/server.crt
Cuando OpenSSL pregunte por Common Name (CN), escribir la Elastic IP o el nombre DNS que se utilizará para abrir el sitio. La clave privada ssl/server.key debe permanecer en el servidor y no debe subirse a GitHub.

El certificado es autofirmado. HTTPS sí cifra la conexión, pero el navegador no puede verificar su identidad usando una autoridad certificadora pública; por eso muestra una advertencia.

Paso 4. Validar y arrancar la aplicación
sudo docker compose config
sudo docker compose up -d --build
sudo docker ps
docker compose config permite detectar errores de sintaxis o variables faltantes antes de arrancar.
up -d --build construye la imagen definida en Dockerfile y arranca los servicios en segundo plano.
docker ps permite comprobar el estado de los contenedores.
Para revisar los registros:

sudo docker logs wordpress
sudo docker logs wordpress2
sudo docker logs mariadb
sudo docker logs nginx
Abrir https://TU_IP_PUBLICA y completar el asistente de instalación de WordPress.

Paso 5. Configurar Redis en WordPress
El Dockerfile instala la extensión Redis para PHP. Después de instalar WordPress, se debe indicar a WordPress el nombre del servicio Redis en wp-config.php.

sudo docker exec -it wordpress bash
Dentro del contenedor, abrir el archivo con un editor disponible (si hace falta, instalar nano dentro del contenedor):

apt update
apt install -y nano
nano /var/www/html/wp-config.php
Añadir estas líneas antes del comentario final de WordPress, antes de /* That's all, stop editing! Happy publishing. */ si aparece:

define('WP_REDIS_HOST', 'redis');
define('WP_REDIS_PORT', 6379);
Guardar, salir del contenedor y reiniciar las instancias:

exit
sudo docker restart wordpress wordpress2
Comprobar que la extensión PHP está disponible:

sudo docker exec wordpress php -m | grep -i redis
La salida esperada incluye redis. Después, en el panel de WordPress, instalar y activar el plugin Redis Object Cache y habilitar Object Cache. El contenedor Redis en ejecución, por sí solo, no demuestra que WordPress esté utilizando Redis: hay que comprobar que el plugin indique una conexión correcta.

Como ambas instancias comparten el volumen de WordPress, wp-config.php es compartido. Por ello, normalmente basta con editarlo desde wordpress y reiniciar ambas instancias.

Paso 6. Activar los plugins de seguridad
En el panel de WordPress, instalar y activar:

Wordfence Security: proporciona funciones de protección y análisis de seguridad. Verificar el estado de las funciones en su panel antes de afirmar que todas están activas.
WPS Hide Login: permite configurar una ruta de acceso personalizada al inicio de sesión. Guardar la ruta exacta configurada y probar tanto esa ruta como la ruta predeterminada.
Redis Object Cache: habilita la caché de objetos con el servicio Redis.
Cambiar la ruta de inicio de sesión no reemplaza contraseñas fuertes, actualizaciones ni los demás controles de seguridad.

6. Cómo funciona Nginx
El archivo nginx/nginx.conf define un grupo upstream llamado wordpress_backend, que contiene wordpress:80 y wordpress2:80.

El bloque que escucha en el puerto 80 responde con una redirección 301 a HTTPS.
El bloque del puerto 443 carga server.crt y server.key y permite TLS 1.2.
proxy_pass http://wordpress_backend reenvía la solicitud al grupo de instancias WordPress.
Las cabeceras Host, X-Real-IP, X-Forwarded-For y X-Forwarded-Proto conservan información sobre el host solicitado, la IP del cliente y el protocolo original.
Nginx termina la conexión TLS y se comunica con WordPress por HTTP dentro de la red privada de Docker. Este diseño cifra el tráfico navegador–Nginx; no cifra por separado el tráfico entre los contenedores en la red Docker.

7. Pruebas para la sustentación
Ejecutar los comandos desde la instancia EC2, en la carpeta del proyecto.

A. Ver los cinco contenedores
sudo docker ps
Deben aparecer mariadb, redis, wordpress, wordpress2 y nginx en estado activo.

B. Comprobar la redirección HTTP
curl -I http://localhost
Si el puerto 80 está publicado y Nginx está activo, se espera una respuesta 301 con una cabecera Location que apunta a HTTPS. Si el puerto 80 está bloqueado en el Security Group, esta comprobación local aún puede funcionar, pero la solicitud HTTP externa no llegará a Nginx.

C. Comprobar el certificado y la clave RSA
openssl x509 -in ssl/server.crt -text -noout | grep -E 'Public-Key|Not Before|Not After'
La clave pública debe mostrar 4096 bit con el comando de generación anterior. En el navegador, abrir https://TU_IP_PUBLICA y explicar que la advertencia se debe al certificado autofirmado.

D. Comprobar TLS 1.2
openssl s_client -connect TU_IP_PUBLICA:443 -tls1_2 </dev/null
Revisar la salida y buscar una sesión negociada con TLS 1.2 y un cifrado. Una conexión correcta no elimina la advertencia de confianza del certificado autofirmado.

E. Demostrar que Nginx puede usar la otra instancia
sudo docker stop wordpress2
Recargar varias veces la página HTTPS y comprobar si sigue respondiendo mediante la instancia que queda activa. Después restaurar el servicio:

sudo docker start wordpress2
sudo docker ps
Si se repite la prueba con wordpress, restaurarlo inmediatamente después. No detener MariaDB, Redis ni Nginx durante la demostración.

F. Mostrar la configuración Redis
sudo docker exec wordpress php -m | grep -i redis
sudo docker exec wordpress grep -n 'WP_REDIS' /var/www/html/wp-config.php
Complementar la prueba mostrando el estado del plugin Redis Object Cache en el panel de WordPress.

G. Revisar la configuración del balanceador
cat nginx/nginx.conf
Señalar las dos entradas de upstream wordpress_backend y explicar que ambas instancias reciben las solicitudes de Nginx.

8. Cambiar la URL almacenada en WordPress
Si se cambia la Elastic IP y WordPress continúa redirigiendo a una dirección antigua, siteurl y home pueden seguir guardando la URL anterior. Con el usuario root de MariaDB, conectarse a la base de datos y ejecutar las consultas solo si se necesita actualizarla:

sudo docker exec -it mariadb mariadb -u root -p
Dentro del cliente SQL:

USE wordpress;
UPDATE wp_options SET option_value='https://TU_IP_PUBLICA' WHERE option_name='siteurl';
UPDATE wp_options SET option_value='https://TU_IP_PUBLICA' WHERE option_name='home';
SELECT option_name, option_value FROM wp_options WHERE option_name IN ('siteurl','home');
Sustituir TU_IP_PUBLICA por la dirección actual. Estas consultas suponen que WordPress usa la base wordpress y el prefijo estándar wp_; si se configuró otro nombre de base o prefijo, ajustar las consultas. También se puede cambiar la URL desde la configuración de WordPress si el panel sigue accesible.

9. Operación habitual
# Estado de los servicios
sudo docker ps

# Logs de Nginx
sudo docker logs nginx

# Validar la configuración Compose
sudo docker compose config

# Reiniciar Nginx después de un cambio de configuración
sudo docker compose restart nginx

# Parar el proyecto
sudo docker compose down

# Volver a levantarlo
sudo docker compose up -d --build
docker compose down conserva los volúmenes con datos si no se usa -v. No ejecutar docker compose down -v salvo que se quiera borrar también la base de datos y los archivos persistentes.

10. Seguridad y limitaciones
No versionar .env, ssl/server.key, copias de seguridad ni datos del sitio.
Si una contraseña real se compartió en un chat, captura o repositorio, cambiarla antes de usar el sistema fuera del laboratorio.
Las contraseñas de ejemplo no deben reutilizarse.
El certificado autofirmado sirve para practicar TLS; para un servicio público se necesita un certificado válido y una estrategia de renovación.
Nginx, WordPress, MariaDB y Redis comparten una sola instancia EC2. Si la instancia falla, todo el sitio deja de estar disponible.
Dos instancias de aplicación no equivalen por sí solas a alta disponibilidad: también importan el servidor subyacente, el almacenamiento compartido, la base de datos y la supervisión.
ModSecurity no está incluido en la configuración descrita. Añadirlo sería una ampliación, no una función que deba afirmarse como implementada.
11. Referencias oficiales
Docker Compose: https://docs.docker.com/compose/
Imagen oficial de WordPress: https://hub.docker.com/_/wordpress
Imagen oficial de MariaDB: https://hub.docker.com/_/mariadb
Imagen oficial de Redis: https://hub.docker.com/_/redis
Nginx: https://nginx.org/en/docs/
OpenSSL: https://docs.openssl.org/
Wordfence: https://www.wordfence.com/help/
WPS Hide Login: https://wordpress.org/plugins/wps-hide-login/
Redis Object Cache: https://wordpress.org/plugins/redis-cache/
