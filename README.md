# Laboratorio de diseño y simulacion con AWS

## Parte 1: Diseño de la infraestructura de nube

## Diseño de infraestructura basica de una web escalable



Los componentes que conformaran la aplicación seran los siguientes:

```
  -  EC2  -  Para la instalacion de nuestro(s) servidor de aplicaciones podemos utilizar instancias EC2 las cuales nos permitiran  redimensionar las caracterustucas de disco, memoria y cpu sobre demanda
  -  RDS  - Con el fin de guardar nuestra informacion podemos hacer uso del tecnologias como RDS el cual nos permitira instalar nuestra base de datos ya sea SQL o NoSQL segun se requiera 
  -  CloudFront y s3 - Para desplegar el frontend de nuestra aplicacion podemos utilizar cloudfront y adicionalmente un bucket de s3 para almacenar recursos estaticos como imagenes o archivos    
  -  ELB - Para el manejo del trafico distribuido a traves de las instancias EC2 en las zonas de disponibilidad utilizaremos balanceadores de carga elasticos o ELB
  -  Grupos de auto escalado -  en caso de ser necesario podriamos utilizar autoescalado con el fin de mantener nuestros disponibilidad acorde a los service level agreements (SLA), asi en dado caso que identifiquemos algun horario en el que el servicio este mas saturado, aumentar el numero de instancias y durante el horario nocturno por ejemplo disminuirlas 
  -  VPC y grupos se seguridad - para separar los recursos de la nube que vamos a desplegar podemos hacer uso de redes virtuales y asignar grupos de seguridad a nuestros recursos mediante VPC
  -  CloudWatch - Para el manejo de metricas y logs de nuestra aplicacion utilizaremos CloudWatch el cual nos servira como guia para evaluar cuando nuestra aplicacion requiere algun cambio
  -  Cert manager - Para la generación y renovación  de los certificados de seguridad hacia nuestra aplicacion web podemos utilizar el cert manager
  -  Route 53 - Utilizaremos Ruta 53 para el manejo de los dominios de nuestro sitio web
```

A continuacion se presenta el diagrama de la arquitectura  en aws:

<img width="575" alt="Captura de pantalla 2024-05-31 a la(s) 6 25 47 p m" src="https://github.com/jc-esquivelojeda/aws-design-lab/assets/123103329/c9daa87a-fde7-4bfd-a91d-646c6ea1e6e0">


## Parte 2: Configuracion de IAM

Se contaran con los roles para: 

```
   - EC2: se le permitirá la comunicacion con las instancias de bases de datos RDS, recibir trafico https, acceder al bucket de s3 y los servicios de aws
   - RDS : se requerira un rol  para el dba admin y para las operaciones sobre de la base de datos, tambien se puede crear un rol de lectura en caso de una herramienta de reporteo consulte la informacion
   - administrador: El rol admin nos permitira administrar los recursos de aws de la aplicacion para conceder los permisos minimos a los otros roles o acceso a temas como informacion de pagos o bien seguridad como el manejo de usuarios IAM y VPC's
   - desarrollador :  se puede generar un rol para los devs que permita realizar actividades como el despliegue o el acceso a los logs del cloudwatch
   - S3 : nos permitira realizar las operaciones crud sobre el bucket
   - CloudWatch: se requiere un rol para permitir el acceso a los logs y las metricas
```

Las politicas a aplicar serian las siguientes:

Politicas del Cloudfront :
```
    - Permitir recibir trafico desde internet via https
    - Permitir la comunicacion con el S3 para lectura de imagenes y archivos 
    - Restringir el acceso al RDS
```

Politicas del EC2 de aplicacion:
```
  - Permitir la comunicacion con el servidor web para enviar y recibir datos
  - Permitir la comunicacion con la base de datos para las consultas y transacciones 
  - Restringir las conexiones salientes a internet que
```

Politicas del  RDS:
```
  - Permitir las conexiones con el servidor de aplicacion dentro de la VPC
  - Denegar las conexiones entrantes desde internet o servicios no relacionados
```

Politicas del s3:
```
  - Permitir la comunicacion con el servidor de aplicacion o el cloudfront para recuperar archivos 
  - Denegar la conexiones fuera de la vpc
```

## Parte 3: Estrategia de administracion de recursos

Para el manejo de escalado y balanceo de carga se puede tomar en cuenta:
```
  - Se puede predecir si se requiere  autoescalado en base al monitoreo del uso de recursos del sistema, analizando el uso de cpu, memoria y disco duro,
  - se tiene que evaluar en base a los logs recibidos el numero de instancias que se requieren
  - se puede habilitar un sistema de autoescalado calendarizado para que en caso de que suba la carga en horas pico se activen instancias adicionales o en caso de  utilizar un cluster de kubernetes, agregar pods adicionales
  - durante el horario nocturno se podria reducir el numero de instancias disponibles en caso de que el uso se reduzca en esas horas
  - Para el balanceo de carga se puede aplicar un ELB el cual se encargara de en base al health check de las instancias dirigir el trafico a la instancia que corresponda
```

Para el manejo del presupuesto:
```
  - podemos definir un presupuesto esperado de consumo de la aplicacion basandonos en uso previo
  - en caso de tener un presupuesto ya establecido se pueden establecer notificaciones que nos avisen en caso de que se rebase cierto monto permitido y asi tomar medidas precautorias
  - Revisar los reportes de gastos que AWS nos proporciona para evaluar en que servicios estamos gastando mas y realizar ajustes
  - Dar seguimiento al gasto mediante la interfaz de aws
```  
  
## Parte 4: Implementacion teorica


Descripcion de la arquitectura web simplificada:

1. Amazon Route 53: El flujo de entrada desde el navegador inicia a partir de ruta 53 quien contiene los dns de la aplicacion web, este redirijira al usuario hacia el cloudfront
2. Cloudfront y S3: despues de que ruta 53 redirige la solicitud a la pagina raiz / que se encuentra en cloudfront, este inicia la carga del frontend de la aplicacion y comienza a realizar las peticiones necesarias desde nuestro frontend, tambien realizara la carga de contenido estatico como imagenes solicitandola hacia nuestro S3, adicionalmente cloudfront sera responsable del CDN para cacheo de contenido estatico
3. Balanceador de carga (ELB): Dspues de que Route 53 define hacia donde mandar el trafico , las peticiones entrantes seran atendidas por el balanceador,el cual las distribuira hacia la instancia  EC2  correspondiente
4. Instancias EC2 : La instancia EC2 ateendera la peticion y regresara la informacion necesaria, estas instalncias estaran en un grupo de autoescalado, el cual aumentara o disminuira segun se requiera
5. RDS: Las instancias EC2 se conectaran al RDS para guardar y recuperar la informacion
6. CloudWatch: Mientras todas las transacciones ocurren cloudwatch sera el responsable de monitorear los servicios y guardar los logs y metricas asi como definir las alarmas para notificar escenarios como el pasarse del presupuesto
7. IAM : EL iam nos permitira administrar los accesos y comunicaciones seguras hacia los servicios que conformaran nuestra aplicacion web, asegurando unas politicas optimas y acceso granular a los servicios
8. VPC: Todas las interacciones se realizaran dentro de la nube privada que nos provea aws y aisladas de los accesos nod eseados
9. Cert manager:  Adicionalmente podemos agregar  un cert manager que este al pendiente de que poseamos certificados de seguridad firmados y actualizados



## Parte 5: Discusion y evaluacion

Eleccion de servicios y como interactuan para proveer una infraestructura segura y resistente, politicas y estrategias de manejo de recursos
  - Los contenidos estaticos y el CDN serán provistos  por Cloudfront y S3, 
  - Las solicitudes de usuario para obtener contenido dinamico seran enrutados por el ELB hacia la instancia EC2 correspondiente
  - Las instancias EC2 transaccionaran con el RDS 
  - Todas las interacciones seran registradas y monitoreadas desde cloudwatch
  - El autoescalamiento en instancias de EC2 sera realizado por nuestra configuracion de autoscaling
  - El chequeo del presupuesto nos asegura que no excedamos el presupuesto
  - Las politicas proporcionadas basandose en el minimo privilegio nos aseguraran que se puede acceder a los recursos unicamente necesarios para proporcionar el servicio sin comprometer la seguridad
  - El monitoreo constante de nuestros servicios, y el establecimiento de alarmas para notificar cualquier rebase en el presupuesto nos permitiran tener control necesario para llevar unas finanzas sanas

