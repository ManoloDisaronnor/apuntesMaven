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



