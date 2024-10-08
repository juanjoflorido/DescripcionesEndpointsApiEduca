# Descripción 
ApiEduca es un API REST que expone las entidades que se gestionan en la Herramienta de Gestión Académica y Administrativa de los centros educativos (Pincel Ekade). No obstante, de forma temporal, también se han incluidos entidades externas como, por ejemplo, algunas de las gestionadas en el Directorio de Centros o el Plan de Estudios de Canarias.

# Seguridad

ApiEduca es un Api de carácter interno y su consumo solo puede realizarse a través de la Intranet corporativa . Además, es condición necesaria disponer de un ApiKey para su acceso. Este ApiKey tendrá asociado unos niveles de autorización que, en buena parte, marcarán los privilegios de acceso y manipulación de los recursos expuestos.

Existen determinados endpoints en los que disponer de un ApiKey con los privilegios necesarios no nos aporta el acceso a los mismo. Dichos endpoints requieren de una autenticación personal. ApiEduca provee de diferentes endpoints para realizar la autenticación cubriendo las diferentes casuísticas. Ver [Anexo I](#AnexoI)


# Generalidades.
Se ha intentado imprimir la máxima regularidad en la implementación de los endpoints que conforman este Api con el objetivo de normalizar su desarrollo y, a la par, simplificar su consumo.  En este apartado describiremos lo que, a nuestro juicio, en conveniente conocer por su carácter transversal.


## Estructura de las respuestas.

Las respuestas a las peticiones a cualquier petición de ApiEduca siempre tiene la misma estructrura JSON. Se trata de un objeto que actúa como envoltura (wrap) ante cualquier petición. Este objeto unificado de respuesta tiene 3 campos: datos, mensajes y metadatos.

### Datos.

Este campo devuelve los datos solicitados (generalmente, ante peticiones GET). Distinguiremos entre colecciones (devolución de un array) y entregas simples (devolución de un objeto). En el desarrollo de ApiEduca hemos creado un catálogo de DTO's (Data Tranfer Object) que representa la interfaz al exterior de las entidades que gestionamos y cuyo objetivo es dar legibilidad, desacoplar las respuestas a las estructuras internas y ocultar aspectos irrelevantes a los consumidores.
En el apartado "colecciones" veremos como, en ocasiones, disponemos de representaciones polomórficas de estas entidades a través de la definición de múltiples DTO's, permitiéndonos equilibrar la necesidad de información con el rendimiento de las respuestas y también con niveles de seguridad en base al contenido expuesto.


### Mensajes.
Este campo devuelve una colección de mensajes (potencialmente vacío). En muchas ocasiones es necesario acompañar las respuestas con mensajes que aporten información adicional a los consumidores. Estos mensajes van desde meros avisos hasta notificaciones de errores. Por ejemplo, el sistema, ante una petición incorrectamente formada, notificará de este hecho al peticionario. También es el mecanismo habitual cuando se realiza una petición que incumple reglas de negocio establecidas ( Ej. "La fecha de finalización de una matrícula no puede ser anterior a la fecha de matrícula"). Entre los datos que se aportan en este objeto 

### Metadatos.
Este campo es, precisamente, el que nos ha llevado a optar por el uso de envolvente a las respuestas. Se trata de un objeto que nos permite aportar información sobre la propia gestión de la petición. A nivel básico, aporta, entre otros datos, la fecha/hora de la petición, usuario autenticado, url de la solicitud, etc. En determinadas ocaciones estos medadatos se enriquecen con informacióin adicional. Tal es el caso de las respuestas de peticiones que devuelven colecciones. En estos casos de aporta información adicional como, por ejemplo, datos de paginación, filtro y orden.


## Códigos de respuesta.
En función de la naturaleza de las peticiones o del resultado obtenido al procesar una petición nos podemos encontrar con múltiples casuísticas. Hemos intentado ajustarnos a la semántica establecida en el protocolo HTTP en lo relativo a los códigos de respuestas. Con carácter general se devuelve un subconjunto de códigos que intentamos describir a continuación.

- 200 (OK): las respuestas acompañadas con esté código responden a dos casos bien diferenciados, aunque ambos casos tienen en común que reflejan peticiones que han podido ser procesadas de forma satisfactoria:
    - Se devuelve al usuario el contenido solicitado.
    - Lo solicitado por el usuario ha sido satisfactoriamente procesado pero la lógica de la aplicación establece que no debe aportarse la información requerida.
Es importante destacar que la intención primera de codificación del segundo caso era un código de respuesta 204 (No content). Sin embargo esto no es posible puesto que el protocolo no permite que se envíe contenido en el "body" de la respuesta y, en el caso de usar envolventes, siempre vamos a requierir devolver contenido.
- 201 (Created): es el código de respuesta ante una petición en la que se solicita la creación de un recurso y ha sido satisfactoria.
- 203 (Non-Authoritative Information): es el código de respuesta que damos ante una petición habilitada para un cliente que ha podido ser atendida pero la lógica de la aplicación establece que no debe aportarse la información requerida. Un ejemplo de respuesta con este código podría ser la solicitud de sesiones de evaluación no publicadas para ciertas aplicaciones.
- 400 (Bad Request): es el código de respuesta ante una petición que no puede ser atendida por estar mal formada, por no seguir las directrices de la especificación de los parámetros query, etc. En resumen, estamos ante petición que no pueden ser resueltas de forma satisfactoria.
- 401 (Unauthorized): como indica su descripción se da esta respuesta cuando el cliente no está autorizado a consumir el endpoint. Puede deberse a que el usuario no ha hecho uso de un token o que la información del token establece que el cliente no es apto para el consumo del endpoint.
- 403 (Forbidden): este código de respuesta solo se devuelve cuando se hace uso de endpoints dirigidos a la autenticación y, fruto del procesado de la petición se dictamina que la autenticación es insatisfactoria. Por ejemplo, no se obtiene un token, debido a credenciales no válidas.
- 409 (Conflict): se devuelve cuando se ejecuta un endpoint de manipulación de un recurso ( métodos POST, PUT, DELETE, PATCH ) y hay una o varias reglas de negocio que impiden la ejecución de la acción solicitada. Se devolverá en el nodo "mensajes" la colección de reglas que se incumplen.
- 500 (Internal Server Error): no se establece ningún desglose. Aparece este error ante excepciones internas de la aplicación.

## Colecciones

### Parámetros de carácter general
Los endpoints de tipo GET que devuelven colecciones de datos conforman el conjunto más representativo de endpoints de ApiEduca y, probablemente, el más demandado. Es muy importante, por tanto, aportarle la suficiente flexibilidad para que cubra a las necesidades de la mayoría de los consumidores, aunque evitando dar respuestas sobreajustadas que provoquen un exceso de configuraciones que compliquen su uso e implementación. Como respuesta a la flexibilidad requerida hemos modelado de forma tranasversal el tratamiento de tal flexibilidad en cuanto a la información devuelta como en la oferta de parámetros. La mayoría de los endpoints que devuelven colecciones disponen de, al menos, uno de los siguientes parámetros.

- <b>Nivel de detalle</b>: este parámetro nos permite especificar la cantidad de información que vamos a recibir en la colección de cada uno de los objetos. Los valores posibles son "r"(reducido), "m"(medio) y "e"(extendido), aunque puede que algunos endpoints no ofrezcan todos los niveles. Es obvio que el uso del nivel reducido redundará en una mejora del rendimientos. En general, se aconseja usar siempre el nivel mínimo que aporte la información requerida.
En este punto hay otro aspecto a destacar y está relacionado con la seguridad. Puede que a detemrinados usuarios solo se les permita hacer uso de niveles reducidos de contenido debido a que los niveles extendidos exponen información a la que dichos usuarios no están autorizados a acceder.

- <b>Opcion</b>:  este parámetro podrá tomar como valor un número natural: 1, 2, 3, ... Cada una de las opciones tiene establecido un conjunto de parámetros "query" validos. Es decir, puede que el endpoint disponga de un nutrido conjunto de parámetros query establecido pero, cada opción, establecerá que subconjunto de los mismos es válido ante dicha opción.

- <b>Paginación</b>: posicionInicial y tamanyoBloque: como es obvio, debemos establecer un sistema de paginación ante endpoints que devuelven colecciones de entidades. Hemos definido dos parámetros para establecer la página requerida en la respuesta. Lo hacemos a través de la posición inicial del paquete solicitado y el número de datos requeridos. Por ejemplo, si indicamos en el parámetro "posicionInicial" el valor 4 y al parámetros "tamanyoBloque" el valor 70, el sistema devolverá la cuarta página de un sistema de bloques de 70 registros.
En caso de omisión de estos parámetros se devolverá la primera página de una paginación de bloques de tamaño 100. ( posicionIniciao=0 y tamanyoBloque=100).


### Orden <pendiente>

## [Anexo I]{#AnexoI} 