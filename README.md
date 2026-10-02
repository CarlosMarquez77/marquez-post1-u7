# Post-contenido — Unidad 7: Patrones Arquitectónicos I

## Descripción
Repositorio del post-contenido de la Unidad 7 de Patrones de Diseño
de Software. Un unico proyecto Spring Boot (multas-biblioteca-api)
para la gestion de multas de biblioteca, con dos partes: una API REST
en capas (Model, Repository, Service, Controller) sobre H2, y pago en
linea de multas con dos pasarelas intercambiables.

## Parte 1 — Arquitectura en Capas
MultaRepository extiende JpaRepository y agrega una consulta
agregada (countByEstudianteIdAndEstado). MultaService concentra las
reglas de negocio (limite de multas pendientes, generacion); el
calculo del monto vive en la propia entidad Multa
(Multa.calcularMonto). MultaController expone /api/multas. Ver
paquetes model/, repository/, service/ y controller/.

```
marquez-post1-u7/
└── multas-biblioteca-api/
    ├── pom.xml
    └── src/main/java/com/example/multas/
        ├── controller/     ← Capa Presentación (@RestController)
        ├── service/        ← Capa Aplicación (@Service)
        ├── model/          ← Capa Dominio (entidades, excepciones de negocio)
        ├── repository/     ← Capa Infraestructura (@Repository)
        └── MultasApplication.java
```

El controlador nunca usa el repositorio directamente, siempre pasa por
MultaService.

## Cómo ejecutar
```
$ cd multas-biblioteca-api && mvn spring-boot:run
```

## Herramientas utilizadas
- Java 17, Spring Boot 3.x, Spring Data JPA, H2
- Apache Maven, Postman/curl, Git, GitHub

## Decisiones de diseño

### Punto de decisión 1 — Cálculo del monto: ¿entidad o Service?
`Multa.calcularMonto` está en la entidad porque el monto solo depende
de los días de atraso que recibe como parámetro (500 por día con tope
de 15.000). El criterio fue: si la regla no necesita ningún colaborador
externo (Repository u otro Service), puede vivir en el propio objeto de
dominio, porque describe cómo se comporta una multa y no cómo se
orquesta un caso de uso.

La alternativa era un método privado en MultaService. Funcionaría, pero
Multa quedaría como un contenedor de datos sin comportamiento (modelo
anémico) y cualquier otra parte que necesitara recalcular un monto
tendría que duplicar la fórmula o pasar por el Service sin necesidad.

### Punto de decisión 2 — Conteo de multas pendientes: ¿consulta o filtrado en memoria?
Contar las multas pendientes de un estudiante sí necesita datos de la
base de datos. Se usa `countByEstudianteIdAndEstado`, que Spring Data
traduce a un COUNT en SQL. La decisión de negocio (si se permite generar
otra multa) la toma MultaService, pero el dato se calcula en la base de
datos, que devuelve solo un número.

La alternativa era traer todas las multas con `findByEstudianteId` y
filtrarlas con streams. Con miles de multas por estudiante, cada multa
nueva cargaría todo el historial en memoria y el tiempo de respuesta
crecería con ese historial.
