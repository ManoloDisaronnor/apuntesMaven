# CREAR PROYECTO MAVEN DESDE CERO USANDO LA ESTRUCTURA ELEX

- Primero, **new Maven project** y dejamos el archetype vacío.
- Implementar las librerías necesarias desde Maven Repository.
- Estructura por ahora: `entity`, `repository`, `controller`, `converter`, `service`, `serviceImpl` *(usar VOs en vez de DTO)*.
- Crear paquete principal en el que estará nuestra clase con `main` para ejecutar la aplicación Spring.

## CLASE PRINCIPAL

```java
@SpringBootApplication
public class ElMejorLoginDelMundo {

	public static void main(String[] args) {
		SpringApplication.run(ElMejorLoginDelMundo.class, args);
	}

}
```

## CONFIGURACION DEL POM.XML

```xml
<properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
    <java.version>21</java.version>
	<mainclass>com.soltel.elmejorlogindelmundo.ElMejorLoginDelMundo</mainclass>
</properties>
```

## CONFIGURACION BÁSICA APPLICATION.YML
```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/soltel?useSSL=false&serverTimezone=UTC
    username: root
    password: test
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true  # Para ver las consultas SQL en la consola
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
```

# ENTITIES
Para configurar las entities con JPA:

- Primero, anotar la clase con ```@Entity``` y definir el nombre de la entidad (en este caso, Usuario).
- La clase debe tener los campos de la base de datos y cada columna debe llevar la anotación ```@Column```.

Ejemplo básico de una tabla usuarios:
```java
@Entity(name = "Usuario")
@Table(name = "USUARIO")
public class UsuarioEntity {
	
	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	@Column(name = "ID")
	private Long id;
	
	@Column(name = "EMAIL", nullable = false, unique = true)
	private String email;
	
	@Column(name = "USERNAME", nullable = false, unique = true)
	private String username;
	
	@Column(name = "PASSWORD", nullable = false)
	private String password;
}
```
> [!NOTE]
> La clase debe tener un constructor vacío y sus getters y setters.

# DTOs / VOs
Al mandar datos al cliente, es recomendable no incluir información sensible como la contraseña. Para ello se implementan *DTOs* o *VOs* (versiones reducidas de la clase) para enviarlos en la petición.

Ejemplo de VO de Usuario sin la contraseña:
```java
public class UsuarioVO {
	private Long id;
	private String email;
	private String username;
}
```
> [!IMPORTANT]
> Esta clase debe tener getters, setters, un constructor por defecto y otro con todos los argumentos.

# CONVERTER
El converter es la clase encargada de convertir los VOs a Entity y viceversa.

Ejemplo básico para Usuario:

```java
public class UsuarioConverter {
	
	public static UsuarioVO toVO(UsuarioEntity usuarioEntity) {
		return new UsuarioVO(usuarioEntity.getId(), usuarioEntity.getEmail(), usuarioEntity.getUsername());
	}
	
	public static UsuarioEntity toEntity(UsuarioVO usuarioVO) {
		UsuarioEntity usuarioEntity = new UsuarioEntity();
		usuarioEntity.setId(usuarioVO.getId());
		usuarioEntity.setEmail(usuarioVO.getEmail());
		usuarioEntity.setUsername(usuarioVO.getUsername());
		return usuarioEntity;
	}
}
```

# REPOSITORIES
En el repository se ubican las consultas no implementadas por defecto, por ejemplo, para buscar por una clave única (UK). Se crearán excepciones que luego se manejarán en el controller para enviar errores o confirmar existencia.

### EJEMPLO DE EXCEPCION
```java
public class EmailAlreadyInUse extends RuntimeException {
	
	public EmailAlreadyInUse(String message) {
		super(message);
	}
}
```
Y ahora en el repository mediante esta anotación podremos hacer una consulta que devuelva un boolean para saber si existe o no un campo único:

### INTERFAZ DE REPOSITORY
```java
@Repository
public interface UsuarioRepository extends JpaRepository<UsuarioEntity, Long> {
	UsuarioVO findByEmail(String email);
	UsuarioVO findByUsername(String username);
}
```

# SERVICES
Para el apartado de services, tendremos que crear el service normal que será una interfaz y un serviceImpl,

EJEMPLOS BASICOS:

### INTERFACE SERVICE
```java
public interface UsuarioService {
	UsuarioVO findById();
	
	UsuarioVO save(UsuarioEntity usuarioEntity);
	
	boolean existsByEmail(String email);
	boolean existsByUsername(String username);
}
```

### SERVICEIMPL
En el serviceImpl se maneja la lógica de negocio, por ejemplo, lanzar excepciones si ya existe un usuario con el correo o nombre de usuario.

Ejemplo completo de un service con lógica para verificar existencia (usando ```existsByEmail``` del repository):

```java
@Service
public class UsuarioServiceImpl implements UsuarioService {
	@Autowired
	private UsuarioRepository usuarioRepository;

	public UsuarioServiceImpl(UsuarioRepository usuarioRepository) {
		super();
		this.usuarioRepository = usuarioRepository;
	}
	
	@Override
	public UsuarioVO findById(Long id) throws NotFoundException {
		UsuarioEntity usuarioEntity = usuarioRepository.findById(id).orElse(null);
		
		if (usuarioEntity == null) {
			throw new NotFoundException("No se ha encontrado ningun usuario con el id " + id);
		}
		
		return UsuarioConverter.toVO(usuarioEntity);
	}

	@Override
	public UsuarioVO save(UsuarioEntity usuarioEntity) throws EmailAlreadyInUse, UsernameAlreadyInUse, AlreadyExists  {
		boolean correoExiste = existsByEmail(usuarioEntity.getEmail());
		boolean usernameExiste = existsByUsername(usuarioEntity.getUsername());
		
		if (correoExiste && usernameExiste) {
			throw new AlreadyExists("Un usuario con este correo y nombre de usuario ya esta registrado");
		}
		if (correoExiste) {
			throw new EmailAlreadyInUse("Este correo ya esta asociado a una cuenta");
		}
		if (usernameExiste) {
			throw new UsernameAlreadyInUse("Este nombre de usuario no esta disponible");
		}
		
		BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
		usuarioEntity.setPassword(passwordEncoder.encode(usuarioEntity.getPassword()));
		
		UsuarioEntity usuarioRegistrado = usuarioRepository.save(usuarioEntity);
		
		return UsuarioConverter.toVO(usuarioRegistrado);
	}

	@Override
	public boolean existsByEmail(String email) {
		UsuarioVO usuarioVO = usuarioRepository.findByEmail(email);
		return usuarioVO != null;
	}

	@Override
	public boolean existsByUsername(String username) {
		UsuarioVO usuarioVO = usuarioRepository.findByUsername(username);
		return usuarioVO != null;
	}
}
```

# CONTROLLER

El controller recibe las peticiones de la API. Antes de crear el controller, se crea una clase ```Response``` básica para indicar si la respuesta ha sido correcta o no, con métodos para éxito y error.

### CLASE RESPONSE
```java
public class Response {
	private Object datos;
	private String mensaje;
	private boolean ok;
	private String[] errores;
	
	public Response(Object datos, String mensaje, boolean ok, String[] errores) {
		super();
		this.datos = datos;
		this.mensaje = mensaje;
		this.ok = ok;
		this.errores = errores;
	}

	public static Response exito(Object datos, String mensaje) {
		return new Response(datos, mensaje, true, null);
	}
	
	public static Response error(String mensaje, String[] errores) {
		return new Response(null, mensaje, false, errores);
	}
}
```
> [!NOTE]
> La clase debe tener sus getters y setters.

Y ahora podemos crear el controller, por ahora será un ejemplo simple que maneje excepciones y que solo tenga por ahora un método para buscar un usuario por id.

### EJEMPLO BÁSICO DE CONTROLLER:
```java
@RestController
@RequestMapping("/usuarios")
public class UsuarioController {
	@Autowired
	private UsuarioService usuarioService;
	
	@GetMapping("/getuser/{id}")
    public ResponseEntity<Response> getUser(@PathVariable("id") Long id) {
        try {
            UsuarioVO usuarioVO = usuarioService.findById(id);
            Response response = Response.exito(usuarioVO, "Usuario encontrado correctamente");
            return ResponseEntity.ok(response);
        } catch (NotFoundException e) {
            String[] errores = {"NOT_FOUND"};
            Response response = Response.error("No se ha encontrado el usuario con el id " + id, errores);
            return ResponseEntity.status(HttpStatus.NOT_FOUND).body(response);
        } catch (Exception e) {
            String[] errores = {"UNKNOWN"};
            Response response = Response.error("Error al encontrar el usuario: " + e.getMessage(), errores);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
        }
    }
}
```

# CONFIGURACION ADICIONAL
## APPLICATION.YML
Para poder acceder a la ruta desde Postman, por ejemplo, y configurar la base de datos para que no se creen tablas o campos innecesarios, se añade lo siguiente:

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/soltel?useSSL=false&serverTimezone=UTC
    username: root
    password: test
    driver-class-name: com.mysql.cj.jdbc.Driver

  jpa:
    hibernate:
      ddl-auto: update
      naming:
        implicit-strategy: org.hibernate.boot.model.naming.ImplicitNamingStrategyLegacyJpaImpl
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
    show-sql: true  # Para ver las consultas SQL en la consola
    properties:
      hibernate:
        dialect: org.hibernate.dialect.MySQL8Dialect
        globally_quoted_identifiers: true
        

server:
  servlet:
    context-path: /login
```


## EJECUCIÓN DE PRUEBAS
Ya podríamos ejecutar nuestra aplicación y acceder mediante el puerto 8080 (por defecto) a /login/usuarios/getuser/id y el servidor nos debería de devolver un json parecido a este si existe:

### EJEMPLO DE RESPUESTA EXITOSA
```json
{
    "datos": {
        "id": 1,
        "email": "manuelsantos199811@gmail.com",
        "username": "soymanu17"
    },
    "mensaje": "Usuario encontrado correctamente",
    "ok": true,
    "errores": null
}
```

### EJEMPLO DE RESPUESTA ERRONEA
```json
{
    "datos": null,
    "mensaje": "No se ha encontrado el usuario con el id 2",
    "ok": false,
    "errores": [
        "NOT_FOUND"
    ]
}
```

## LISTA DE DEPENDENCIAS DEL PROYECTO BASE (JDK21)
- spring-boot-starter-web
- spring-boot-starter-data-jpa
- mysql-connector-j
- spring-security-core
- spring-boot-starter-jersey
> [!NOTE]
> Estos tres últimos son opcionales.
- jakarta.json-bind-api
- yasson
- jersey-media-json-binding
