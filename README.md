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
