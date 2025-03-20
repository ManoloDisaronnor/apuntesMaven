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

- Primero, anotar la clase con @Entity y definir el nombre de la entidad (en este caso, Usuario).
- La clase debe tener los campos de la base de datos y cada columna debe llevar la anotación @Column.

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
![NOTE] [La clase debe tener un constructor vacío y sus getters y setters.]

