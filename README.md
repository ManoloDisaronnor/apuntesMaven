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

