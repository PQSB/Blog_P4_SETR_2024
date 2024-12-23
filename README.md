# Blog_P4_SETR_2024

**Fecha:** 23/12/2024

**Autores:** Andrés Galea Torrecilla y Adrián Navarredonda Arizaleta

**Nombre equipo:** Francesco

**ID equipo:** 3

**Vídeo de ejecución:** https://urjc-my.sharepoint.com/:v:/g/personal/a_galea_2022_alumnos_urjc_es/EYRW2fpPwlNNlg6ACK3QV3gBEDX6kDvIr5wm-v6xOjJKyg?e=dRG7Cy

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

Para realizar estas tareas, decidimos que lo mejor sería introducir el planificador FreeRTOS para Arduino. Este planificador tiene dos tareas:
- Leer el ultrasonidos, y parar los motores en caso de obstáculo detectado.
- Leer el infrarrojo, y comandar a los motores en consecuencia.

Ambas tareas son periódicas. La primera tarea tiene un delay entre ejecuciones de 25 ms con una prioridad de 3; la segunda tarea tiene un delay entre ejecuciones de 0 ms y una prioridad de 4.

El funcionamiento de la primera tarea es el siguiente:
- Leer sensor de ultrasonidos.
- En caso de obstaculo detectado, detener motores, y llamar a función end_lap.
- Calcular el porcentaje de línea vista en la vuelta, con las variables rellenas en la tarea dos, explicada mas adelante.
- En la fucion end_lap, se enviarán los mensajes de obstáculo detectado, fin de vuelta, y estadística de línea vista, al ESP32.
    - Activar flag de vuelta terminada

El funcionamiento de la segunda tarea es el siguiente:
- Revisar que el flag de fin de vuelta no esté activa
- Leer el sensor de infrarrojos
- Sumar 1 a una variable, pues se ha leído el sensor de infrarrojos, y en caso de haber visto la línea, sumar 1 a otra variable. Esto servirá para el cálculo del porcentaje de línea vista en la vuelta, explicado en la tarea uno.
- Si en la iteración anterior la línea se había perdido y en esta se ha encontrado, mandar al ESP32 el mensaje Línea encontrada y el mensaje Fin de búsqueda de línea.
- Ejecutar comportamiento siguelineas, dividido en dos casos: haber visto la línea y no haberla visto. Para estos comportamientos se ha definido una constante SPEED. Estos comportamientos, a su vez, se dividen en más casos:
    - Ver la línea:
        1. Se ha visto línea en el centro del sensor: se comanda a ambos motores la velocidad SPEED.
        2. Se ha visto la línea en el sensor de la izquierda: se comanda al motor de la izquierda SPEED/2, y al motor de la derecha SPEED, para girar hacia la izquierda.
        3. Se ha visto la línea en el sensor de la derecha: se comanda al motor de la derecha SPEED/2, y al motor de la izquierda SPEED, para girar hacia la derecha.
    - No ver la línea, en este caso se manda también el mensaje Línea perdida y el mensaje Inicio de búsqueda de línea:
        1. Se vio la línea por última vez a la izquierda: se comanda al motor de la izquierda 0, y al motor de la derecha SPEED, para girar más bruscamente hacia la izquierda.
        2. Se vio la línea por última vez a la derecha: se comanda al motor de la derecha 0, y al motor de la izquierda SPEED, para girar más bruscamente hacia la derecha.

### ESP32
El ESP32 es el encargado de:
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

## Librería communication_stub
Tiene las funciones utilizadas para el procesamiento de los mensajes recibidos y la creación de los mensajes JSON a publicar por el ESP32 en cada caso. Las funciones que incluye son:
- **serial_send:** recibe un String con el mensaje a enviar y el puerto serie por el que se quiere enviar y envía el mensaje añadiendo el carácter **'{'** antes del mensaje y el carácter **'}'** al final de este.

- **proccess_serial_msg:** esta función recibe un entero por referencia y un puntero a entero y devuelve el valor del campo acción recibido en el entero pasado por referencia, y el valor recibido (en caso de que lo haya) actualizando el contenido de la memoria en la dirección a la que apunta el puntero.

- **valid_action:** devuelve *true* si la acción recibida es válida y *false* si no lo es.

- **create_json_msg:** devuelve un string con el mensaje JSON creado a partir de los parámetros recibidos.

## Dificultades
- Determinar un número correcto de tareas: incialmente teníamos separadas en dos tareas la lectura del sensor de infrarrojos y la actuación de los motores, lo que era un gasto innecesario de recursos ya que el periodo del infrarrojo era menor que el de los motores, pero además, esto causaba que la lectura y la actuación no fueran sincronizadas provocando un comportamiento incorrecto del siguelíneas.

- Creación de un protoclo de comuniación que no malgaste recursos: incialemente el protocolo que habíamos diseñado enviaba una cantidad de datos mucho mayor, lo que añadía latencia y gasto computacional al Arduino innecesariamente.

- Asignar tareas y prioridades apropiadas que no causen la no ejecución de algunas tareas.
