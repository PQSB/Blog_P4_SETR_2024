# Blog_P4_SETR_2024

**Fecha:** 19/12/2024

**Autores:** Andrés Galea Torrecilla and Adrián Navarredonda

**Vídeo de ejecución:** 

En esta practica de la asignatura de Sistemas empotrados y de tiempo real, se pide implementar un siguelineas capaz de enviar mensajes a traves de MQTT a un servidor. Para ello se dispone de 2 placas, un Arduino Uno en el que se implementara el comportamiento del siguelineas, y un ESP32 encargado de las comuncionaciones MQTT con el servidor. 

El Sistema dede de ser capaz de:

    - Seguir la línea lo más rápido posible sin salirse.
    - Comunicación IoT a través de MQTT.
    - Comunicación serie entre ESP32 y Arduino UNO.
    - Si el robot pierde la línea realizar una búsqueda de la línea de nuevo.
    - Detección de obstáculos.

## Detalles de implementación

### Arduino

Para la implementacion del Arduino, decidimos que lo mejor seria introducir el planificador FreeRTOS para Arduino, este planificador tiene dos tareas. La primera tarea consiste en leer sensor de ultrasonidos, y en caso de detectar un obstaculo delante del coche, se marca el fin de vuelta, enviando al ESP32 tres mensajes, estos siendo el mensaje de obstaculo detectado, el mensaje de Vuelta terminada, y el mensaje de porcentaje de vuelta con linea vista.

En cuanto a la otra tarea, esta se encarga de leer el sensor de infrarrojos, asi como de comandar a los motores. En esta tarea, primero se leera el sensor de infrarrojos, despues de sumara 1 a el numero de total de lecturas del infrarrojo, y 1 a el numero total de veces que se ha visto la linea si esta se ha visto, esto para el porcentaje de vuelta con linea vista. Tras esto, se ejecutara el comportamiento siguelineas. En caso de detectar la linea, y haberla perdido en iteraciones anteriores, se enviara al ESP los mensajes de linea encontrada, y fin de busqueda de linea

Para el comportamiento siguelineas, se ha implementado un control basado en casos, distinguiendo 2 casos principales, en uno se esta viendo la linea, y en el otro no. En el caso de que se este viendo la linea, se distinguen 3 casos, el primero en que la linea se ha detectado en el sensor central del infrarrojo, por lo que se comandara los motores a la misma velocidad para ir recto. En el caso de que se vea la linea en el sensor izquierdo de infrarrojos, en cuyo caso los motores de la izquierda reduciran su velocidad a la mitad, para girar hacia la izquierda. Y el ultimo caso en el que la linea es visible a la derecha, en este caso se reducira a la mitad la velocidad de los motores de la derecha, haciendo que el coche gire hacia la derecha.
El otro caso principal en el que no se esta detectando la linea, implementa el comportamiento busca linea y se envia el mensaje de linea perdidda y inicio de busqueda de linea al ESP, y se divide en dos casos. En el primer caso, la ultima que se vio la linea, estaba a la izquierda, en este caso se pararan los motores de la izquierda por completo, para asi girar mas bruscamente que en el giro a la izquierda normal, para encontrar la linea antes. El otro caso es que la ultima vez que se vio la linea, estuviera a la derecha, en este caso se pararan los motores de la derecha, para asi girar mas bruscamente que en el giro a la derecha normal, para encontrar la linea antes.

### ESP32
El ESP es el encargado de:
- la publiación de mensajes por *MQTT*.
- la publación cada cuatro segundos del mensaje ping.
- calcular el tiempo de vuelta (restando el tiempo en el que se recibe el mensaje **END_LAP** menos el tiempo almacenado cuando ser recibió el mensaje **START_LAP**).

De esta forma se libera al arduino la carga añadida de gestionar el ping y los cálculos de tiempos, además de reducir el número de bytes a enviar por el puerto serie.

El funcionamiento del ESP32 es el siguiente:
- Al iniciarse:
  1. Conectarse a la wifi y al servidor *MQTT*.

  2. Comunicar al Arduino que ya está conectado para que este comience la vuelta.
- En el void loop:
  1. Comprobar si debe ejecutarse el thread para mandar el ping.

  2. En caso de que haya algo para leer, leer del puerto serie un **char**.

  3. Si el **char** leido se corresponde con el de fin de mensaje, habiéndose recibido antes el de inicio de mensaje, entonces se procesa el mensaje que se ha estado recibiendo y se comprueba si es válido.

  4. Construir un JSON con los valores obtenidos del procesamiento del mensaje.

  5. Publicar el JSON.

Para crear el mensaje JSON hemos utilizado la librería **ArduinoJson**.

## Comunicación serie
Para la comuniación serie entre el Arduino y el ESP32 hemos implementado el siguiente protocolo:

- En caso de que haya un valor a enviar además de la acción: **{action|value}**
- Si sólo es necesario enviar una acción: **{action}**

El char **{** indica que lo siguiente que se reciba es el mensaje hasta que se recibe le char **}**.
**action** es un entero que se corresponde con los posibles valores de acción.
**value** es el valor numérico que es necesario en algunos mensajes.

## Dificultades
