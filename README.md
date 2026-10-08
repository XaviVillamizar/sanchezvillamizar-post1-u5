# Post-contenido — Unidad 5: Integración en Aplicaciones Web

## Descripción
Repositorio del post-contenido de la Unidad 5 de Patrones de Diseño de Software.
Un único proyecto Spring Boot (`reservas-labs-api`) para la reserva de
laboratorios de cómputo, con dos partes: una API REST con arquitectura en capas
(Entity, Repository, Service, Controller) sobre H2, y una vista Thymeleaf (MVC
clásico) que reutiliza el mismo `ReservaService`.

## Arquitectura en capas

```
src/main/java/com/universidad/reservaslabs/
├── model/        Laboratorio, Reserva, EstadoReserva (entidades JPA)
├── repository/   LaboratorioRepository, ReservaRepository (Spring Data JPA)
├── service/      ReservaService (reglas de negocio)
├── controller/   LaboratorioController, ReservaController (REST)
├── web/          ReservaWebController, ReservaWebExceptionHandler (MVC)
└── exception/    ReservaConflictException, ReservaInvalidaException,
                  RecursoNoEncontradoException, GlobalRestExceptionHandler

src/main/resources/
├── application.properties
└── templates/reservas/   lista.html, nueva.html
```

## Cómo ejecutar

```
mvn clean package
mvn spring-boot:run
```

- API REST: http://localhost:8080/api/reservas
- Vista MVC: http://localhost:8080/reservas
- Consola H2: http://localhost:8080/h2-console (JDBC URL `jdbc:h2:mem:reservas_labs_db`, usuario `sa`, sin contraseña)

## Endpoints REST

| Método | Ruta | Descripción | Respuestas |
|---|---|---|---|
| GET | `/api/laboratorios` | Lista laboratorios | 200 |
| GET | `/api/laboratorios/{id}` | Obtiene un laboratorio | 200 / 404 |
| POST | `/api/laboratorios` | Crea un laboratorio | 201 / 400 |
| GET | `/api/reservas` | Lista reservas | 200 |
| GET | `/api/reservas/{id}` | Obtiene una reserva | 200 / 404 |
| GET | `/api/reservas/laboratorio/{id}` | Reservas de un laboratorio | 200 |
| POST | `/api/reservas` | Crea una reserva | 201 / 400 / 404 / 409 |
| DELETE | `/api/reservas/{id}` | Cancela una reserva | 204 / 404 / 409 |

## Rutas MVC

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/reservas` | Lista de reservas |
| GET | `/reservas/nueva` | Formulario de nueva reserva |
| POST | `/reservas` | Crea la reserva |
| POST | `/reservas/{id}/cancelar` | Cancela la reserva |

## Reglas de negocio (`ReservaService`)

1. **Solapamiento:** no se puede reservar un laboratorio en un horario que se
   cruce con otra reserva activa del mismo laboratorio (409).
2. **Horario de atención:** entre 07:00 y 21:00 (400).
3. **Duración:** entre 30 minutos y 3 horas (400).
4. **Cancelación tardía:** no se cancela una reserva cuyo inicio ya pasó (409).

## Decisiones de diseño

### Punto de decisión 1 — Ubicación de la validación de solapamiento
El filtrado de solapamientos vive en una consulta JPQL del Repository
(`ReservaRepository.buscarSolapamientos`), mientras que la decisión de rechazar
la reserva vive en `ReservaService.crear`. La alternativa descartada era traer
todas las reservas del laboratorio a memoria y compararlas con Java puro, lo cual
no escala: la tabla de reservas crece sin límite con el tiempo y cada creación
cargaría el historial completo. Filtrando en SQL solo viajan las reservas que
realmente se cruzan. El Repository responde una pregunta de datos ("¿qué
reservas se solapan con este rango?") y el Service una pregunta de negocio ("¿se
permite crear esta reserva?"). Si el Controller llamara directamente a
`buscarSolapamientos()` sin pasar por el Service, la decisión de rechazo y el
mensaje de conflicto quedarían en la capa HTTP, mezclando reglas de dominio con
transporte y obligando a duplicarlas en cualquier otra superficie (como la vista
MVC).

### Punto de decisión 2 — Reglas con y sin apoyo del Repository
`validarHorarioYDuracion` vive íntegramente en el Service, con Java puro y sin
tocar el Repository, porque solo depende de los campos de la reserva que se está
creando (inicio, fin), sin consultar ninguna otra fila. El criterio general es:
si la regla necesita comparar contra datos que solo la base de datos conoce
(otras reservas existentes), conviene apoyarse en una consulta del Repository;
si solo depende del propio objeto que se valida, involucrar al Repository sería
un viaje innecesario a la base de datos. Estas reglas lanzan
`ReservaInvalidaException` (400), distinguiéndolas del conflicto con datos
existentes, que lanza `ReservaConflictException` (409). Además, `ReservaService`
no es un Service anémico: aporta cuatro reglas propias, y por eso se justifica
como capa. Por el mismo criterio, `LaboratorioController` usa el Repository
directamente: el catálogo de laboratorios es un CRUD sin reglas de negocio, y
un `LaboratorioService` que solo delegara sería el antipatrón de Service
anémico.

### Punto de decisión 3 — Cómo comparten Service el Controller MVC y el REST
`ReservaController` (REST) y `ReservaWebController` (MVC) reciben por
constructor la misma clase `ReservaService`, gestionada por Spring como un único
bean singleton. Ninguno reimplementa la validación de solapamiento ni la de
horario. La alternativa descartada era copiar la lógica en el controlador MVC o
crear un `ReservaWebService` casi idéntico: corregir una regla en el futuro
obligaría a cambiarla en dos lugares, que es justo el problema que la capa
Service existe para evitar. La inyección se ve en el constructor de
`ReservaController` (`ReservaService service`) y en el de `ReservaWebController`
(`ReservaService service`, más `LaboratorioRepository` solo para cargar el combo
de laboratorios).

### Punto de decisión 4 — Manejo de errores consistente entre MVC y REST
Se usan dos manejadores de excepciones: `GlobalRestExceptionHandler`
(`@RestControllerAdvice(annotations = RestController.class)`) devuelve JSON con
código HTTP, y `ReservaWebExceptionHandler`
(`@ControllerAdvice(assignableTypes = ReservaWebController.class)`) redirige al
formulario con un mensaje flash. Ambos parten del mismo vocabulario de
excepciones de dominio (`ReservaConflictException`, `ReservaInvalidaException`,
`RecursoNoEncontradoException`), por lo que el mensaje de negocio es el mismo en
ambas superficies y solo cambia la presentación. Se descartó un único
`@RestControllerAdvice` global porque siempre serializa a JSON, y una página
Thymeleaf necesita una redirección con mensaje legible. También se descartó un
único manejador que detecte el tipo de cliente (por ejemplo, inspeccionando el
header `Accept`), porque añadiría una rama condicional por cada excepción; dos
clases acotadas mantienen una responsabilidad por superficie de presentación.

## Herramientas utilizadas
- Java 17, Spring Boot 3.2, Spring Data JPA, H2, Thymeleaf, Lombok
- Apache Maven, curl, Git, GitHub, VS Code

## Conclusiones
Este post-contenido mostró que una arquitectura en capas no se trata de seguir
una plantilla mecánica, sino de decidir dónde vive cada responsabilidad: el
Repository responde preguntas de datos, el Service toma decisiones de negocio y
los controladores solo traducen entre HTTP o vistas y el dominio. Lo más difícil
fue decidir dónde ubicar cada regla, y el criterio que mejor funcionó fue si la
regla necesita datos externos al objeto (Repository) o solo depende del propio
objeto (Service). También quedó claro que reutilizar un único `ReservaService`
en la API REST y en la vista Thymeleaf evita duplicar reglas, y que un mismo
vocabulario de excepciones puede presentarse distinto (JSON o redirección) sin
repetir la lógica de negocio. Finalmente, evitar el Service anémico y no crear un
`LaboratorioService` innecesario enseñó que cada capa debe justificar su
existencia.