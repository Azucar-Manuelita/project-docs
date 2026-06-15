---
name: django-patterns
description: Guia unica para evaluar la arquitectura actual del backend, enfocada en la estructura real del proyecto, buenas practicas de Python/Django y criterios MUST/SHOULD.
origin: ECC
---

# Guia Unica de Evaluacion del Backend Actual

Esta guia describe como debe evaluarse el backend de este repositorio sin proponer cambios de arquitectura. Se enfoca en la estructura existente, la aplicacion correcta de las capas, las dependencias reales y las buenas practicas del lenguaje.

## Tabla de Contenidos

1. [Alcance](#alcance)
2. [Estructura del Proyecto](#estructura-del-proyecto)
3. [Roles y Responsabilidades](#roles-y-responsabilidades)
4. [Buenas Prácticas](#buenas-prácticas)
5. [Ejemplo End-to-End](#ejemplo-end-to-end)
6. [Dependencias Clave](#dependencias-clave)
7. [Tests y Validación](#tests-y-validación)
8. [Criterios de Evaluación](#criterios-de-evaluación)
9. [Cómo Evaluar Cada Capa](#cómo-evaluar-cada-capa)
10. [Ejemplos Prácticos](#ejemplos-prácticos)
11. [Checklist Pre-Commit](#checklist-pre-commit)
12. [Resumen](#resumen)

---

## Alcance

- No se analiza el entorno virtual.
- No se sugieren cambios de arquitectura.
- No se propone una nueva estructura de carpetas.
- Se asume que la configuracion de contenedor Docker reemplazara el entorno local.

---

## Estructura del Proyecto

El proyecto actual esta organizado en estas unidades logicas:

- **core/urls.py** - enrutamiento principal del proyecto
- **gym_app/models.py** - definicion de entidades y esquema de datos
- **gym_app/repositories/** - acceso a datos y consultas ORM
- **gym_app/services/** - logica de negocio y reglas del dominio
- **gym_app/controllers/** - manejo de HTTP, validacion minima y respuesta
- **gym_app/tests/** - pruebas del backend

Esta estructura es la base que debe analizarse: cada capa tiene un proposito especifico y debe respetar su responsabilidad.

---

## Roles y Responsabilidades

### Controllers

**Responsabilidad principal:**
- Recibir peticiones HTTP
- Validar estructura minima del request
- Traducir datos de request a llamadas de servicio
- Mapear errores a respuestas HTTP

**Instrucciones:**
- No incluir logica de negocio compleja.
- No interactuar directamente con el ORM.
- Devolver codigos HTTP coherentes.
- Delegar el trabajo de dominio a los servicios.

**Ejemplo (ViewSet mínimo):**
```python
from rest_framework import viewsets
from apps.users.serializers import UserSerializer
from apps.users.services.user_service import UserService

class UserViewSet(viewsets.ViewSet):
    service = UserService()

    def list(self, request):
        users = self.service.list_users()
        return Response(UserSerializer(users, many=True).data)

    def create(self, request):
        data = request.data
        user = self.service.create_user(data)
        return Response(UserSerializer(user).data, status=201)
```

---

### Services

**Responsabilidad principal:**
- Contener las reglas del negocio
- Orquestar operaciones entre repositorios
- Validar condiciones de dominio
- Preparar datos para la respuesta

**Instrucciones:**
- No manejar requests ni responses HTTP.
- No leer request.body o request.data.
- Delegar el acceso a datos a los repositorios.
- Lanzar excepciones de dominio cuando corresponda.

**Ejemplo (service):**
```python
from django.db import transaction
from apps.users.repositories.user_repository import UserRepository
from apps.users.domain.models import CreateUserDTO

class UserService:
    def __init__(self):
        self.repo = UserRepository()

    @transaction.atomic
    def create_user(self, payload: CreateUserDTO):
        # validaciones de negocio
        user = self.repo.create(payload)
        # más lógica (ej: enviar email)
        return user

    def list_users(self):
        return self.repo.list()
```

---

### Repositories

**Responsabilidad principal:**
- Encapsular el acceso a datos
- Ejecutar consultas ORM y persistencia
- Abstraer la interaccion con el modelo

**Instrucciones:**
- No incluir reglas de negocio.
- No validar logica de dominio compleja.
- No manejar errores HTTP.
- Regresar objetos de modelo o colecciones de datos.

**Ejemplo (repository):**
```python
from apps.users.domain.models import User

class UserRepository:
    def list(self):
        return User.objects.filter(is_active=True).select_related('profile')

    def create(self, payload):
        return User.objects.create(**payload)

    def get_by_email(self, email):
        return User.objects.filter(email=email).first()
```

---

### Models (Domain)

**Responsabilidad principal:**
- Definir las entidades del dominio
- Declarar campos, relaciones y restricciones de datos
- Aplicar validacion de campo simple

**Instrucciones:**
- Mantener los models como representacion del esquema de datos.
- Contener campos, ForeignKey, ManyToManyField, etc.
- Incluir validadores de campo y metadatos como indexes cuando haga falta.
- No debe conocer sobre HTTP, serializers o repositories concretos.

**Ejemplo (domain model):**
```python
from django.db import models

class User(models.Model):
    email = models.EmailField(unique=True)
    name = models.CharField(max_length=255)
    is_active = models.BooleanField(default=True)

    def deactivate(self):
        self.is_active = False
        self.save(update_fields=['is_active'])
```

---

## Buenas Prácticas

### Desarrollo y Arquitectura

- Use typing en capas `Service` y `Repository` (mypy compatible).
- Documente interfaces públicas con docstrings (Sphinx-friendly).
- Use `serializers` para validación de entrada y salida; use `Service` para validaciones de negocio.
- Use `transaction.atomic` en operaciones que cambian múltiples agregados.
- Centralice la configuración del cache y api clients.

### Python y Django

- Usar nombres descriptivos y consistentes para variables, funciones y clases.
- Mantener funciones pequeñas y con una sola responsabilidad.
- Aplicar type hints en firmas públicas cuando sea posible.
- Escribir docstrings en funciones y clases públicas.
- Evitar logica anidada excesiva.
- Evitar valores magicos en el codigo; usar constantes cuando corresponda.
- No mezclar transporte HTTP con la logica de negocio.
- Evitar consultas ORM directas en controllers.

### Calidad de Código y Clean Code

- Seguir convenciones de estilo Python (PEP 8).
- Preferir claridad sobre concision excesiva.
- Evitar comentarios innecesarios; documentar por que se hace algo, no que se hace.
- Mantener bloques de codigo legibles y sencillos.
- Escribir codigo facil de leer y facil de mantener.
- Fraccionar responsabilidades en funciones y clases pequenas.
- Elegir nombres significativos y evitar abreviaturas oscuras.

### Documentación y Redacción

- Escribir documentos claros y estructurados.
- Usar listas y secciones para facilitar la lectura.
- Mantener un tono directo y concreto.
- Evitar jergas innecesarias.
- Documentar los casos limites y las validaciones importantes.
- Asegurarse de que los mensajes sean faciles de entender.

---

## Ejemplo End-to-End

### Crear Usuario (Resumen del Flujo Completo)

1. **domain/models.py** - define `User` y DTOs (como dataclasses o TypedDict).

2. **repositories/user_repository.py** - encapsula `User.objects.create` y consultas.

3. **services/user_service.py** - valida dominio, usa `transaction.atomic`, invoca `repo.create`.

4. **controllers/user_views.py** - usa `UserSerializer` para parsear request y llama al `UserService`.

La arquitectura completa garantiza separación de responsabilidades en cada nivel.

---

## Dependencias Clave

Para el analisis del backend deben tomarse en cuenta estas dependencias:

- **django** - framework central del proyecto
- **djangorestframework** - si hay serializers, viewsets o APIs REST
- **django-filter** - si se usa filtrado en vistas o servicios
- **pytest** / **pytest-django** - para pruebas automatizadas
- **psycopg2** o **psycopg2-binary** - driver PostgreSQL si esta presente
- **flake8**, **mypy** - calidad de codigo, estilo y tipos si existen

Se debe verificar que las importaciones en el codigo esten alineadas con las dependencias del repositorio.

---

## Tests y Validación

- **Unit tests:** probar `Service` con `Repository` mockeado.
- **Integration tests:** probar `Repository` contra una base de datos de pruebas (pytest + django-db).
- **Cobertura mínima recomendada:** 70% en lógica de negocio.

---

## Criterios de Evaluación

### Reglas de Revisión (MUST / SHOULD)

#### MUST (Bloqueadores)

- Controllers no deben realizar lógica de negocio ni consultas complejas.
- Services no deben hacer llamadas HTTP directas a terceros sin abstracción.
- Repositories no deben contener lógica de negocio.
- Domain no debe importar módulos de infra (views, serializers).
- Falta de manejo de errores y rollbacks en transacciones.

#### SHOULD (Recomendadas)

- Documentar cambios de estado en `Domain` con tests.
- Usar factories/fixtures en tests para `Repository`.
- Mantener servicios pequeños y con una sola responsabilidad.

### MUST Have

- Los controllers validan la entrada y delegan a servicios.
- Los services contienen la logica de negocio y no hacen consultas directas al ORM.
- Los repositories encapsulan las consultas y la persistencia de datos.
- Los models son esquema y validacion de campo, no reglas de negocio.
- Las funciones publicas usan type hints cuando sea posible.
- Las excepciones se manejan y se traducen a respuestas HTTP adecuadas.
- Existen tests para servicios y repositories.
- No hay logica de negocio en gym_app/controllers/.
- No hay acceso ORM directo desde gym_app/controllers/.
- El codigo mantiene nombres legibles y evita abreviaturas oscuras.

### SHOULD Have

- Los controllers incluyen logging basico en puntos criticos.
- Los services usan validadores reutilizables cuando correspondan.
- Los repositories usan metodos bien nombrados y evitan duplicacion.
- Los models declaran indices o constraints cuando la estructura lo requiera.
- Los tests cubren casos de error y tienen nombres descriptivos.
- El proyecto documenta las dependencias usadas en un archivo de requirements o similar.
- El codigo sigue un estilo Python consistente.
- La documentacion es clara, organizada y util.
- Los mensajes de error y validacion son precisos y expresivos.
- Las pruebas estan redactadas para describir la intencion.

## Cómo Evaluar Cada Capa

### Evaluación de Controllers

**Buscar:**
- json.loads(request.body) o request.data solo para parsing.
- validacion de campos requeridos.
- llamadas explicitas a ..._service o ...Service.
- traduccion de errores a JsonResponse, Response, status o codigos HTTP.

**No es deseable:**
- llamadas directas a Model.objects o QuerySet.
- logica de negocio junto a parseo de request.
- construccion de respuestas complejas dentro del controller.

### Evaluación de Services

**Buscar:**
- validaciones de dominio.
- orquestacion entre repositories.
- transformaciones de datos antes de devolver resultados.
- lanzamiento de excepciones de negocio.

**No es deseable:**
- uso de request, request.body, request.data.
- creacion de respuestas HTTP.
- importaciones directas del ORM para consultas en lugar de hacerlo a traves de repositorios.

### Evaluación de Repositories

**Buscar:**
- metodos de consulta y persistencia con ORM.
- get, filter, select_related, prefetch_related.
- encapsulacion de consultas recurrentes.

**No es deseable:**
- validaciones de negocio complejas.
- decisiones de flujo basadas en logica de dominio.
- uso de request.

### Evaluación de Models

**Buscar:**
- definicion de campos y relaciones.
- validadores de campo simples.
- metadatos Meta con indexes, db_table, ordering.

**No es deseable:**
- logica de negocio que deberia estar en services.
- llamadas a repositories o services.
- manejo de HTTP o requests.

---

## Ejemplos Prácticos de Evaluación

### Caso 1: Controller con Query Directo

Si un controller tiene `User.objects.filter(...)`, es una violacion de separacion de capas.

### Caso 2: Service que usa Request Data

Si un service lee `request.data`, se considera mezcla de responsabilidades.

### Caso 3: Repository Validando Reglas de Negocio

Si un repository decide si un usuario es elegible segun reglas de negocio complejas, es error de capa.

### Caso 4: Model con Validación de Campo

Si un model usa `models.CheckConstraint(...)` o valida un campo con `validators`, es un uso correcto de la capa de datos.

---

## Checklist Pre-Commit

### Arquitectura
- [ ] Controllers delegan a Services
- [ ] Services usan Repositories/Domain
- [ ] Repositories encapsulan ORM y optimizaciones
- [ ] No hay lógica de negocio en Models

### Calidad
- [ ] Tipado en funciones públicas (Service/Repository)
- [ ] Manejo de transacciones y errores
- [ ] Tests unitarios para Service y Repository
- [ ] No logs de debug en producción
- [ ] Documentación clara de interfaces públicas

### Testing
- [ ] Tests cubren casos de éxito y error
- [ ] Coverage mínimo 70% en lógica de negocio
- [ ] Pruebas nombradas descriptivamente
- [ ] Integration tests para Repositories

### Código
- [ ] Sigue PEP 8
- [ ] Nombres significativos
- [ ] Sin abreviaturas oscuras
- [ ] Funciones pequeñas y enfocadas
- [ ] Documentación completa

---

## Resumen

Este archivo es la guia principal de evaluacion para el backend actual. Debe mantenerse simple, expresivo y directo. El analisis debe centrarse en si el codigo respeta la estructura definida y si aplica correctamente las responsabilidades de cada capa.

**Mantener este SKILL como la guía canonical para revisiones de arquitectura en proyectos Django.**
