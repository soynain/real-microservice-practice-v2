# real-microservice-practice-v2
En esta práctica a diferencia de la anterior, si lo haré lo más realisa posible, no mostrare todo pero si la idea con capturas,
hasta el deployment.

Los objetivos son: 

*aprender a implementar tracing entre dos micros.*

*Wrapper genérico con ayuda de un jar generado externamente*

*Implementación de logs con elastic search*

*Aprender del framework Quarkus y ver como nos ayuda a generar el deploy más rápido en AWS*

*Jenkins*

1.Configuración del pipe con docker-compose
2.Fases de pipe (Testing, validación de cobertura con Jacoco y sonarqube, análisis de CWE's, empaquetado y publicación en AWS ECR con EKS o sin EKS)

Con todo esto coleccionado, ya implementamo algo más real.

## Avances 21/01/2025
**1. Creación de los micros y de un wrapper customizado para los responses (de acuerdo a Quarkus)**

  **1.1. Comprensión del framework quarkus**
    Que dolor de.... de verdad, yo creo es por mi inglés pero, me costó demasiado descubrir como madres haer un simple build
    en docker, solo para probar como funcionaba.

  Multistages son para builds universales en windows, el resto son para escenarios cocnretos. También puedes aplicar
  la de spring boot que es solo copiar y pegar el jar.
  
  **1.2 Creación del micro a y b con arquitectura hexagonal**
  En spring boot se acopla mejor el concepto de, la mvc por capas, para eso fue diseñado. Sin embargo
  quarkus sirve más con arquitectura DDD, la cual alguna vez vi en unos códigos de migraciones.

  La arquitectura consiste en 3 capichis: Infraestructura, dominio y aplicación.
  En infra defines controladores, repositorios e incluso eventos a mandar por rabbit o kafka.
  En aplicación defines casos de uso, son clases de Single responsability
  con un nombre de clase bien especificado y un método de acuerdo a ese caso de uso.
  Por último, dominio son tus pueertos que vendrian a ser tu clases con las que
  guardas en bdd o mandas a una SQS, y tambien ahí defines tus DTOs y modelos.

  En losd puertos defines lo que un sistema requiere hacer como guardar algo en base de datos

  <img width="1602" height="1054" alt="image" src="https://github.com/user-attachments/assets/a19c09da-dca3-44ff-98b1-775ebfdcde27" />

  o mandar un mensaje,en infraestructura defines como se logra ese objetivo
  en tus ports. Sirve para que cuando tu mockees, no andes probando la implementación de infraestructuras.
  Solo mockeas y ya.

  Ahora el prox desafio es ver como funcionan las dependencias de quarkus. Hoy perdi bastantito tiempo
  por lo del docker.

  **1.2.1 Conexión entre dos micros con quarkus**
  Que dolor de... pero bueno, se logró. Les pasaré unos tips a los principiantes como yo que anden checando
  apenas este framework: 

  Esta docu está mal en la parte de implementar un builder con quarkus para los url's (tratando de imitar los resttemplates de spring)

  <img width="1613" height="1142" alt="image" src="https://github.com/user-attachments/assets/2196ee01-4d13-460c-8229-fe80fada9b95" />

  No tienes que declarar dos veces Path en la implementación del resource externo y de la interfaz, porque eso corrompe tu path,
  para descubrirlo tuve que hacer un interceptor para validar eso. De la docu no hay que fiarse y no entendía el porqué,ya que me salía
  404.

  Se implementa así:
  ```Java
  @ApplicationScoped
  public class UserRegistrationClientImpl {
      
      private UserRegistrationClient regClient;
  
      @ConfigProperty(name="app.domains.microB.uri")
      private String uri;
  
      
  
      @PostConstruct
      public void init() {
          this.regClient = QuarkusRestClientBuilder.newBuilder()
              .baseUri(URI.create(new String(this.uri+"")))
              .register(new ClientLoggingFilter())
              .build(UserRegistrationClient.class);
      }
  
      @POST
      public UserRegistrationResponse registerUser(UserRegistrationRequest user){
          return regClient.registerUser(user);
      }
  }

   @Path("/microB")
   public interface UserRegistrationClient {
   
       @POST
       @Path("/user")
       UserRegistrationResponse registerUser(UserRegistrationRequest user);
   }
  
   public class ClientLoggingFilter implements ClientRequestFilter {
      @Override
      public void filter(ClientRequestContext requestContext) throws IOException {
          // TODO Auto-generated method stub
          System.out.println("FINAL URI = " + requestContext.getUri());
      }
  }
  ```

  Tomen en cuenta esos detalles si tratan de guiarse al pie de la letra de la documentación. Ahora, noté que
  mi microB no está guardando mis registros en base de datos.

  Ya guarda esta cosa

  <img width="1137" height="266" alt="image" src="https://github.com/user-attachments/assets/e00b74c6-b540-4f62-9434-f3f493c4d890" />

  De punto A a punto B ya llamamos correctamente.

  Recuerda que si usas @Entity, tienes que especificar la tabla, y aparte si no configuras bien tu Autoincrement, te crea
  una tabla de secuencia.

  Para conectar dos micros de quarkus, requieres usar el QuarkusRestClientBuilder si lo requieres más customizado.

  Para quarkus la inyección de propiedades se da por @ConfigProperty(name="app.domains.microB.uri") y no por value como en spring boot.

  Para penache, siempre anotar el @Transactional
  
  <img width="1129" height="379" alt="image" src="https://github.com/user-attachments/assets/81c9b4a6-2873-4144-8645-13c1f4b4b984" />

**2. Fundamentos de elastic search**
  **2.1 ¿Cómo conectarlo a quarkus?**  
