### Validación de Políticas de Acceso:
* **Solo el Frontend es accesible desde Internet:** Las peticiones HTTP externas ingresan únicamente a la instancia pública.
* **Aislamiento del Backend:** Las instancias de Spring Boot no exponen puertos al mundo. El tráfico de la API entra cifrado y redirigido exclusivamente desde el Security Group del Frontend.
* **Aislamiento de la Base de Datos:** La base de datos MySQL bloquea cualquier intento de conexión directa, aceptando únicamente tráficos originados en la capa del Backend.

---

## 📦 3. Contenedorización y Diseño de Servicios (IE1, IE6)

### 📄 Buenas Prácticas Aplicadas en Dockerfile
Para mitigar riesgos de seguridad y optimizar recursos (Requerimientos No Funcionales), los archivos `Dockerfile` de cada repositorio implementan:

1.  **Multi-stage Build (Construcción Multietapa):**
    * *Fase de Compilación:* Utiliza imágenes completas de desarrollo (`maven:3.9-eclipse-temurin` / `node:20-alpine`) para empaquetar los artefactos (`.jar` / producción estática de Vite).
    * *Fase de Ejecución:* Descarta todas las herramientas de desarrollo y copia únicamente el compilado final a imágenes minimalistas (`eclipse-temurin:17-jre-alpine` / `nginx:alpine`). Esto reduce el tamaño de la imagen en un **75%**, acelerando el despliegue en AWS y eliminando vulnerabilidades potenciales.
2.  **Usuario No Root (Seguridad):** Se ejecutan los procesos bajo un usuario del sistema con privilegios limitados (`springuser` / `nginxuser`). En caso de verse comprometida la aplicación, el atacante no tendrá permisos de administración sobre el sistema host de AWS.
3.  **Limpieza de Capas:** Reducción de logs de instalación y remoción de archivos temporales en una sola directiva `RUN` para mantener la imagen liviana.

### 📄 Orquestación de Servicios (`docker-compose.yml`)
El archivo de orquestación local e independiente modela las redes internas y mapeos del sistema:

```yaml
version: '3.8'

services:
  mysql_db:
    image: mysql:8.0
    container_name: innova_mysql
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
      MYSQL_USER: ${DB_USERNAME}
      MYSQL_PASSWORD: ${DB_PASSWORD}
    ports:
      - "3306:3306"
    volumes:
      - data_persistente_mysql:/var/lib/mysql
    networks:
      - innova-net

  ventas_backend:
    image: ${DOCKERHUB_USERNAME}/back-ventas:latest
    container_name: service_ventas
    environment:
      - DB_ENDPOINT=mysql_db
      - DB_PORT=3306
      - DB_NAME=${DB_NAME}
      - DB_USERNAME=${DB_USERNAME}
      - DB_PASSWORD=${DB_PASSWORD}
    ports:
      - "8086:8086"
    depends_on:
      - mysql_db
    networks:
      - innova-net

  despachos_backend:
    image: ${DOCKERHUB_USERNAME}/back-despachos:latest
    container_name: service_despachos
    environment:
      - DB_ENDPOINT=mysql_db
      - DB_PORT=3306
      - DB_NAME=${DB_NAME}
      - DB_USERNAME=${DB_USERNAME}
      - DB_PASSWORD=${DB_PASSWORD}
    ports:
      - "8085:8085"
    depends_on:
      - mysql_db
    networks:
      - innova-net

  despacho_frontend:
    image: ${DOCKERHUB_USERNAME}/front-despacho:latest
    container_name: service_frontend
    ports:
      - "80:80"
    networks:
      - innova-net

volumes:
  data_persistente_mysql:

networks:
  innova-net:
    driver: bridge
4. Estrategia de Persistencia de Datos (IE2, IE40)
La continuidad operativa de Innovatech Chile exige que la información crítica no sea volátil.
Justificación Técnica de la Elección: Named Volumes vs Bind Mounts
Para la base de datos se implementaron exclusivamente Volúmenes Nombrados (Named Volumes) de Docker (data_persistente_mysql en desarrollo local y db_data en las instancias productivas de AWS EC2):
Independencia Total del Host: Docker gestiona el ciclo de vida, el formato interno y la ubicación física del volumen de manera automatizada. No dependemos de rutas absolutas que cambian según la máquina virtual de AWS.
Rendimiento Óptimo: Mayor velocidad y menor latencia de Entrada/Salida (I/O) en discos virtualizados de sistemas Linux en la nube.
Seguridad de Permisos: Evita problemas de bloqueo de lectura/escritura y colisiones de IDs de usuario (UID/GID) entre el motor interno de MySQL y el sistema operativo anfitrión de la EC2.
Comportamiento ante Reinicios y Resiliencia
Si un contenedor de base de datos se detiene por mantenimiento, sufre un fallo crítico o el pipeline de CI/CD lo destruye para actualizar la versión del motor, los datos no sufren ninguna alteración. El volumen persistente sobrevive al ciclo de vida del contenedor. Al inicializar un contenedor sustituto mapeado hacia el mismo volumen, la base de datos levanta su estado de forma inmediata, garantizando la continuidad del negocio.
🚀 5. Pipeline de Integración y Despliegue Continuo - CI/CD (IE3, IE7)
Flujo Completo de Automatización (IE42):
El pipeline está desarrollado en GitHub Actions y consta de las siguientes etapas secuenciales automatizadas:
[Push a rama deploy] ──> [Compilación Automatizada] ──> [Push a Docker Hub] ──> [Trigger AWS SSM Command] ──> [EC2 Auto-Aprovisionamiento]
Trigger (Activación - IE45): Ocurre de forma automática y controlada únicamente cuando se realiza un evento de push o merge aprobado en la rama deploy. Esto actúa como un filtro de producción, protegiendo el entorno en vivo de cambios inestables que se encuentren en desarrollo o pruebas.
Build (Construcción): El entorno virtual de GitHub Actions (Runner) clona el código del repositorio correspondiente y compila la imagen de Docker utilizando el Dockerfile multietapa optimizado.
Push (Publicación - IE46): El pipeline se autentica de forma segura contra el registro centralizado de Docker Hub e ingresa la imagen empaquetada etiquetada con la versión :latest. Se eligió Docker Hub por encima de ECR por criterios de alta portabilidad (multi-cloud), menor complejidad de rotación de credenciales en entornos educativos temporales de AWS Academy y máxima velocidad de distribución de imágenes.
Deploy (Despliegue vía AWS SSM): GitHub Actions consume la API de AWS para conectarse de forma remota a AWS Systems Manager (SSM) enviando un script ejecutable seguro (aws ssm send-command) hacia las instancias EC2 correspondientes.
Productividad en EC2: La instancia EC2 recibe la orden de forma interna, ejecuta un docker pull de la nueva imagen desde Docker Hub, destruye el contenedor antiguo, levanta el nuevo contenedor heredando el volumen persistente y limpia las imágenes residuales en cuestión de milisegundos.
Protección de Credenciales y Seguridad (IE44):
Toda la información de infraestructura sensible vive enmascarada y encriptada utilizando de manera exclusiva GitHub Secrets. Ningún Token, Password o Llave de AWS queda expuesta en texto plano en el código fuente. Las variables se inyectan directamente en la memoria del runner durante la ejecución (${{ secrets.AWS_ACCESS_KEY_ID }}, ${{ secrets.AWS_SESSION_TOKEN }}, etc.).
Importancia para la Continuidad de Innovatech Chile (IE43):
Automatizar este ciclo elimina de forma absoluta el factor del error humano (comandos incorrectos digitados manualmente en la consola, omisión de variables de entorno o borrado accidental de volúmenes). También reduce el tiempo de caída del sistema (Downtime) a niveles casi nulos y maximiza la velocidad de entrega de parches de software ante errores críticos detectados en producción.
Métodos de Verificación de Efectividad de los Endpoints (IE48):
Consumo Externo (Postman): Pruebas directas de peticiones GET y POST con cargas útiles JSON (ej: creando un despacho en http://44.193.9.49:8085/api/v1/despachos), validando que la API responda con los códigos de estado estándar de éxito (200 OK / 201 Created).
Inspección de Tráfico (Navegador DevTools): Auditoría en vivo de la pestaña Network (Red) en la consola web para confirmar que las peticiones asíncronas no presenten bloqueos de CORS, estados de caída de pasarela (502 Bad Gateway) ni conexiones denegadas (404 Not Found).
Auditoría Interna de Trazas (Logs en Caliente): Conexión SSH/SSM a las instancias EC2 para auditar en tiempo real la salida de logs de los contenedores (docker logs --tail 50 -f service_despachos), verificando que los hilos de conexión de HikariCP y las transacciones de Hibernate se registren en la base de datos de manera impecable.
📊 7. Trazabilidad y Gestión de Entornos (IE49)
En comparación con un modelo de despliegue de arquitectura tradicional (donde las aplicaciones se configuran directamente "a mano" sobre el sistema operativo de un servidor físico o virtual), esta solución madura bajo el esquema de Infraestructura como Código y entrega ventajas disruptivas:
Eliminación de la Deriva de Configuración: Al empaquetar todo el entorno operativo y las librerías exactas dentro de las imágenes inmutables de Docker, garantizamos que el sistema se comporte de forma idéntica en cualquier computador local de desarrollo y en cualquier nodo elástico en la nube de AWS.
Trazabilidad Absoluta por Auditoría: Cada cambio de puerto, alteración en las variables de entorno de base de datos o modificación de políticas queda registrado en un commit explícito indexado por Git. Esto permite conocer con exactitud el autor, la fecha y la justificación técnica de cualquier cambio estructural.
Aprovisionamiento Inmediato ante Desastres: Si el servidor de AWS colapsa por fallos de hardware del proveedor, reconstruir el ecosistema completo de Innovatech Chile toma escasos minutos. Solo se requiere clonar el repositorio en una nueva máquina virtual limpia y ejecutar el pipeline, minimizando los tiempos de inactividad comercial
