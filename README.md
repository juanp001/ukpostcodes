# Ingesta de UK Postcodes

## Requerimientos

**Librerías**

Para poder ejecutar el código es necesario tener instaladas las librerías del archivo requirements.txt

**Base de datos**

Como base de datos se utilizó una base de datos mariaDB

La tabla que se creo se llama coordenates_postcodes

Los campos de la tabla son 

- latitude:  VARCHAR(80)
- longitude: VARCHAR(80)
- postcode:  VARCHAR(80)

Se decidió guardar las longitudes como texto debido a que esos dos campos conforman la llave de la tabla y para consultas de tipo "=" es comparar contra texto que contra número con decimales.

**Prueba**

En la carpeta test se encuentra una ejecución de los primeros 1.957.601 datos del csv postcodesgeo2.csv. El archivo inserted contiene las datos que se insertaron en la base de datos y el archivo log contiene todos los puntos procesados.

---

## Pipeline de limpieza: **cleanTypes**

En esta etapa se remueven todos los registros que contenga coordenadas nulas o que no sean numéricos. Por ejemplo de los datos de abajo se removerían los siguientes:

 - 1.2,5.6   Se mantiene
 - 1.2f,7.8  Se remueve
 - 0.9,      Se remueve

Este pipeline retorna dos Dataframes de pandas, uno que contiene los datos correctos y el otro los que se removieron

---

## Pipeline de creación de JSON: **createJsonGeo**

Este pipeline recibe un dataframe de pandas y retorna un json con el limite de códigos postales a recibir para una coordenada específica.

Ejemplo: 

Entrada:
|latitude|longitude
|---|---|
|0.34|-5.89|

Salida:

```
{
  "geolocations" : [{
    "longitude":  -5.89,
    "latitude": 0.34
    "limit":1
  }]
}
``` 

---
## Pipeline de consumo de API: **getPostCodes**

Este pipeline recibe los siguientes parámetros:

- url: Url de la api
- data: JSON que contiene la info a enviar a la API
- headers: Headers para la consulta HTTP
- rawData: Dataframe que contiene los datos antes de ser un JSON
- maxRetry: Número máximo de reintentos

Con base en los parámetros el pipeline ejecuta la consulta al API, si se obtiene un **error de conexión** se reintenta tantas veces como en maxRetry este especificado y si al final no se consigue una respuesta se genera una excepción.

Si la respuesta obtiene un codigo 200 se leen todos los datos del JSON de respuesta, pero si es un código diferente al 200 se marcan todos los puntos como fallidos.

Al final se retorna un Dataframe parecido al que se observa a continuación.

|latitude|longitude|status|message
|---|---|---|---|
|0.34|-5.89|ok|postcode 1
|0|0|error|postcode not found

---

## Clase de base de datos: **coordinatesPostcodes**

Esta clase tiene 3 métodos

**connectDb**

Sirve para conectarse a una base de datos


**insertPostcode**

Inserta los registros que devuelve la API de postcodes en la base de datos en la tabla coordenates_postcodes. Es importante destacar que si ya hay un valor para las coordenadas lo actualiza y no crea un registro nuevo.

Retorna una variable de exito para determinar si la ejecución se hizo. Tambien retorna un mensaje de error en caso de que la operación no haya tenido éxito.



**closeConnection**

Cierra la conexión de base de datos

---

## Funcion: **generateLogFile**

Recibe el directorio, el nombre del archivo y un dataframe que contiene los datos a registrar en log. 

Genera un csv en la ruta especificada

---

## Script principal

En este script se hace el llamado a todos los pipelines y clases creadas para la ejecución del ETL.

La constante mas importante del flujo es **CHUNKSIZE** puesto que con este se puede controlar cuantos registros del csv de coordenadas se van a procesar en cada ciclo. Es importante resaltar que este valor no puede ser mayor a 100 debido a que es el número máximo que la API soporta para una consulta.

El flujo del programa esta representado en el diagrama de flujo que se puede encontrar en la carpeta imágenes.

A continuación se explica cada una de las variables del script principal:
- URL: Url de la api 
- HEADERS: Headers http para consumir la API
- TARGET_COLUMNS: Columnas a limpiar
- COLUMNS_RENAME: Nuevos nombres para las columnas que se van a limpiar
- CHUNKSIZE: Cantidad de registros a procesar en cada interación
- NEWCOL: Nombre de la nueva columna en el JSON
- ROOT_JSON: Root del JSON 
- INPUT_DATA_FILE: Ruta donde esta el archivo de coordenadas 
- OUTPUT_LOG_DIR: Ruta donde se van a disponer los logs
- OUTPUT_LOG_FILENAME: Nombre del archivo de logs
- OUTPUT_INSERTED_RECORDS_DIR: Ruta donde se va a disponer el archivo csv de registros insertados
- OUTPUT_INSERTED_RECORDS_FILE: Nombre del archivo de registros insertados
- DB_USER: Usuario de la base de datos
- DB_PASSWORD: Contraseña de la base de datos
- DB_NAME: Nombre de la base de datos
- DB_URI: Host de la base de datos
- INSERTED_FLAG: Flag para determinar si el archivo csv escribe titulos o no
- LOG_FLAG: Flag para determinar si el archivo csv escribe titulos o no


