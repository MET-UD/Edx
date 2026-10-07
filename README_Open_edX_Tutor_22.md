# Open edX 22.0.2 + Tutor 22.0.2 — Instalación local en Windows 10 + WSL2 + Docker Desktop

## 1. Objetivo

Este documento registra, de forma reproducible, la instalación local de Open edX para un entorno de práctica técnica.

La instalación realizada utiliza:

- Windows 10 x64 como sistema anfitrión.
- WSL2.
- Ubuntu 24.04.3 LTS como entorno Linux.
- Docker Desktop como motor Docker.
- Docker Engine accesible desde WSL2.
- Python 3.12.
- Entorno virtual Python (`venv`).
- Tutor 22.0.2.
- Open edX 22.0.2.
- Tema Indigo.
- Docker Compose.
- MySQL 8.4.
- MongoDB 7.0.
- Redis 7.4.
- Meilisearch 1.36.
- Caddy 2.11.
- Exim SMTP relay.
- Open edX MFE 22.1.0.

El objetivo no fue instalar un segundo Docker Engine dentro de Ubuntu. Docker Desktop proporciona el Engine y Ubuntu/WSL2 proporciona la consola Linux desde la cual se ejecutan Docker y Tutor.

---

# 2. Arquitectura resultante

```text
                         WINDOWS 10
                             │
                             │
                    ┌────────▼────────┐
                    │   Docker Desktop │
                    │ Docker Engine    │
                    └────────┬────────┘
                             │
                    integración WSL2
                             │
              ┌──────────────▼──────────────┐
              │       Ubuntu 24.04.3 LTS     │
              │              WSL2            │
              │                              │
              │  Python 3.12                 │
              │  venv (.venv)                │
              │  Tutor 22.0.2                │
              │  Docker CLI                  │
              └──────────────┬───────────────┘
                             │
                       Tutor 22.0.2
                             │
                  Docker Compose / tutor_local
                             │
       ┌─────────────────────┼─────────────────────┐
       │                     │                     │
       ▼                     ▼                     ▼
     Caddy                  LMS                  CMS/Studio
       │                     │                     │
       │                     ├── LMS Worker        ├── CMS Worker
       │                     │
       ├─────────────── MFE / Frontend
       │
       ├── MySQL
       ├── MongoDB
       ├── Redis
       ├── Meilisearch
       └── SMTP
```

Open edX utiliza `openedx-platform` como núcleo, con LMS y Studio/CMS, y complementa este núcleo con servicios desplegables independientemente y micro-frontends (MFEs). El backend utiliza principalmente Python/Django y los MFEs utilizan React. 

---

# 3. Requisitos de hardware utilizados

Equipo utilizado durante la instalación:

- CPU: AMD Ryzen 5 7520U
- 4 núcleos / 8 hilos
- RAM física: aproximadamente 15.28 GB utilizables
- SSD: Samsung MZVL4512HBLU-00BTW, aproximadamente 477 GB
- Windows 10 x64
- Espacio libre disponible durante la preparación: aproximadamente 120 GB

Docker informó:

```text
CPUs: 8
Memoria: 7946502144 bytes
```

Esto equivale aproximadamente a 7.4 GiB disponibles para Docker.

Ubuntu informó:

```text
Mem:           7.4Gi
used           899Mi
free           4.8Gi
buff/cache     1.9Gi
available      6.5Gi

Swap:          2.0Gi
```

La documentación actual de Open edX recomienda, para una experiencia fluida de desarrollo, al menos 16 GB de RAM y 50 GB de espacio libre. 

---

# 4. Tecnologías y versiones

## Sistema operativo

Windows:

```text
Windows 10 x64
Build 10.0.19045.6466
```

WSL:

```text
WSL 3.0.1.0
Kernel 6.18.40.1-1
WSL2
```

Distribución:

```text
Ubuntu 24.04.3 LTS
```

Docker:

```text
Docker Desktop 29.8.1
Docker Compose 5.5.1
Docker Engine 29.8.1
```

Python:

```text
Python 3.12.3
```

Tutor:

```text
Tutor 22.0.2
```

Open edX:

```text
Open edX 22.0.2
```

MFE:

```text
Open edX MFE 22.1.0
```

---

# 5. Arquitectura de WSL2 y Docker Desktop

Se verificó:

```powershell
wsl --list --verbose
```

Resultado:

```text
NAME              STATE     VERSION
Ubuntu            Running   2
docker-desktop    Running   2
```

La arquitectura utilizada es:

```text
Windows
   │
   ├── WSL2
   │    └── Ubuntu
   │         └── Tutor + Docker CLI
   │
   └── Docker Desktop
        └── Docker Engine
             └── contenedores Open edX
```

No se instaló un Docker daemon independiente dentro de Ubuntu.

Se comprobó desde Ubuntu:

```bash
docker --version
docker ps
```

y Docker pudo visualizar el contenedor existente de n8n.

---

# 6. Configuración de DNS de WSL

Inicialmente WSL tenía problemas de resolución DNS.

Se comprobó que la conectividad IP funcionaba:

```bash
ping -c 3 8.8.8.8
```

pero:

```bash
ping -c 3 archive.ubuntu.com
```

fallaba con:

```text
Temporary failure in name resolution
```

Se configuró `/etc/wsl.conf`:

```ini
[boot]
systemd=true

[user]
default=ser

[network]
generateResolvConf = false
```

Después se reemplazó `/etc/resolv.conf`:

```bash
sudo rm /etc/resolv.conf
sudo bash -c 'echo "nameserver 1.1.1.1" > /etc/resolv.conf'
```

Finalmente se comprobó:

```bash
ping -c 3 archive.ubuntu.com
```

y la resolución DNS funcionó correctamente.

---

# 7. Actualización de Ubuntu

Se ejecutó:

```bash
sudo apt update
```

La actualización terminó correctamente.

Después:

```bash
sudo apt install python3-pip python3-venv
```

Esto instaló:

- pip
- venv
- dependencias de Python
- paquetes necesarios para crear el entorno virtual.

Se verificó:

```bash
python3 --version
```

Resultado:

```text
Python 3.12.3
```

También:

```bash
pip --version
```

y se comprobó que `venv` estaba disponible.

Durante la instalación apareció un aviso relacionado con:

```text
systemd-binfmt.service
```

pero no impidió utilizar Python, pip ni venv.

---

# 8. Creación del directorio de trabajo

Se creó:

```bash
cd ~
mkdir -p openedx
cd openedx
```

La ruta final utilizada fue:

```text
/home/ser/openedx
```

---

# 9. Creación del entorno virtual Python

Se ejecutó:

```bash
python3 -m venv .venv
```

Activación:

```bash
source .venv/bin/activate
```

El prompt pasó a mostrar:

```text
(.venv) ser@Sebas:~/openedx$
```

El entorno virtual es importante porque evita mezclar las dependencias de Tutor con las dependencias Python globales de Ubuntu.

---

# 10. Instalación de Tutor

Se instaló Tutor con:

```bash
python3 -m pip install "tutor[full]"
```

La instalación terminó correctamente.

Se verificó:

```bash
tutor --version
```

Resultado:

```text
tutor, version 22.0.2
```

La documentación de Open edX recomienda Tutor como método de instalación basado en Docker para Open edX. 

---

# 11. Configuración inicial de Tutor

Inicialmente:

```bash
tutor config printvalue LMS_HOST
```

mostró:

```text
Error: Project root does not exist.
```

Esto era esperado porque todavía no se había generado la configuración inicial.

Se ejecutó:

```bash
tutor config save --interactive
```

Se seleccionó:

```text
Are you configuring a production platform?
n
```

Título:

```text
Open edX - Prueba DNIA
```

Correo:

```text
contact@local.openedx.io
```

Idioma:

```text
es-419
```

Nota: `es` no es un código válido para esta configuración de Tutor. Tutor mostró la lista de códigos permitidos y se utilizó `es-419`.

La configuración quedó guardada en:

```text
/home/ser/.local/share/tutor/config.yml
```

y el entorno generado en:

```text
/home/ser/.local/share/tutor/env
```

---

# 12. Verificación de LMS_HOST

Se ejecutó:

```bash
tutor config printvalue LMS_HOST
```

Resultado:

```text
local.openedx.io
```

Por tanto:

```text
LMS  = http://local.openedx.io
Studio = http://studio.local.openedx.io
```

Para el entorno local:

```text
ENABLE_HTTPS=False
```

---

# 13. Lanzamiento de Open edX

El comando utilizado fue:

```bash
tutor local launch
```

Tutor volvió a solicitar:

```text
Are you configuring a production platform?
```

Se respondió:

```text
n
```

Los demás valores se conservaron:

```text
Open edX - Prueba DNIA
contact@local.openedx.io
es-419
```

---

# 14. Descarga de imágenes Docker

Durante el primer lanzamiento Tutor descargó las imágenes necesarias.

Se observaron 133 elementos descargados:

```text
[+] up 133/133
```

Entre las imágenes principales:

```text
docker.io/overhangio/openedx:22.0.2-indigo
docker.io/overhangio/openedx-mfe:22.1.0-indigo
docker.io/overhangio/openedx-permissions:22.0.2
docker.io/mysql:8.4.11
docker.io/mongo:7.0.39
docker.io/redis:7.4.10
docker.io/getmeili/meilisearch:v1.36.0
docker.io/caddy:2.11.4
docker.io/devture/exim-relay:4.96-r1-0
```

Esto representa una de las partes más largas de la instalación.

Una vez descargadas, las imágenes permanecen almacenadas en Docker y no es necesario volver a descargarlas completas en cada `tutor local launch`.

---

# 15. Servicios creados

Tutor creó el proyecto Docker Compose:

```text
tutor_local
```

Los servicios finales observados fueron:

```text
tutor_local-caddy-1
tutor_local-cms-1
tutor_local-cms-worker-1
tutor_local-lms-1
tutor_local-lms-worker-1
tutor_local-meilisearch-1
tutor_local-mfe-1
tutor_local-mongodb-1
tutor_local-mysql-1
tutor_local-redis-1
tutor_local-smtp-1
```

Estos servicios forman la plataforma Open edX local.

---

# 16. Función de cada servicio

## Caddy

Actúa como servidor/proxy frontal y publica el servicio HTTP.

En esta instalación:

```text
0.0.0.0:80 -> 80
```

## LMS

Learning Management System.

Es la aplicación utilizada por los estudiantes.

Funciones:

- autenticación;
- dashboard;
- acceso a cursos;
- progreso;
- contenido;
- perfiles;
- navegación del aprendizaje.

## CMS / Studio

Es el sistema de autoría de Open edX.

Se utiliza para:

- crear cursos;
- modificar cursos;
- crear secciones;
- crear subsecciones;
- crear unidades;
- agregar componentes;
- configurar contenidos;
- publicar cambios.

La documentación oficial identifica Studio como el CMS/entorno de autoría y al LMS como la experiencia de aprendizaje. 

## LMS Worker

Ejecuta tareas asíncronas asociadas al LMS mediante Celery.

## CMS Worker

Ejecuta tareas asíncronas asociadas al CMS/Studio.

## MFE

Micro-Frontends.

Open edX utiliza React para los MFEs y muchas partes modernas de la interfaz se implementan mediante estos frontends. 

## MySQL

Base de datos relacional.

Se utiliza para datos transaccionales y de aplicación.

## MongoDB

Open edX utiliza MongoDB para almacenar contenido de cursos. 

## Redis

Se utiliza para diferentes mecanismos de caching y tareas/servicios que necesitan almacenamiento rápido.

## Meilisearch

Motor de búsqueda utilizado por componentes de Open edX.

## SMTP

Servidor de relay para correo electrónico en el entorno local.

---

# 17. Migraciones e inicialización

Durante `tutor local launch`, Tutor ejecutó tareas de inicialización.

Entre ellas:

```text
./manage.py lms migrate
```

También:

- creación de índices de Meilisearch;
- creación/configuración de usuarios de servicio;
- configuración OAuth entre LMS y Studio;
- configuración de switches;
- carga de políticas de autorización;
- configuración del tema Indigo;
- inicialización de servicios.

Se observaron mensajes de advertencia de Django/dependencias, por ejemplo `DeprecationWarning` y advertencias de modelos.

Estas advertencias no impidieron que los servicios terminaran de inicializarse.

---

# 18. Verificación final

La comprobación definitiva fue:

```bash
tutor local status
```

El resultado mostró todos los servicios con estado:

```text
Up
```

Entre ellos:

```text
tutor_local-cms-1
tutor_local-cms-worker-1
tutor_local-lms-1
tutor_local-lms-worker-1
tutor_local-mfe-1
tutor_local-mongodb-1
tutor_local-mysql-1
tutor_local-redis-1
tutor_local-meilisearch-1
tutor_local-caddy-1
tutor_local-smtp-1
```

Por tanto, la instalación local quedó operativa.

---

# 19. URLs de la instalación

LMS:

```text
http://local.openedx.io
```

Studio:

```text
http://studio.local.openedx.io
```

Meilisearch:

```text
http://meilisearch.local.openedx.io
```

Aplicaciones/MFEs:

```text
http://apps.local.openedx.io
```

Estas son las URLs que Tutor mostró al finalizar la inicialización.

---

# 20. Cómo iniciar Open edX después de apagar el computador

Este es el procedimiento recomendado.

## Paso 1 — Iniciar Docker Desktop

Abrir Docker Desktop en Windows.

Esperar hasta que Docker indique que está funcionando correctamente.

## Paso 2 — Abrir Ubuntu/WSL

Desde PowerShell:

```powershell
wsl -d Ubuntu
```

o abrir directamente Ubuntu.

## Paso 3 — Ir al proyecto

```bash
cd ~/openedx
```

## Paso 4 — Activar el entorno virtual

```bash
source .venv/bin/activate
```

El prompt debe quedar parecido a:

```text
(.venv) ser@Sebas:~/openedx$
```

## Paso 5 — Comprobar Docker

```bash
docker ps
```

Debe responder correctamente.

## Paso 6 — Iniciar Open edX

Para iniciar los servicios existentes:

```bash
tutor local start
```

## Paso 7 — Comprobar el estado

```bash
tutor local status
```

Todos los servicios principales deberían mostrar:

```text
Up
```

## Paso 8 — Abrir la plataforma

LMS:

```text
http://local.openedx.io
```

Studio:

```text
http://studio.local.openedx.io
```

---

# 21. ¿Cuándo utilizar `tutor local launch`?

`launch` se utiliza para configurar y preparar una plataforma, especialmente durante la instalación inicial.

No es necesario utilizarlo cada vez que se enciende el computador.

Para una plataforma ya configurada:

```bash
tutor local start
```

es suficiente para arrancar los servicios.

---

# 22. Cómo detener Open edX

Para detener la plataforma sin eliminarla:

```bash
tutor local stop
```

Esto detiene los contenedores pero conserva la configuración y los datos.

Para volver a iniciarla:

```bash
tutor local start
```

---

# 23. Reiniciar servicios

Para reiniciar la plataforma:

```bash
tutor local reboot
```

Para reiniciar componentes específicos:

```bash
tutor local restart
```

---

# 24. Ver logs

Estado general:

```bash
tutor local status
```

Logs:

```bash
tutor local logs
```

Logs de un servicio específico pueden consultarse mediante Docker Compose si es necesario.

---

# 25. Ejecutar comandos dentro del LMS

Ejemplo:

```bash
tutor local exec lms bash
```

Para ejecutar Django directamente:

```bash
tutor local exec lms ./manage.py lms shell
```

Ejemplo de comprobación de usuario:

```bash
tutor local exec lms ./manage.py lms shell -c "from django.contrib.auth import get_user_model; print(get_user_model().objects.count())"
```

---

# 26. Crear un superusuario

El comando utilizado/previsto para administrar usuarios es:

```bash
tutor local exec lms ./manage.py lms createsuperuser
```

Si el usuario ya existe, Django devolverá un error de duplicación.

Por ejemplo, durante las pruebas apareció:

```text
IntegrityError: Duplicate entry ... for key 'auth_user.email'
```

Esto significa que el usuario ya estaba registrado y no debe crearse nuevamente.

Puede comprobarse:

```bash
tutor local exec lms ./manage.py lms shell -c "from django.contrib.auth import get_user_model; U=get_user_model(); u=U.objects.get(email='CORREO'); print(u.username, u.is_staff, u.is_superuser, u.is_active)"
```

---

# 27. Estructura conceptual de Open edX

La arquitectura puede resumirse así:

```text
                    Open edX
                       │
          ┌────────────┴────────────┐
          │                         │
         LMS                     Studio/CMS
          │                         │
          └────────────┬────────────┘
                       │
                openedx-platform
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     MySQL          MongoDB         Redis
        │              │
        │              └── contenido de cursos
        │
        └── datos transaccionales

                       │
                      MFE
                       │
                    React

                       │
                  Meilisearch
```

El núcleo `openedx-platform` contiene tanto LMS como CMS/Studio. Open edX ha evolucionado hacia una arquitectura formada por un monolito modular, aplicaciones desplegables independientemente y micro-frontends. 

---

# 28. Arquitectura de curso en Studio

Para la prueba técnica es importante recordar la jerarquía:

```text
Curso
│
├── Sección
│    │
│    ├── Subsección
│    │    │
│    │    ├── Unidad
│    │    │    ├── HTML
│    │    │    ├── Video
│    │    │    ├── Problema
│    │    │    └── Otros componentes
│    │    │
│    │    └── Unidad
│    │
│    └── Subsección
│
└── Sección
```

En una demostración técnica conviene mostrar:

1. Crear curso.
2. Configurar información básica.
3. Crear sección.
4. Crear subsección.
5. Crear unidad.
6. Agregar componente HTML.
7. Agregar un problema.
8. Guardar.
9. Publicar.
10. Ver el contenido desde LMS.

---

# 29. Publicación de contenido

Una modificación en Studio no necesariamente significa que inmediatamente esté publicada para estudiantes.

El flujo conceptual es:

```text
Autor
  │
  ▼
Studio
  │
  ▼
Edición del contenido
  │
  ▼
Guardar
  │
  ▼
Publicar
  │
  ▼
LMS
  │
  ▼
Estudiante
```

Esto es especialmente importante para la prueba técnica porque pueden pedir demostrar la diferencia entre:

- modificar;
- guardar;
- publicar;
- visualizar como estudiante.

---

# 30. Tutor y Docker

Tutor no reemplaza Docker.

La relación es:

```text
Tutor
  │
  └── genera/configura
        │
        ▼
Docker Compose
        │
        ▼
Docker Engine
        │
        ▼
Contenedores Open edX
```

Tutor simplifica:

- configuración;
- generación del entorno;
- imágenes;
- Compose;
- variables;
- servicios;
- inicialización;
- plugins;
- operaciones de administración.

---

# 31. Proyecto Docker de Open edX

El proyecto creado por Tutor se identifica como:

```text
tutor_local
```

Por eso los contenedores tienen nombres como:

```text
tutor_local-lms-1
tutor_local-cms-1
tutor_local-mysql-1
```

Esto permite distinguirlos del contenedor independiente de n8n.

---

# 32. Relación con n8n

El equipo ya tenía un contenedor n8n funcionando.

Open edX fue desplegado mediante otro proyecto Docker:

```text
tutor_local
```

No fue necesario eliminar:

```text
n8n
```

ni realizar:

```bash
docker system prune
docker volume prune
docker container prune
```

Por seguridad, estos comandos no deben ejecutarse como parte del procedimiento normal de Open edX.

---

# 33. Comandos de operación rápida

## Verificar Tutor

```bash
tutor --version
```

## Ver configuración

```bash
tutor config printvalue LMS_HOST
tutor config printvalue CMS_HOST
```

## Iniciar

```bash
tutor local start
```

## Detener

```bash
tutor local stop
```

## Reiniciar

```bash
tutor local reboot
```

## Estado

```bash
tutor local status
```

## Logs

```bash
tutor local logs
```

## Entrar al LMS

```bash
tutor local exec lms bash
```

## Django shell

```bash
tutor local exec lms ./manage.py lms shell
```

## Crear superusuario

```bash
tutor local exec lms ./manage.py lms createsuperuser
```

---

# 34. Procedimiento de recuperación

Si Docker está funcionando pero Open edX no responde:

```bash
cd ~/openedx
source .venv/bin/activate
tutor local status
```

Si los servicios están detenidos:

```bash
tutor local start
```

Volver a comprobar:

```bash
tutor local status
```

Si algún servicio aparece detenido o reiniciándose:

```bash
tutor local logs
```

No eliminar contenedores ni volúmenes antes de revisar los logs.

---

# 35. Si se pierde Internet durante una instalación

Las imágenes Docker que ya fueron descargadas permanecen en el almacenamiento local de Docker.

Por tanto, una interrupción de Internet no implica necesariamente comenzar desde cero.

Para detener de forma segura un proceso interactivo:

```text
Ctrl + C
```

Posteriormente, con Internet restaurado:

```bash
cd ~/openedx
source .venv/bin/activate
tutor local launch
```

Tutor/Docker reutilizarán los recursos que ya estén disponibles y descargarán lo que falte.

---

# 36. Diagnóstico básico

## Docker no responde

Desde Ubuntu:

```bash
docker ps
```

Si falla, verificar Docker Desktop en Windows.

También:

```powershell
wsl --list --verbose
```

Debe aparecer Ubuntu en WSL2 y Docker Desktop funcionando.

## DNS de Ubuntu falla

Comprobar:

```bash
ping -c 3 8.8.8.8
ping -c 3 archive.ubuntu.com
```

Si funciona el primero pero falla el segundo, el problema es DNS.

## Open edX no responde

Ejecutar:

```bash
tutor local status
```

y después:

```bash
tutor local logs
```

---

# 37. Buenas prácticas

1. Mantener Tutor dentro del entorno virtual.
2. No instalar un segundo Docker daemon dentro de Ubuntu.
3. No eliminar volúmenes de Open edX sin conocer su contenido.
4. No utilizar `docker system prune` como solución genérica.
5. Mantener respaldos antes de modificaciones importantes.
6. Utilizar Git para personalizaciones de código.
7. Separar configuración, código y datos.
8. Documentar las versiones de Tutor y Open edX.
9. Utilizar plugins de Tutor para extensiones mantenibles.
10. Evitar modificar directamente archivos internos de los contenedores.

---

# 38. Tecnologías que conviene dominar para la prueba

## Nivel 1 — imprescindible

- Docker
- Docker Compose
- WSL2
- Linux
- Python
- Tutor
- Open edX
- LMS
- Studio/CMS
- MFE
- MySQL
- MongoDB
- Redis

## Nivel 2

- Django
- Celery
- REST API
- OAuth
- Git
- GitHub
- YAML
- Bash
- Docker networking
- Volúmenes Docker

## Nivel 3

- XBlocks
- Tutor plugins
- MFEs
- frontend React
- personalización de temas
- configuración de Open edX
- políticas de autorización

---

# 39. Git y buenas prácticas

Para una modificación de Open edX:

```text
Repositorio upstream
        │
        ▼
      Fork
        │
        ▼
  rama de trabajo
        │
        ▼
      cambio
        │
        ▼
       test
        │
        ▼
      commit
        │
        ▼
       push
        │
        ▼
 Pull Request
```

Buenas prácticas:

- No trabajar directamente sobre `main`.
- Crear ramas descriptivas.
- Hacer commits pequeños.
- Escribir mensajes de commit claros.
- No incluir secretos.
- No subir contraseñas.
- Revisar los cambios antes del commit.
- Mantener el código sincronizado con upstream.

---

# 40. Personalización CSS

Para personalizaciones visuales no se recomienda modificar directamente los archivos internos de un contenedor.

Las personalizaciones deberían gestionarse mediante los mecanismos de temas/extensiones soportados por Open edX/Tutor.

Para una modificación simple del aspecto visual, conviene conocer:

```text
Theme
  │
  ├── templates
  ├── CSS/SCSS
  ├── assets
  └── configuración Tutor
```

El objetivo es que la personalización pueda reproducirse después de reconstruir o actualizar los contenedores.

---

# 41. Plugins y extensiones

Tutor dispone de un sistema de plugins.

Los plugins permiten extender o modificar el despliegue sin editar directamente los archivos generados por Tutor.

Para inspeccionar los plugins instalados:

```bash
tutor plugins list
```

Para consultar ayuda:

```bash
tutor plugins -h
```

Para una prueba técnica, conviene poder explicar:

```text
Plugin
  │
  ├── configuración
  ├── hooks
  ├── imágenes
  ├── Docker Compose
  ├── mounts
  └── servicios
```

Los XBlocks constituyen otra forma de extender Open edX a nivel de componentes educativos. La documentación oficial muestra que los paquetes de XBlock se pueden montar en Tutor e instalar en los contenedores correspondientes. 

---

# 42. Checklist antes de una prueba técnica

## Sistema

- [ ] Windows iniciado.
- [ ] Docker Desktop iniciado.
- [ ] WSL2 funcionando.
- [ ] Ubuntu funcionando.
- [ ] Internet disponible.

## Tutor

```bash
cd ~/openedx
source .venv/bin/activate
tutor --version
```

Debe mostrar:

```text
tutor, version 22.0.2
```

## Docker

```bash
docker ps
```

## Open edX

```bash
tutor local status
```

Todos los servicios deben estar `Up`.

## LMS

```text
http://local.openedx.io
```

## Studio

```text
http://studio.local.openedx.io
```

## Administración

Comprobar que el usuario administrativo funciona.

## Práctica

- [ ] Crear curso.
- [ ] Crear sección.
- [ ] Crear subsección.
- [ ] Crear unidad.
- [ ] Añadir HTML.
- [ ] Añadir video.
- [ ] Añadir problema.
- [ ] Publicar.
- [ ] Ver desde LMS.
- [ ] Editar.
- [ ] Volver a publicar.

---

# 43. Secuencia mínima para el día de la prueba

Abrir Ubuntu:

```bash
cd ~/openedx
source .venv/bin/activate
```

Comprobar:

```bash
docker ps
```

Iniciar:

```bash
tutor local start
```

Comprobar:

```bash
tutor local status
```

Abrir:

```text
http://local.openedx.io
```

y:

```text
http://studio.local.openedx.io
```

Si algo falla:

```bash
tutor local status
```

y:

```bash
tutor local logs
```

No ejecutar una reinstalación completa salvo que realmente sea necesario.

---

# 44. Resultado final

La instalación quedó estructurada de la siguiente manera:

```text
Windows 10
    │
    ├── WSL2
    │     └── Ubuntu 24.04.3
    │           ├── Python 3.12
    │           ├── venv
    │           ├── Tutor 22.0.2
    │           └── Docker CLI
    │
    └── Docker Desktop
          └── Docker Engine
                │
                └── tutor_local
                      ├── LMS
                      ├── Studio/CMS
                      ├── LMS Worker
                      ├── CMS Worker
                      ├── MFE
                      ├── MySQL
                      ├── MongoDB
                      ├── Redis
                      ├── Meilisearch
                      ├── Caddy
                      └── SMTP
```

Estado final verificado:

```text
tutor local status
```

con todos los servicios principales en estado:

```text
Up
```

La plataforma está configurada como instalación local:

```text
LMS:
http://local.openedx.io

Studio:
http://studio.local.openedx.io
```

---

# 45. Referencias oficiales

- Open edX — Platform Architecture
- Open edX — Platform Overview
- Open edX — Installation and Starting Guide
- Open edX — Tutor Installation
- Open edX — Developer Quickstarts
