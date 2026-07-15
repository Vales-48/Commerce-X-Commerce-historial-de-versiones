# Commerce X Commerce (CxC)

Plataforma de ecommerce construida como arquitectura de microservicios independientes, desarrollada para el Examen Final Transversal de **Desarrollo FullStack 1 (DSY1103)**, DuocUC.

Cada microservicio tiene su propia base de datos, expone una API REST documentada con Swagger/OpenAPI, y se comunica con los demás mediante Feign Client cuando lo necesita. Un API Gateway (Spring Cloud Gateway) centraliza el acceso a todos los servicios.

## Integrantes

- Lucas Sepúlveda
- Valentín Salfate

---

## Arquitectura del sistema

| Microservicio | Puerto | Base de datos | Responsabilidad |
|---|---|---|---|
| Seguridad | 8081 | `seguridad_cxc` | Registro, login y emisión de JWT |
| Productos | 8082 | `productos_cxc` | Catálogo de productos (CRUD) |
| CargaMasiva | 8083 | `productos_cxc` (compartida con Productos) | Carga masiva de productos en lote |
| Clientes | 8084 | `clientes_cxc` | Datos de clientes (CRUD) |
| Pedidos | 8085 | `pedidos_cxc` | Creación y seguimiento de pedidos |
| Carrito | 8086 | `carrito_cxc` | Carrito de compras previo a un pedido |
| Envíos | 8088 | `envios_cxc` | Despacho y seguimiento de entregas |
| Notificaciones | 8089 | `notificaciones_cxc` | Envío simulado de notificaciones a clientes |
| Categorías | 8090 | `categorias_cxc` | Catálogo de categorías de productos |
| Reseñas | 8091 | `resenas_cxc` | Calificaciones y comentarios sobre productos |
| **Gateway** | **8080** | — | Punto de entrada único a todos los microservicios |

**10 microservicios de negocio + Gateway = 11 servicios en total.**

> Nota sobre CargaMasiva: comparte intencionalmente la base de datos con Productos — hace inserciones en lote directo contra la misma tabla `productos`, optimizado con `EntityManager` (flush/clear cada 50 registros) en vez de pasar por la API REST de Productos.

### Quién le habla a quién (comunicación entre microservicios vía Feign)

- **Pedidos** → valida el cliente contra **Clientes** y cada producto contra **Productos**, y calcula el total con el precio real (nunca confía en el precio que mande quien hace la petición).
- **Carrito** → valida cada producto contra **Productos** al agregarlo al carrito.
- **Envíos** → valida que el pedido exista contra **Pedidos** antes de crear el envío.
- **Reseñas** → valida producto y cliente contra **Productos** y **Clientes** antes de guardar la reseña.
- **Notificaciones** y **Categorías** no dependen de otros microservicios (decisión de diseño para no acoplar su disponibilidad a la de otros servicios).

### Relaciones JPA (`@OneToMany` / `@ManyToOne`) dentro de un mismo microservicio

- **Pedidos**: `Pedido` → `DetallePedido` (líneas del pedido).
- **Carrito**: `Carrito` → `ItemCarrito` (líneas del carrito).
- **Envíos**: `Envio` → `SeguimientoEnvio` (historial de estados, se genera automáticamente en cada cambio de estado).
- **Reseñas**: `Resena` → `RespuestaResena` (respuestas del vendedor a una reseña).

### Nota sobre seguridad

El microservicio de Seguridad emite y valida JWT (login/registro), pero por alcance del proyecto esa validación **no está propagada** todavía a los demás microservicios (Productos, Pedidos, etc. no exigen el token en sus endpoints). Si se quiere security end-to-end, el siguiente paso sería agregar un filtro JWT (o validarlo en el Gateway) que revise el header `Authorization` antes de enrutar hacia cada servicio.

---

## Requisitos previos

- Java 21
- Maven 3.9+
- Docker y Docker Compose (para levantar todo el stack de una vez)
- Una herramienta de pruebas REST como Postman o Insomnia

---

## Cómo ejecutar el proyecto

### Opción 1: Docker Compose

```bash
docker-compose up --build
```

Esto levanta MySQL (con las 9 bases de datos ya creadas vía `docker/init-db.sql`), los 10 microservicios de negocio y el Gateway. Cuando todos los contenedores estén arriba, todo se accede a través del Gateway en `http://localhost:8080`.

Para bajar todo:
```bash
docker-compose down
```

Para bajar todo y borrar también los datos de MySQL:
```bash
docker-compose down -v
```

### Opción 2: Ejecución local individual

1. Levantar una instancia de MySQL local en el puerto 3306, usuario `root` sin contraseña (o ajustar `application.yml` de cada servicio).
2. Crear las bases de datos manualmente (ver `docker/init-db.sql`) o dejar que Flyway las use si ya existen.
3. Pararse dentro de la carpeta del microservicio y ejecutar:
   ```bash
   cd Microservicios/<Nombre>/ejemplos/<carpeta>
   mvn spring-boot:run
   ```
   (o correr la clase `*Application.java` directamente desde IntelliJ/VS Code).
4. Cada microservicio corre en su puerto individual (ver tabla de arquitectura). El Gateway asume por defecto que todos corren en `localhost`.

**Orden recomendado si se levanta manualmente uno por uno** (para que las validaciones vía Feign no fallen por dependencias que aún no están arriba):

1. Seguridad, Productos, Clientes, Categorías, Notificaciones (no dependen de nadie más)
2. CargaMasiva (depende de la base de datos de Productos)
3. Carrito, Reseñas (dependen de Productos, y Reseñas también de Clientes)
4. Pedidos (depende de Productos y Clientes)
5. Envíos (depende de Pedidos)
6. Gateway (al final, ya que enruta hacia todos los anteriores)


### Ejecutar los tests

Dentro de cada microservicio:
```bash
mvn clean test
```

---

## Documentación Swagger (por microservicio, en local)

**Importante:** la UI de Swagger de este proyecto vive en una ruta personalizada, **no** en la default de springdoc. Si entran a `/swagger-ui.html` o `/swagger-ui/index.html` les va a dar 404 aunque el microservicio esté corriendo bien — esa es la causa más probable de que "Productos no funcionaba": la ruta correcta es **`/doc/swagger-ui.html`**.

| Microservicio | Swagger UI (ruta correcta) |
|---|---|
| Seguridad | http://localhost:8081/doc/swagger-ui.html |
| Productos | http://localhost:8082/doc/swagger-ui.html |
| CargaMasiva | http://localhost:8083/doc/swagger-ui.html |
| Clientes | http://localhost:8084/doc/swagger-ui.html |
| Pedidos | http://localhost:8085/doc/swagger-ui.html |
| Carrito | http://localhost:8086/doc/swagger-ui.html |
| Envíos | http://localhost:8088/doc/swagger-ui.html |
| Notificaciones | http://localhost:8089/doc/swagger-ui.html |
| Categorías | http://localhost:8090/doc/swagger-ui.html |
| Reseñas | http://localhost:8091/doc/swagger-ui.html |

Si aun así no carga, revisar en ese orden: 1) que el microservicio efectivamente esté corriendo (log de consola sin errores), 2) que estén pegándole al puerto correcto, 3) que `mvn clean install` haya bajado bien la dependencia `springdoc-openapi-starter-webmvc-ui` (revisar que no haya quedado a medio descargar).

### Para importar en Postman

Cada microservicio expone su spec OpenAPI en JSON en `/v3/api-docs` (ej. `http://localhost:8082/v3/api-docs` para Productos). En Postman: **Import → Link → pegar esa URL** — genera automáticamente la colección completa con todos los endpoints, sin tener que armarlos a mano.

---

## Rutas principales del Gateway

Todo pasa por `http://localhost:8080`, el Gateway redirige según el path:

| Ruta en el Gateway | Va hacia |
|---|---|
| `/auth/**` | Seguridad (login, registro) |
| `/admin/**` | Seguridad (endpoints solo ROLE_ADMIN) |
| `/api/v1/productos/**` | Productos |
| `/api/productos/**` | CargaMasiva |
| `/api/v1/clientes/**` | Clientes |
| `/api/v1/pedidos/**` | Pedidos |
| `/api/v1/carritos/**` | Carrito |
| `/api/v1/envios/**` | Envíos |
| `/api/v1/notificaciones/**` | Notificaciones |
| `/api/v1/categorias/**` | Categorías |
| `/api/v1/resenas/**` | Reseñas |

---

## Flujo de uso de ejemplo (de principio a fin)

Un recorrido típico por el sistema, todo a través del Gateway (`http://localhost:8080`). Los ejemplos usan `curl`, pero funcionan igual en Postman.

**1. Registrarse y loguearse**
```bash
curl -X POST http://localhost:8080/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"The-Admin","password":"OSU","role":"ROLE_ADMIN"}'

curl -X POST http://localhost:8080/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"The-Admin","password":"OSU"}'
# -> devuelve { "token": "eyJ..." }
```

**2. Crear una categoría**
```bash
curl -X POST http://localhost:8080/api/v1/categorias \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Perifericos","descripcion":"Mouse, teclados y mas"}'
```

**3. Crear un producto**
```bash
curl -X POST http://localhost:8080/api/v1/productos \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Mouse Gamer","categoria":"Perifericos","precio":15990,"codigo":"PER-001","stock":50}'
```

**4. Crear un cliente**
```bash
curl -X POST http://localhost:8080/api/v1/clientes \
  -H "Content-Type: application/json" \
  -d '{"nombre":"vale","apellido":"sepu","correo":"salfa@correo.com","telefono":"+56912345678","rol":"CLIENTE"}'
```

**5. Armar un carrito y agregar un producto**
```bash
curl -X POST http://localhost:8080/api/v1/carritos \
  -H "Content-Type: application/json" -d '{"clienteId":1}'
# -> anota el "id" del carrito que devuelve

curl -X POST http://localhost:8080/api/v1/carritos/1/items \
  -H "Content-Type: application/json" \
  -d '{"productoId":1,"cantidad":2}'
```

**6. Crear el pedido** (Carrito y Pedidos son independientes en este proyecto — el pedido se arma pasando directamente los productos, no se "convierte" el carrito automáticamente vía API)
```bash
curl -X POST http://localhost:8080/api/v1/pedidos \
  -H "Content-Type: application/json" \
  -d '{"idCliente":1,"detalles":[{"idProducto":1,"cantidad":2}]}'
```

**7. Confirmar el pedido y crear su envío**
```bash
curl -X PUT http://localhost:8080/api/v1/pedidos/1/estado \
  -H "Content-Type: application/json" -d '{"nuevoEstado":"CONFIRMADO"}'

curl -X POST http://localhost:8080/api/v1/envios \
  -H "Content-Type: application/json" \
  -d '{"idPedido":1,"direccionEntrega":"Av. Siempre Viva 742"}'
```

**8. Avanzar el envío (esto agrega automáticamente un registro al historial)**
```bash
curl -X PUT http://localhost:8080/api/v1/envios/1/estado \
  -H "Content-Type: application/json" \
  -d '{"nuevoEstado":"EN_RUTA","comentario":"Salio de bodega central"}'
```

**9. Notificar al cliente**
```bash
curl -X POST http://localhost:8080/api/v1/notificaciones \
  -H "Content-Type: application/json" \
  -d '{"idCliente":1,"tipo":"EMAIL","asunto":"Tu pedido va en camino","mensaje":"Tu pedido #1 esta en ruta"}'

curl -X POST http://localhost:8080/api/v1/notificaciones/1/enviar
```

**10. Dejar una reseña del producto**
```bash
curl -X POST http://localhost:8080/api/v1/resenas \
  -H "Content-Type: application/json" \
  -d '{"idProducto":1,"idCliente":1,"calificacion":5,"comentario":"Excelente mouse"}'

curl http://localhost:8080/api/v1/resenas/producto/1/promedio
```

Con esto ya se recorrió el flujo completo por los 10 microservicios de negocio: seguridad → categoría → producto → cliente → carrito → pedido → envío → notificación → reseña.

---

## Estructura del repositorio

```
Microservicios/
  Seguridad/ejemplos/productos/
  Productos/ejemplos/productos/
  CargaMasiva/ejemplos/productos/
  Cliente/ejemplos/productos/
  Pedidos/ejemplos/pedidos/
  Carrito/ejemplos/carrito/
  Envios/ejemplos/envios/
  Notificaciones/ejemplos/notificaciones/
  Categorias/ejemplos/categorias/
  Resenas/ejemplos/resenas/
  Gateway/ejemplos/gateway/
docker-compose.yml
docker/init-db.sql
```

Cada carpeta de microservicio es un proyecto Maven independiente (su propio `pom.xml`, `Dockerfile` y `application.yml` con perfiles `dev`/`prod`).

---

## Despliegue remoto

Cada microservicio ya tiene su `Dockerfile` listo para desplegarse en Railway, Render u otra plataforma similar. Variables de entorno que necesita cada uno en producción (perfil `prod`):

- Todos: `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `PORT`
- Pedidos: además `PRODUCTOS_SERVICE_URL`, `CLIENTES_SERVICE_URL`
- Carrito: además `PRODUCTOS_SERVICE_URL`
- Envíos: además `PEDIDOS_SERVICE_URL`
- Reseñas: además `PRODUCTOS_SERVICE_URL`, `CLIENTES_SERVICE_URL`
- Gateway: `SEGURIDAD_URL`, `PRODUCTOS_URL`, `CARGAMASIVA_URL`, `CLIENTES_URL`, `PEDIDOS_URL`, `CARRITO_URL`, `ENVIOS_URL`, `NOTIFICACIONES_URL`, `CATEGORIAS_URL`, `RESENAS_URL`
