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

Ahmm... ok... me tomó aproximadamente la mitad de la mañana más... desde las 11:47 que empecé a investigar los conceptos básicos hasta una pausa
a las 3:20, y de ahí retomé 10:30 y terminé a lñas 12:02 mientras escribo esto.

Si es un tema también con una curva de aprendizaje alto, pero lo padre de la IA es que no requieres perder tiempo extra viendo vídeos
o teniendo la necesidad de pagar udemy.  Si fue un servicio tedioso de configurar, seré honsto y creo será una muy buena oportunidad para
desglosar los pasos necesarios y a mi estilo, de una manera sencilla para que ustedes puedan configurar un poquín más rápido esto.

Ok, empecemos. Elastic search es un motor de búsqueda de datos. Sirve paa ejecuta querys de un pool de datos, con distintos datos, se usa más
en análisis de logs. Aquí entra el concepto de la *pila ELK*.

Pero, *¿qué es la pila ELK?* se resume en un stack sobre el ecosistema elastic:
E de elastic: crea las consultas
L de logstash: motor de ingesta de datos, y transformación, insertar y transformar
K de Kibana, un GUI muy práctico para análisis de logs.

Es decir, si quieres logs, te toca configurar estos 3 servicios, lo bueno es que hay un docker file que se encarga de eso.

Las versiones estables de estas 3 herramientas son de la versión 8, yo usé la 9, así que mis notas pueden tener
cierta diferencia al configurar tu docker compose. Yo copio y pego el mio para que puedan basarse:

```Docker-compose.yml
# Launch Elasticsearch
version: '3.2'

services:
  elasticsearch:
    container_name: elasticsearch
    pull_policy: missing
    image: docker.io/elastic/elasticsearch:9.1.5
    volumes:
      - ./elasticsearch:/usr/share/elasticsearch/data:Z
  ##    - ./elasticsearchdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      ES_JAVA_OPTS: "-Xms512m -Xmx512m"
      cluster.routing.allocation.disk.threshold_enabled: false
      discovery.type: single-node
    networks:
      - elk

  kibana:
    container_name: kibana
    pull_policy: missing
    image: docker.io/elastic/kibana:9.1.5
    volumes:
      - ./kibana/config:/usr/share/kibana/config
    ports:
      - "5601:5601"
    networks:
      - elk
    depends_on:
      - elasticsearch
  logstash:
    container_name: logstash
    pull_policy: missing
    image: docker.io/elastic/logstash:9.1.5
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline:ro,Z
    ports:
      - "12201:12201/udp"
      - "5000:5000"
      - "9600:9600"
    networks:
      - elk
    depends_on:
      - elasticsearch

networks:
  elk:
    driver: bridge

```

El setup actual del proyecto luce así:

<img width="901" height="677" alt="image" src="https://github.com/user-attachments/assets/5b8190d1-623b-416c-9b20-93cacc6e4254" />

Docker compose a nivel raiz sobre tu conjunto de proyectos, los folders son tus volumenes, importante tenerlosen cuenta en proyectos de docker
y hoy fue la oportunidad para comprenderlos mejor!!

¿Por qué son importantes los volumenes? lo veremos a continuación.

**2.0.1 Configuración de roles y volumenes**

Para configurar esta pila es en elsiguiente orden: primero elstic, después en tu docker-compose añades kibana y a lo último logstash,
esto para evitar problemas con los volumenes en tus priemras veces cuando no te salga.

Primero requieres configurar los built in users, con los siguientes comandos:

 ````docker.sh
 docker exec -it elasticsearch  bin/elasticsearch-reset-password -u logstash_system
 
 docker exec -it elasticsearch  bin/elasticsearch-reset-password -u elastic

 Y configurar el enrollment token para asociarlo internamente a tu kibana

 docker exec -it elasticsearch bin/elasticsearch-create-enrollment-token --scope kibana
 ````

 Guarda tus contras en un sitio protegido. Continuamos.

Elastic funciona como el motor central, lo alimentas con logstash y con kibana muestras a nivel usuario.
Requieres configurar persistencia, cuando tu reinicias contenedores o ejecutr mucho el docker compose, tus datos tienden a borrarse.
Con volumenes, se guardan, solo coniste en la asociación de una carpeta local en tu sistema la cual vinculas con el directorio equivalente en el UNIX del
contenedor. 

Continuamos por Kibana, debes guardar los usuarios derivados de elastic para que puedas loguearte fácilmente. Hay
algo llamado built in users https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/built-in-users

Algunos sirven para loguarse al GUI de kibana, o interactuar con plugins, otros solo sirven para crear la comunicación
con las herramientas externas de elastic. Para que Kibana pueda conectarse INTERNAMENTE a elastic, requieres usuario y contraseña, también
puedes crear api keys con la api de elastic... pero no soy devops.

Entonces, estos volumenes ayudan a que PRIMERO, no se te corrompan los contenedores, segunda a que no se te borren a cada rato
tus credes. Para Kibana su archivo de configuración es kibana.yml vinculado al volumen *- ./kibana/config:/usr/share/kibana/config*
````
#
# ** THIS IS AN AUTO-GENERATED FILE **
#

# Default Kibana configuration for docker target
server.host: "0.0.0.0"
server.name: kibana
elasticsearch.hosts: [ "http://elasticsearch:9200" ]
monitoring.ui.container.elasticsearch.enabled: true

elasticsearch.username: "kibana_system"
elasticsearch.password: "tu contra we"

````
Kibana system es un built in que te sirve para comunicarlo con elastic pero no para loguearte (lo explico más adelante).
Por último, logstash aplica el mismo principio... aunque diferente. El volumen de logstash es *./logstash/pipeline:/usr/share/logstash/pipeline:ro,Z*
y requieres modificar el archivo logstash.conf
````
input {
  tcp {
    port => 5000
    codec => json
  }
}

output {
  stdout {
    codec => rubydebug
  }
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "logstash-%{+YYYY.MM.dd}"
    user  => "logstash_internal"
    password => "tu  otnra we"
  }
}
````
logstash_internal no es un built-in. Este requiere un usuario creado con un rol. Primero se crea el rol, después el usuario:

````curl.sh
curl -u elastic:LA_CONTRA_WE http://localhost:9200/_security/user/logstash_internal

curl -u elastic:LA_CONTRA_WE -X POST http://localhost:9200/_security/role/logstash_writer -H "Content-Type: application/json" -d "{\"cluster\":[\"monitor\",\"manage_index_templates\"],\"indices\":[{\"names\":[\"logstash-*\",\"logs-*\"],\"privileges\":[\"create_index\",\"write\",\"create\",\"index\"]}]}"

````

Ejecutas simplemente estos dos curls y con eso ya creas tu usuario y rol funcionl para logstash. Para consultarlo puedes usar 

curl -u elastic:TU_CONTRA_WE http://localhost:9200/_security/user/logstash_internal

<img width="1739" height="575" alt="image" src="https://github.com/user-attachments/assets/490b7352-a042-4021-8802-c468c41e416d" />

Esto es debido a que como mencioné previamente, elastic tiene su propia API, aquí puedes ver un ejemplo para crear una APIkey
que es una alternativa más moderna a usar usuario y contra, yo no quisé configurar eso, pero esto y mucho más lo encuentras acá

https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-security-create-api-key

Al final cuando tengas todo, tus contenedores ya estarán sanos cuando los ejecutes con docker compose up -d

<img width="1664" height="179" alt="image" src="https://github.com/user-attachments/assets/254246ab-3e39-4232-a00e-c28dbd947ac4" />

Para comprobar el estado correcto de kibana, visita http://localhost:9200/. Te pedirá contra, metes la del usuario elastic que es el
root. 

<img width="2508" height="682" alt="image" src="https://github.com/user-attachments/assets/b520828d-8d7f-40a9-8eaf-eb4683238939" />

Si te aparece así, todo correcto. Para comprobar Kibana, te vas a http://localhost:5601/login?next=%2F

<img width="1780" height="1115" alt="image" src="https://github.com/user-attachments/assets/595d68bc-a359-4b71-884f-86c4047c906d" />

Te logueas con el usuario elastic, que es el root, pero con la api de elastic puedes cambiar eso después.

<img width="2337" height="1376" alt="image" src="https://github.com/user-attachments/assets/1f1b52cd-31df-42f9-afb6-4897f755391c" />

Y listo ^^ kibana configurado.

Por último logstash, para esto, una manera padre de probar es desde un script powershell, ya que estos tienen
un cliente tcp integrado, haz un archivo ps1 de la siguiente manera:

````
$client = New-Object System.Net.Sockets.TcpClient("localhost",5000)
$stream = $client.GetStream()
$writer = New-Object System.IO.StreamWriter($stream)
$writer.AutoFlush = $true
$writer.WriteLine('{"message":"hola desde powershell"}')
$client.Close()
````

Lo ejecutas y no te saldrá nada, pero en los logs de logstash si:

<img width="1672" height="278" alt="image" src="https://github.com/user-attachments/assets/eff6ddcb-b5a4-4bf3-8ff2-ee23132f2fc6" />

Si ejecutas curl -u elastic:TU_CONTRA http://localhost:9200/_cat/indices?v

Te debe aparecer el index que configuraste en logstash.conf

<img width="1657" height="79" alt="image" src="https://github.com/user-attachments/assets/1ea7bfe7-8d41-46ad-9f8f-4a6c5fcf4360" />

Y si consultas el endpoint curl -u elastic:TU_CONTRA http://localhost:9200/logstash-*/_search?pretty donde logstash-* representa tu indice <img width="486" height="52" alt="image" src="https://github.com/user-attachments/assets/de81630e-c802-4f43-8727-1c65f25c7f9a" />



 <img width="1961" height="1085" alt="image" src="https://github.com/user-attachments/assets/98c3cbbf-bc11-44b0-b856-ab0fb2a4ef3a" />

 Te saldrán tus mensajes, signific que logstash ya está configurado y ya podemos pasar al apartado de desarrollo para configurar
 nuestros logs en quarkus con algún wrapper.
  
 **2.1 ¿Cómo conectarlo a quarkus?**  
 Bueno, afortunadamente esta parte es más fácil. Ya hay una librería a alto nivel en Quarkus que lo configuras
 con tus credenciales y un constructor cliente inicia un objeto que te lo permite mandar de esta manera:
 
 https://quarkus.io/guides/elasticsearch#configuring-elasticsearch

 ````elasticclient.java
package org.acme.service;

import java.io.IOException;
import java.util.Random;
import java.util.UUID;

import org.acme.models.Example;

import co.elastic.clients.elasticsearch.ElasticsearchClient;
import co.elastic.clients.elasticsearch.core.IndexRequest;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class KibanaService {

    @Inject
    ElasticsearchClient client; 

    public void insert() throws IOException{
        Example<String> ex = new Example<>();
        Random random = new Random();

        ex.setId(random.nextInt(10));
        ex.setMesage("SI FUNCIONA EL KIBANA");
        ex.setRandomObj(new String("Ayy bebe"));
        
        IndexRequest<Example> request = IndexRequest.of(  
            b -> b.index("logstash-2026-01")
                .id(UUID.randomUUID().toString())
                .document(ex)); 

        client.index(request);
    }
}

 ````


 <img width="2544" height="1013" alt="image" src="https://github.com/user-attachments/assets/5f267592-6beb-4430-ad1f-21f2bd0f5c30" />
 Al mandar el mensaje, ya te lo manda al indice que creas, y algo útil en esto es que la creación de los indices es dinámico
 desde el programa, es decir, no requieres programaticamente configurar los indices a cada rato en logstash.conf, desde el código 
 si no existe, lo crea y se te refleja en el GUI.

 Pero hay que configurarlo para que muestre un JSON, en vez de este formao raro, desde el menú de los indices inicias discover y wala, un borrador de logs.
 También en teoría, se pueden añadir columnas extras:
 
 <img width="2548" height="1156" alt="image" src="https://github.com/user-attachments/assets/b02de6d5-ec2e-48b9-adab-504f2548efde" />

 Algo asi... pero debe existir una forma más nativa
 
 <img width="1291" height="55" alt="image" src="https://github.com/user-attachments/assets/86ae7535-3d60-449a-8fc3-fa265574a6a1" />

 Moviendole al discovery, al parecer los indexa de acuerdo al pojo que mandamos, así que no hay tanto tema:

 <img width="2537" height="1039" alt="image" src="https://github.com/user-attachments/assets/4f4945fe-e8b5-4c30-ab53-99e3c95c6137" />

 Pues ya tenemos un conector. Ahora hay que extender nuestro plugin para que haga lo siguiente:
 Actué como interceptor, que para todos los jars implementados, traceé las peticiones, cuando las tenga úbicadas, que las mande al
 elastic directamente bajo un pojo. Ahora no sé hasta ahora que tan separado deba ir el diseño o responsabilidades, porque no solo es
 el log de kibana, si no también el wrapper generico.

 Mientras veía unas capturas que tenía por ahí, medité que, no requiero wrappers genericos más que en una capa de front, en las demás no.
 Por lo tanto, ahí entraría como un plugin extra, una librería que nomás, te haga el wrapper de un responde a nivel experience. En otras capas,
 ahora otro tema complejo eran los logs en código, estos también podían salir en elastic. Debo buscar una manera de entender el,
 como se hace conexión desde el request hasta el response, e intermediamente. Lo que entiendo es que en s4lj se conecta un
 adaptador con xml y así manda la petición en automático al enviar. Ahora, dicen que no es muy optimo y que puede petar.

 Entonces son tres vias de entrada: los requests/responses que recibe el micro, los requests/responses
 que hace el micro y los logs que el usuario define. Con esos traceas toda tu actividad, y descartas falta de datos.

 Va a ser otro pex absorver ese conocimiento, pero ya sería el último punto fuerte. Esperemos y no requiera de kafka.

 **2.1.2. Diseño del commons-logging**
 
 Por ahora tengo este diseño conceptual, cualquier micro que implemente esta libreria, será interceptada
 en todo el universo, por lo tanto los devs tienen la olbigación de implementar un toString sobre todos los pojos y clases personalizadas
 
 <img width="624" height="651" alt="image" src="https://github.com/user-attachments/assets/8af6d9b1-67aa-42aa-bcba-7caea2c9344e" />

 Espero no sea tedioso esto. Uno no tiende a diseñar estas cosas, ya hay gente que las ha creado, pero si hicieras una migración a micros, que
 trabajoso sería. Y eso que no hemos llegado a pipes y nexus jajaja.

 Retomando el punto anterior, no se requiere un jar con cliente para mandar pojos customizados al servidor por http,
 de hecho eso es ineficiente y mientras investigaba, me encontré con el concepto del logback.

 En **spring boot** se usa logback y su suite (que no sabia para que funcionaba en mi última experiencia laboral pero ahora ya lo sé),
 sirve para canalizar una conexión TCP por medio de un servidor TCP embebido en tu jar. Se ocupa un logback.xml.

 ¿Qué pasa? que quarkus funciona muy diferente a spring boot, en resumidas quarkus fuerza a que uses jboss.Logger, e incluso
 en su código fuente lo fuerza, no puedes sobreescribirlo.
 
 <img width="1247" height="436" alt="image" src="https://github.com/user-attachments/assets/2799066c-affb-4348-9385-7f900196cfa2" />

 ¿Qué puedes hacer? el approach nativo que es el uso de slf4j-jboss-logmanager https://es.quarkus.io/guides/logging#json-logging,
 lo configuras desde tu properties fácilmente, solo pones el endpoint del tcp de tu logstash

 ````application.properties
# Enable the socket log handler
quarkus.log.socket.enable=true
# Specify the Logstash endpoint (e.g., localhost:4560)
quarkus.log.socket.endpoint=localhost:5000
# Enable JSON formatting for the socket handler
quarkus.log.socket.json=true
# Optional: Format the JSON in ECS format
quarkus.log.socket.json.log-format=ECS
# Optional: Configure exception output type
quarkus.log.socket.json.exception-output-type=formatted

 ````

Y para mandar no necesitas un cliente elastic... solo trigereas un log.info:
````kibanaService.java
package org.acme.service;

import org.acme.models.Example;
import org.jboss.logging.Logger;

import jakarta.enterprise.context.ApplicationScoped;

@ApplicationScoped
public class KibanaService {
    private static final Logger log = Logger.getLogger(KibanaService.class);

    public void insert(){
        Example<Object> hello = new Example<>();
        hello.setId(1);
        hello.setRandomObj(new String("dd"));
        hello.setMesage("DASDADSSDAASD");
        log.info("SI FUNCIONA CON EL INDICE "+hello.toString());
    }
}
````

Y con eso, logro este hermoso formato que aseguro, si es productivo, hasta me trae recuerdos!!!

<img width="2559" height="1315" alt="image" src="https://github.com/user-attachments/assets/31424256-8f87-475e-911c-c6b40f178a52" />

Ahora queda setear campos customizados, se hace en corto, y para removerlos se hace desde logstash. Alrato lo configuramos.

Luego de un receso, continuamos. En la anterio captura ya hemos logrdo configurar el kibana con logs. Pero muchos de los campos que nos aparecen
si ejecutamos el logger del jboss no son coherentes, nos interesan solo ciertos campos, y también añadir nuestros propios campos dinámicos.

Para quitar esos campos que hacen ruido, debemos modificar nuestro *logstash.conf* añadiendo una línea que identificará
nuestro plugin jar y, borrara los campos que le indiquemos:

````logstash.json
filter {
  if [service.name] == "commons-logging-lib" {
    mutate {
       remove_field => [
        "process.pid",
        "process.parent.pid",
        "process.command_line",
        "thread.name",
        "host.hostname",
        "process.name",
        "mdc",
        "service.version",
        "service.environment",
        "ecs.version",
        "@version",
        "process.thread.name",
        "log.logger",
        "data_stream.type",
        "process.thread.id",
        "_ignored",
        "_index",
        "_score",
        "event.sequence",
        "service.name",
        "ndc"
      ]
    }
  }
}

````

Al final tomará efecto esto.

<img width="2539" height="1059" alt="image" src="https://github.com/user-attachments/assets/de4f7b65-3d55-498f-a056-9f4e4feba1a4" />


Ahora, hay un tópico interesante que no es tan sonado realmente, o al menos hasta lo que he visto en mi experiencia. La customización real
de los loggers. En jboss se maneja el concepto del MDC. Hay más información en este blog que lo que quarkus te ofrece:

https://www.damirscorner.com/blog/posts/20230106-UsingMdcInQuarkus.html

https://quarkus.io/guides/logging#logging-format

En resumidas, te deja añadir meta datos extras a tus logs, dinámicamente desde tu código. ideal para intereptors.

Aquí un ejemplo con las propiedades que usaremos:

````KibanaService.java

````

Cuando se reinicie kibana o un micro o el logstash, importante recalcar que estos logs de iniciando son correctos para imprimir

<img width="2094" height="1171" alt="image" src="https://github.com/user-attachments/assets/80d06498-f2bd-4204-883d-c00f563662e0" />

Cuando tengas tu log, te aparecerán las columnas pero con MDC, pero tu no quieres eso. Entonces tienes que hacer un flat sobre MDC para transformarlo en nuevas columnas

<img width="2533" height="895" alt="image" src="https://github.com/user-attachments/assets/83e238ab-bc48-4add-b094-90c59465fd17" />

Se logra modificando una vez más el logstash.conf
````logstash.conf
filter {
  if [service.name] == "commons-logging-lib" {
    if [mdc] {
      ruby {
        code => '
          event.get("mdc").each { |k, v|
            event.set(k, v)
          }
        '
      }
    }

    mutate {
       remove_field => [
        "process.pid",
        "process.parent.pid",
        "process.command_line",
        "thread.name",
        "host.hostname",
        "process.name",
        "service.version",
        "service.environment",
        "ecs.version",
        "@version",
        "process.thread.name",
        "log.logger",
        "data_stream.type",
        "process.thread.id",
        "_ignored",
        "_index",
        "_score",
        "event.sequence",
        "service.name",
        "ndc",
        "mdc"
      ]
    }
  }
}
````
Y tu flatten surgirá efecto, así se agregan nuevas columnas a tus logs de elastic:

<img width="853" height="219" alt="image" src="https://github.com/user-attachments/assets/e90d86b4-0ed8-4cb0-9db4-7f30e0152629" />

**2.1.2.1 Configuración de los interceptors**

Para lograr esta tarea, lo haremos por medio de unos controllers de prueba que configuraremos en la libreria.

Si está dificilón saber como descubrir eficazmente el request body.

Después de una minibatalla, descubrí cómo funciona el ciclo de vida de unas clases interceptoras, entonces,
tienes 3 tipos de interceptores en quarkus que listo de forma ordenada: 

ContainerRequestFilter: Antes de serializar el request body.

ReaderInterceptor: Body request generado en bytes, previo a insertarse en un pipeline
con jackson para convertirlo al pojo de tu request.

ResponseInterceptor: cuando ya se recibió la respuesta pero todavia no se serializa

WritterInterceptor: igual que el ReaderInterceptor, pero para responses.

Noo, y faltan los logs del quarkus client porque esos también hay que imprimir jajaja.

Pero bueno, ya logramos la parte más pesada. Para esto declara tus 3 clases:


ContainerRequestFilter
````java
@Provider
public class RequestInterceptorInit implements ContainerRequestFilter{

    private static final Logger log = Logger.getLogger(KibanaService.class);

    @ConfigProperty(name = "app.name")
    private String name;

    @Override
    public void filter(ContainerRequestContext requestContext) throws IOException {
        // TODO Auto-generated method stub
        requestContext.setProperty("microservice", this.name);
        requestContext.setProperty("request-uri", requestContext.getUriInfo().getPath());
        requestContext.setProperty("request-method", requestContext.getMethod());
        requestContext.setProperty("transaction-id", requestContext.getHeaderString("transaction-id"));
        requestContext.setProperty("response-code", "-");   
    }
}
````

ReaderInterceptor
````java
@Provider
@Priority(Priorities.ENTITY_CODER)
public class RequestBodyInterceptor implements ReaderInterceptor{

    private static final int MAX = 9000;

    private static final Logger log = Logger.getLogger(RequestBodyInterceptor.class);
    
    @Override
    public Object aroundReadFrom(ReaderInterceptorContext requestContext) throws IOException, WebApplicationException {
        // TODO Auto-generated method stub
        MediaType mt = requestContext.getMediaType();

        byte[] body = requestContext.getInputStream().readNBytes(MAX);
        requestContext.setInputStream(new ByteArrayInputStream(body));

        MDC.put("microservice", requestContext.getProperty("microservice"));
        MDC.put("request-uri", requestContext.getProperty("request-uri"));
        MDC.put("request-method", requestContext.getProperty("request-method"));
        MDC.put("transaction-id", requestContext.getProperty("transaction-id"));
        MDC.put("response-code", requestContext.getProperty("response-code"));
        
        log.info("Request: ".concat(new String(body, StandardCharsets.UTF_8)));
        return requestContext.proceed();
    }
    
}
````

Con esto, ya puedes interceptar el request body de los micros, lo cual también es importante a la hora
de que hagas tus querys con Kibana, ya que puedes ubicar campos dentro del mismo json.

<img width="2490" height="1141" alt="image" src="https://github.com/user-attachments/assets/f2f39629-9902-4697-93c4-4e65b79ce3a6" />

Me falta los responses entrantes, y refinar un poco el message porque si debe ubicar keywords en los jsons.

Al fin, terminando con la mitad de la coniguración, ya tenemos un log completo con request y response:

<img width="2544" height="695" alt="image" src="https://github.com/user-attachments/assets/5cba7a76-ab64-45ac-955e-596c84e65819" />

Para los responses si costó una buena chatgepeteada honestamente. Lo trataré de resumir así, para responses no puedes comunicar ContainerResponseFilter
con WritterInterceptor, ni por MDS ni por el mismo pipeline de ContainerRequestContext. Puedes hacer un workaround usando vertex, que es motor
reactivo de quarkus, MDC guarda por hilo, vertex por request, porque en el modelo reactivo los threads se rotan.

````RequestInterceptorInit.java
@Provider
@Priority(Priorities.USER)
public class RequestInterceptorInit implements ContainerRequestFilter{

    private static final Logger log = Logger.getLogger(KibanaService.class);

    @ConfigProperty(name = "app.name")
    private String name;

    @Override
    public void filter(ContainerRequestContext requestContext) throws IOException {
        // TODO Auto-generated method stub
        requestContext.setProperty("microservice", this.name);
        requestContext.setProperty("request-uri", requestContext.getUriInfo().getPath());
        requestContext.setProperty("request-method", requestContext.getMethod());
        requestContext.setProperty("transaction-id", requestContext.getHeaderString("transaction-id"));
        requestContext.setProperty("response-code", "-");   
    }
}

@Provider
@Priority(Priorities.ENTITY_CODER)
public class ResponseBodyIntereptor implements WriterInterceptor{

    private static final Logger log = Logger.getLogger(ResponseBodyIntereptor.class);

    @Override
    public void aroundWriteTo(WriterInterceptorContext requestContext) throws IOException, WebApplicationException {
        // TODO Auto-generated method stub
        MediaType mt = requestContext.getMediaType();
        io.vertx.core.Context vertxCtx =
            io.vertx.core.Vertx.currentContext();

        ByteArrayOutputStream buffer = new ByteArrayOutputStream();
        OutputStream original = requestContext.getOutputStream();

        requestContext.setOutputStream(buffer);
        requestContext.proceed();

        byte[] body = buffer.toByteArray();
        MDC.put("microservice", vertxCtx.getLocal("microservice"));
        MDC.put("request-uri", vertxCtx.getLocal("request-uri"));
        MDC.put("request-method", vertxCtx.getLocal("request-method"));
        MDC.put("transaction-id", vertxCtx.getLocal("transaction-id"));
        MDC.put("response-code", vertxCtx.getLocal("response-code"));
        log.info("Response: ".concat(new String(body, StandardCharsets.UTF_8)));
        MDC.clear();

        original.write(body);
    }
    
}
````

Y así, ya hemos terminado la mitad del interceptor, ahora solo falta configurar las solicitudes de quarkus client. Lo haré mañana.
