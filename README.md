# randomquotes – Tutoriales 06 y 07

**Autor:** Juan Esteban Peña Rojas
**Materia:** Arquitectura de Software (EAFIT)

Microservicio en Node.js/Express que devuelve una frase aleatoria. Este repositorio contiene el trabajo de dos tutoriales:

- [Tutorial 06](#tutorial-06--express-y-docker-en-una-instancia-aws-ec2): la aplicación en un contenedor Docker, desplegada en una instancia AWS EC2.
- [Tutorial 07](#tutorial-07--microservicio-con-réplicas-en-aws-ecs-y-cicd): el mismo microservicio con 4 réplicas detrás de un balanceador de carga en AWS, y despliegue automático con GitHub Actions.

## Contenido del repositorio

| Archivo | Descripción |
|---|---|
| `index.js` | Aplicación Express (puerto 80). En su versión final (Tutorial 07) devuelve una frase y el `Container Id` (`os.hostname()`). |
| `Dockerfile` | Imagen basada en `node:24-alpine`. |
| `package.json` / `package-lock.json` | Dependencias (Express). |
| `.github/workflows/deploy.yml` | Pipeline de CI/CD del Tutorial 07. |
| `evidencia/` | Capturas de pantalla de ambos tutoriales. |

---

# Tutorial 06 – Express y Docker en una instancia AWS EC2

Microservicio Express que devuelve una frase aleatoria, desplegado con Docker en una instancia AWS EC2.

## Despliegue
```bash
docker build -t randomquotes-juan-pena:1.0 .
docker run -d --name randomquotes-juan-pena --restart unless-stopped -p 8083:80 randomquotes-juan-pena:1.0
```

La aplicación escucha en el puerto 80 dentro del contenedor y se publica en el puerto 8083 de la instancia, que es el asignado a este estudiante en la instancia compartida del grupo. Los nombres de imagen y contenedor llevan el sufijo `juan-pena` para no chocar con los de los compañeros.

## Evidencia
La aplicación corriendo en AWS (http://IP:8083). Cada recarga muestra una frase distinta:

en la carpeta eviencia taller 6 

## Notas
- La instancia compartida tiene unos 900 MB de RAM y no tenía swap. Para poder construir la imagen sin afectar los contenedores de los compañeros se agregó un archivo de swap de 1 GB.
- En este tutorial la respuesta es solo la frase. El identificador del contenedor se agregó después, en el Tutorial 07.

---

# Tutorial 07 – Microservicio con réplicas en AWS (ECS y CI/CD)

Se despliega el mismo microservicio en AWS con 4 réplicas detrás de un balanceador de carga, y cada `push` a `main` lo vuelve a desplegar automáticamente con GitHub Actions. La respuesta incluye el identificador del contenedor para poder ver qué réplica atendió cada solicitud.

## Arquitectura desplegada

- **Amazon ECR:** repositorio privado `randomquotes` (región us-east-2) con la imagen `latest`.
- **Amazon ECS (Fargate):** clúster `randomquotes-cluster`, definición de tarea `randomquotes-task` (0.25 vCPU, 0.5 GB, puerto 80) y servicio `randomquotes-service` con **4 tareas**.
- **Application Load Balancer:** `randomquotes-alb` (HTTP 80) con grupo de destino `randomquotes-targets` (tipo IP, health check en `/`).
- **Grupos de seguridad:** `randomquotes-alb-sg` (HTTP 80 desde internet) y `randomquotes-task-sg` (HTTP 80 solo desde el balanceador).
- **GitHub Actions:** workflow que se ejecuta en cada push a `main`.

## Pasos realizados

1. Se modificó `index.js` para mostrar el identificador del contenedor (`os.hostname()`).
2. Se creó el repositorio ECR y se publicó la imagen desde AWS CloudShell (`docker build`, `docker tag`, `docker push`).
3. Se creó el clúster ECS (Fargate) y la definición de tarea.
4. Se crearon los grupos de seguridad y el servicio ECS con 4 tareas y un Application Load Balancer.
5. Se comprobó el balanceo y la recuperación automática.
6. Se configuró el workflow de GitHub Actions y se verificó un despliegue automático.

## Evidencia

en evidencia taller 7

### 1. Imagen publicada en ECR


### 2. Servicio ECS conectado al balanceador de carga


### 3. Cuatro destinos sanos en el grupo de destino


### 4. Balanceo de carga y recuperación automática
Al recargar la URL del balanceador aparecen identificadores de contenedor distintos (por ejemplo `ip-172-31-9-146` e `ip-172-31-19-132`). Las capturas se tomaron mientras se detenía una tarea: el servicio pasó a 3 en ejecución y 1 en espera, y la aplicación siguió respondiendo.

### 5. Despliegues automáticos con GitHub Actions


### 6. Aplicación actualizada tras un push (v2)
Se agregó ` v2` a una frase de `index.js`, se hizo `push` a `main` y el workflow desplegó la nueva versión sin pasos manuales.



## Respuestas a las preguntas del tutorial

**Papel del Application Load Balancer.** Recibe todas las solicitudes en una única dirección (su nombre DNS) y las reparte entre las tareas sanas del grupo de destino. Así los usuarios no necesitan conocer las IP de cada tarea, la carga se distribuye entre las 4 réplicas y, gracias al health check, deja de enviar tráfico a una tarea que no responde.

**¿Por qué se crea una nueva tarea al detener una?** Porque el servicio ECS declara un estado deseado de 4 tareas. Cuando una se detiene, el número en ejecución cae por debajo de ese valor y ECS inicia una tarea de reemplazo para recuperarlo.

**¿Qué componente mantiene la cantidad deseada?** El **programador (scheduler) del servicio de ECS**, que compara continuamente las tareas en ejecución con el *desired count* y crea o elimina tareas para igualarlos.

**¿Qué efecto tiene el health check del balanceador?** El balanceador consulta periódicamente la ruta `/` de cada destino. Si un destino no responde, se marca como *unhealthy* y deja de recibir solicitudes. Además, ECS puede reemplazar las tareas que fallan esa comprobación. Esto se observó durante las pruebas: el balanceador devolvió `503` mientras no había destinos sanos y volvió a responder cuando el primero pasó a *healthy*.

## Notas sobre el despliegue

- La cuenta de AWS utilizada no permite configurar el proveedor de identidad OIDC (la consola indica que la característica requiere habilitar funciones avanzadas). Por eso el workflow se autentica con un **usuario IAM de mínimo alcance** (`AmazonEC2ContainerRegistryPowerUser` y `AmazonECS_FullAccess`) cuyas claves se guardaron como *secrets* de GitHub (`AWS_ACCESS_KEY_ID` y `AWS_SECRET_ACCESS_KEY`). Ninguna clave está en el repositorio. Con OIDC sería preferible, porque evita credenciales de larga duración.
- La imagen usa la etiqueta `latest` por simplicidad. En producción convendría etiquetar con el SHA del commit y registrar una nueva revisión de la definición de tarea.
- Durante la configuración, el servicio se creó por primera vez con 1 tarea deseada y con el grupo de seguridad `default`. Eso causó un `ERR_CONNECTION_TIMED_OUT` y luego un `503`; se corrigió asignando los grupos de seguridad correctos y subiendo las tareas deseadas a 4.
- Al terminar las pruebas se eliminaron los recursos de AWS (servicio, balanceador, grupo de destino, clúster, repositorio ECR y usuario IAM) para no consumir créditos.

## Conclusiones

- Empaquetar la aplicación en un contenedor (Tutorial 06) permite ejecutarla igual en una instancia EC2, en ECS o en cualquier otro entorno.
- Publicar la imagen en un registro (ECR) y usar ECS con Fargate evita administrar servidores: se define cuántas réplicas se desean y la plataforma las mantiene.
- Un balanceador con health checks aporta escalabilidad horizontal y tolerancia a fallos: si una réplica cae, las demás siguen atendiendo y ECS la reemplaza.
- Los grupos de seguridad son críticos: la mayoría de los problemas del despliegue (timeouts, `503`, destinos *unhealthy*) se debieron a reglas de red mal asignadas.
- La integración continua con GitHub Actions reduce el despliegue a un `git push`, lo que disminuye errores manuales y acorta el tiempo de entrega.
