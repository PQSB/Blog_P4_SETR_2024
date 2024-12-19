# Blog_P4_SETR_2024

**Fecha:** 19/12/2024

**Autores:** Andrés Galea Torrecilla y Adrián Navarredonda

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

El Arduino es el encargado de:
    - Comandar a los motores.
    - Leer el sensor de infrarrojos.
    - Leer el sensor de ultrasonidos.

Para realizar estas tareas, decidimos que lo mejor seria introudcir el planificador FreeRTOS para Arduino, este planificador tiene dos tareas:
    - Leer el ultrasonidos, y para los motores en caso de obstaculo detectado.
    - Leer el infrarrojo, y comandar a los motores en consecuencia.

Ambas tareas son periodicas. La primera tarea tiene un delay entre ejecuciones de 20ms con una prioridad de 3, la segunda tarea tiene un delay entre ejecuciones de 0ms y una prioridad de 4.

El funcionamiento de la primera tarea es el siguiente:
    - Leer sensor de ultrasonidos.
    - En caso de obstaculo detectado, detener motores, y llamar a funcion end_lap.
    - Calcular el porcentaje de linea vista en la vuelta, con las variables rellenas en la tarea dos, explicada mas adelante.
    - En la fucion end_lap, se enviara los mensajes de obstaculo detectado, fin de vuelta, y estadistica de linea vista, al ESP32.
    - Activar flag de vuelta terminada

El funcionamiento de la segunda tarea es el siguiente:
    - Revisar que el flag de fin de vuelta no este activa
    - Leer el sensor de infrarrojos
    - Sumar 1 a una variable, pues se ha leido el sensor de infrarrojos, y en caso de haber visto la linea, sumar 1 a otra variable, esto servirar para el calculo del porcentaje de linea vista en la vuelta, explicado en la tarea uno.
    - Si en la iteracion anterior la linea se habia perdido y en esta se ha encontrado, mandar al ESP32 el mensaje Linea encontrada, y el mensaje Fin de busqueda de linea.
    - Ejecutar comportamiento siguelineas, divido en dos casos, haber visto la linea, y no haberla visto, para estos comportamientos se ha definido una constante SPEED, estos comportamientos a su vez se dividen en mas casos:
        - Ver la linea:
            1. Se ha visto linea en el centro del sensor, se comanda a ambos motores la velocidad SPEED.
            2. Se ha visto la linea en el sensor de la izquierda, se comanda al motor de la izquierda SPEED/2, y al motor de la derecha SPEED, para girar hacia la izquierda.
            3. Se ha visto la linea en el sensor de la derecha, se comanda al motor de la derecha SPEED/2, y al motor de la izquierda SPEED, para girar hacia la derecha.
        - No ver la linea, en este caso se manda tambien el mensaje linea perdida, y el mensaje Inicio de busqueda de linea:
            1. Se vio la linea por ultima vez a la izqueirda, se comanda al motor de la izquierda 0, y al motor de la derecha SPEED, para girar mas bruscamente hacia la izquierda.
            2. Se vio la linea por ultima vez a la derecha, se comanda al motor de la derecha 0, y al motor de la izquierda SPEED, para girar mas bruscamente hacia la derecha.

Para la implementacion del Arduino, decidimos que lo mejor seria introducir el planificador FreeRTOS para Arduino, este planificador tiene dos tareas. La primera tarea consiste en leer sensor de ultrasonidos, y en caso de detectar un obstaculo delante del coche, se marca el fin de vuelta, enviando al ESP32 tres mensajes, estos siendo el mensaje de obstaculo detectado, el mensaje de Vuelta terminada, y el mensaje de porcentaje de vuelta con linea vista.

En cuanto a la otra tarea, esta se encarga de leer el sensor de infrarrojos, asi como de comandar a los motores. En esta tarea, primero se leera el sensor de infrarrojos, despues de sumara 1 a el numero de total de lecturas del infrarrojo, y 1 a el numero total de veces que se ha visto la linea si esta se ha visto, esto para el porcentaje de vuelta con linea vista. Tras esto, se ejecutara el comportamiento siguelineas. En caso de detectar la linea, y haberla perdido en iteraciones anteriores, se enviara al ESP los mensajes de linea encontrada, y fin de busqueda de linea

Para el comportamiento siguelineas, se ha implementado un control basado en casos, distinguiendo 2 casos principales, en uno se esta viendo la linea, y en el otro no. En el caso de que se este viendo la linea, se distinguen 3 casos, el primero en que la linea se ha detectado en el sensor central del infrarrojo, por lo que se comandara los motores a la misma velocidad para ir recto. En el caso de que se vea la linea en el sensor izquierdo de infrarrojos, en cuyo caso los motores de la izquierda reduciran su velocidad a la mitad, para girar hacia la izquierda. Y el ultimo caso en el que la linea es visible a la derecha, en este caso se reducira a la mitad la velocidad de los motores de la derecha, haciendo que el coche gire hacia la derecha.
El otro caso principal en el que no se esta detectando la linea, implementa el comportamiento busca linea y se envia el mensaje de linea perdidda y inicio de busqueda de linea al ESP, y se divide en dos casos. En el primer caso, la ultima que se vio la linea, estaba a la izquierda, en este caso se pararan los motores de la izquierda por completo, para asi girar mas bruscamente que en el giro a la izquierda normal, para encontrar la linea antes. El otro caso es que la ultima vez que se vio la linea, estuviera a la derecha, en este caso se pararan los motores de la derecha, para asi girar mas bruscamente que en el giro a la derecha normal, para encontrar la linea antes.

### ESP32
El ESP es el encargado de:
- la publiación de mensajes por *MQTT*.
- la publicación cada cuatro segundos del mensaje ping.
- calcular el tiempo de vuelta (restando el tiempo en el que se recibe el mensaje **END_LAP** menos el tiempo almacenado cuando se recibió el mensaje **START_LAP**).

De esta forma se libera al Arduino de la carga añadida de gestionar el ping y los cálculos de tiempos, además de reducir el número de bits a enviar por el puerto serie.

El funcionamiento del ESP32 es el siguiente:
- Al iniciarse:
  1. Conectarse a la wifi y al servidor *MQTT*.

  2. Comunicar al Arduino que ya está conectado para que este comience la vuelta.
- En el void loop:
  1. Comprobar si debe ejecutarse el thread para mandar el ping.

  2. En caso de que haya algo para leer, leer del puerto serie (el conectado al Arduino) un **char**.

  3. Si el **char** leido se corresponde con el de fin de mensaje, habiéndose recibido antes el de inicio de mensaje, entonces se procesa el mensaje que se ha estado recibiendo y se comprueba si es válido.

  4. Construir un JSON con los valores obtenidos del procesamiento del mensaje.

  5. Publicar el JSON.

Para crear el mensaje JSON hemos utilizado la librería **ArduinoJson**.

## Comunicación serie
Para la comuniación serie entre el Arduino y el ESP32 hemos implementado el siguiente protocolo:

- En caso de que haya un valor a enviar además de la acción: **{action|value}**
- Si sólo es necesario enviar una acción: **{action}**

El char **"{"** indica que lo siguiente que se reciba es el mensaje hasta que se recibe el char **"}"**.
**action** es un entero que se corresponde con los posibles valores de acción.
**value** es el valor numérico que es necesario en algunos mensajes.

Las funciones para comprobar si el mensaje es válido, procesar el mensaje y crear el JSON están definidas en una librería que hemos llamado **communication_stub**. Esta librería también incluye los enteros correspondientes a cada tipo de acción.

## Dificultades
- Determinar un número correcto de tareas: incialmente, teníamos separadas en dos tareas la lectura del sensor de infrarojos y la actuación de los motores, lo que era un gasto inccesario de recursos ya que el periodo del infrarrojo era menor que el de los motores, pero además, esto causaba que la lectura y la actuación no fueran sincronizadas provocando un comportamiento incorrecto del siguelíneas.

- Creación de un protoclo de comuniación que no malgaste recursos: incialemente, el protocolo que habíamos diseñado enviaba una cantidad de datos mucho mayor, lo que provocaba que añadía latencia y gasto computacional al Arduino innecesariamente.

- Asignar tareas y prioridades apropiadas que no causen la no ejecución de algunas tareas.
