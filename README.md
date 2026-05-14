# Búsqueda semántica en una base de datos utilizando el objeto vector

## Requisitos

Para poder ejecutar este laboratorio es necesario contar con los siguientes prerrequisitos:

1. **Base de datos Oracle con capacidades de IA**
    
    Es necesario contar con una base de datos **Oracle Database 23ai o 26ai**, ya sea del tipo:
    
    - Transaction Processing
    - Data Warehouse
    
2. **Credenciales para acceso a servicios de Inteligencia Artificial**
    
    Es importante contar con un API Key creada en la cuenta de OCI que nos permitirán realizar solicitudes desde la base de datos a los servicios de inteligencia artificial.
    
    La configuración del API Key se ve dde la siguiente manera.

    ```docker
    [DEFAULT]
    user=ocid1.user.oc1..
    fingerprint=24:
    tenancy=ocid1.tenancy.
    region=us-chicago-1
    ```
    

## Paso 1: Carga de los datos

Para realizar la carga de los datos, podemos ir al menú principal de la base de datos Autonomous AI Databases y seleccionar **Data Load.**

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image.png)

En la consola seleccionamos Load Data

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%201.png)

Y en Select Files seleccionamos el csv de personas disponible ![aquí](./personas100.csv).

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%202.png)

Al cargar el archivo csv, se detectará automáticamente el tipo de datos de cada columna, sin embargo, es necesario confirmar la configuración en el botón **Review Settings**.

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%203.png)

Una vez hayamos revisado los datos en el panel de configuración, podemos presionar **Close**, muchas veces no es necesario editar los valores, sólo es necesario confirmarlos.

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%204.png)

Cuando el badge de Review Settings desaparece, podemos hacer clic en **Start**

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%205.png)

Si todo sucede exitosamente, podremos ver el tiempo de duración de la carga de los archivos, la cantidad de columnas creadas y el estado de la carga.

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%206.png)

Ahora que tenemos los datos cargados, podemos volver a la página principal de la base de datos y ejecutar el resto de este laboratorio en la consola SQL

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%207.png)

## Paso 2: Creación de una credencial

El siguiente comando usa el paquete [dbms_vector](https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/dbms_vector1.html) para crear una credencial con el procedimiento [create_credential](https://docs.oracle.com/en/database/oracle/oracle-database/26/arpls/dbms_vector1.html#GUID-4BBCBF46-3903-4EBB-8BE8-A7690151CF25).

Los campos necesarios en este bloque pueden ser encontrados en la configuración de la credencial, el campo private_key corresponde al contenido del archivo .pem descargado en la creación de la key, el cuadrado rojo en la siguiente imagen.

![key](./Búsqueda%20semántica%20en%20una%20base%20de%20datos%20utilizando/image11.png)

```sql
declare
  jo json_object_t;
begin
  jo := json_object_t();
  jo.put('user_ocid','ocid1.user.oc1..');
  jo.put('tenancy_ocid','ocid1.tenancy.oc1..');
  jo.put('compartment_ocid','ocid1.compartment.oc1..');
  jo.put('private_key','MII...6+7eS');
  jo.put('fingerprint','00:ab');
  dbms_vector.create_credential(
    credential_name   => 'OCI_CRED',
    params            => json(jo.to_string));
end;
```

Verifica que la credencial aparece en la lista de credenciales

```sql
select * from user_credentials;
```

Después de crear la credencial, es importante permitirle a la base de datos realizar llamados a endpoints externos utilizando el procedimiento APPEND_HOST_ACE. Este procedimiento agrega una entrada de control de acceso (ACE) a la lista de control de acceso (ACL) de un host de red.

```sql
BEGIN  
    DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host => '*',
    ace  => xs$ace_type(privilege_list => xs$name_list('http'),
                        principal_name => 'ADMIN',
                        principal_type => xs_acl.ptype_db)
);
END;
```

Verifica que las credenciales son correctas y que la conexión es posible, el siguiente bloque de código debería responder un objeto vector

```sql
select dbms_vector_chain.utl_to_embedding('hello', json('
   {
        "provider"       : "ocigenai",
        "credential_name": "OCI_CRED",
        "url"            : "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
        "model"          : "cohere.embed-multilingual-v3.0"
    }')) as embedding from dual;
```

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%208.png)

Este vector corresponde a la representación numérica de la palabra “hello”.

Si la conexión no fue exitosa, es posible eliminar la credencial con el procedimiento DROP_CREDENTIAL y volver al inicio del paso 2.

```sql
BEGIN
   DBMS_CLOUD.DROP_CREDENTIAL('OCI_CRED');
END;
```

## Paso 3: Vectorizar todos los items de la tabla

La tabla insertada contiene en la columna Persona; descripciones de personas en inglés.

| **85473** | A feminist historian or academic specializing in women's rights and social movements, particularly in the context of Northern Ireland. | ["Education","Academia","Specialized Expertise"] |
| --- | --- | --- |
| **1218927** | An academic librarian focused on supporting student research and writing, likely at a community college or university level. | ["Education","Academia","Specialized Expertise"] |
| **6511252** | A water sanitation engineer or environmental scientist interested in low-cost solutions to global waterborne disease problems. | ["Environmental","Scientific","Professional"] |
| **6745913** | An English language teacher or educator focused on the effective integration of technology in language learning and instructional planning. | ["Education","Academia","Specialized Expertise"] |
| **11752403** | A preschool or early childhood educator whose specialty lies in nutrition and interactive learning activities. | ["Education","Academia","Specialized Expertise"] |
| **19934058** | An elementary school teacher of English Language Arts, likely focused on handwriting instruction for 2nd to 5th-grade students. | ["Education","Academia","Specialized Expertise"] |
| **19576128** | A French teacher at the middle school level, likely integrating technology and interdisciplinary approaches into their language instruction, with a focus on developing students' interpretive, interpersonal, and presentational skills in French, while also promoting cultural awareness and global perspectives. | ["Education","Linguistics","French_language"] |
| **16368966** | A historian of East Asian politics and international relations, likely with a focus on modern Korean history and the Cold War era. | ["Education","Academia","Specialized Expertise"] |
| **2928409** | A climate scientist or a glaciologist studying the impacts of climate change through ancient ice cores, likely specializing in paleoclimatology. | ["Environmental","Scientific","Conservation"] |
| **8696000** | A mechanical engineer specializing in hydraulic systems and piping design, with an emphasis on corrosion prevention and materials science. | ["Academic","Scientific","Educational"] |

Vamos a agregar una columna a la tabla de personas, esta tabla será de tipo vector

```sql
ALTER TABLE PEOPLE
ADD (
  embedding VECTOR
);

-- Expected result: Table PEOPLE altered.
```

Para cada item en la tabla, vamos a insertar en la columna embedding un elemento de tipo vector obtenido a partir del procedimiento [utl_to_embedding](https://docs.oracle.com/en/database/oracle/oracle-database/26/vecse/utl_to_embedding-and-utl_to_embeddings-dbms_vector_chain.html), utilizado previamente.

```sql
UPDATE PEOPLE
SET embedding =
  dbms_vector_chain.utl_to_embedding(
    TO_CLOB(persona),
    JSON('{
      "provider"       : "ocigenai",
      "credential_name": "OCI_CRED",
      "url"            : "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
      "model"          : "cohere.embed-multilingual-v3.0"
    }')
  )
WHERE embedding IS NULL
  AND ROWNUM <= 1000;
  
-- 100 rows updated.
```

Otra función bastante útil para trabajar con búsqueda vectorial es [VECTOR_DISTANCE](https://www.notion.so/Tecnologia-De-La-Informacion-En-Salud-SPA-2ed0ff41122881eca2d5f55b557961e5?pvs=21)

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%209.png)

Esta función me permite identificar la distancia entre dos vectores, la función soporta las siguientes distancias.

- `COSINE` es la métrica predeterminada. Calcula la distancia de coseno entre dos vectores.
- `DOT` calcula el producto punto negado de dos vectores. La función `INNER_PRODUCT` calcula el producto punto, es decir, la negación de esta métrica.
- `EUCLIDEAN`, también conocida como distancia L2, calcula la distancia euclidiana entre dos vectores.
- `EUCLIDEAN_SQUARED`, también llamada `L2_SQUARED`, es la distancia euclidiana sin calcular la raíz cuadrada.
- `HAMMING` calcula la distancia de Hamming entre dos vectores contando el número de dimensiones que difieren entre los dos vectores.
- `MANHATTAN`, también conocida como distancia L1 o distancia Manhattan, calcula la distancia de Manhattan entre dos vectores.
- `JACCARD` calcula la distancia de Jaccard. Los dos vectores utilizados en la consulta deben ser vectores `BINARY`.

![image.png](B%C3%BAsqueda%20sem%C3%A1ntica%20en%20una%20base%20de%20datos%20utilizando/image%2010.png)

El siguiente. bloque de código calcula la distancia entre dos vectores. Pruebe el resultado del siguiente bloque para diferentes tipos de métricas.

```sql
SELECT VECTOR_DISTANCE(VECTOR('[0.0, 1.0]'), VECTOR('[0.0, 10.0]'), EUCLIDEAN)
```

El siguiente bloque de código me permite vectorizar una frase y realizar una búsqueda vectorial en la base de datos.

```sql
SELECT
  p.persona,
  VECTOR_DISTANCE(
    p.EMBEDDING,
    dbms_vector_chain.utl_to_embedding(
      TO_CLOB('una ingeniera de inteligencia artificial'),
      JSON('{
        "provider"       : "ocigenai",
        "credential_name": "OCI_CRED",
        "url"            : "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
        "model"          : "cohere.embed-multilingual-v3.0"
      }')
    ),
    COSINE
  ) AS cosine_distance
FROM personas100 p ORDER BY COSINE_DISTANCE
FETCH FIRST 10 ROWS ONLY;
```


Perfecto, hemos implementado una búsqueda vectorial 🥳🎉

Algunas preguntas útiles para explorar pueden ser

¿Los resultados son consistentes al ejecutar la búsqueda en diferentes lenguajes? ¿francés? ¿portugués?

¿Cuál métrica funciona mejor para búsquedas semánticas?

¿En qué casos de uso sería útil realizar búsqueda vectorial?



El resultado de esta ejecución muestra un campo execution time, en el ejemplo anterior, el tiempo de ejecución es de 36.767 segundos. El siguiente paso es acelerar este tiempo de respuesta creando un índice.

![image](./Búsqueda%20semántica%20en%20una%20base%20de%20datos%20utilizando/image%2012.png)


## Paso 4: Creación de índices

A continuación, vamos a crear un índice IVF, este índice va a dividir el espacio vectorial en clusters y ejecutará la búsqueda únnicamente en el cluster más cercano. Este índice vive en disco y nos permitirá trabajar con datasets enormes.

```sql
CREATE VECTOR INDEX idx_personas ON PERSONAS100(EMBEDDING) ORGANIZATION NEIGHBOR PARTITIONS DISTANCE COSINE WITH TARGET ACCURACY 95;

-- Vector INDEX created.
```

Cuando el índice esté creado, podemos agregar el bloque `FETCH APPROX FIRST 10 ROWS ONLY` para indicar que la búsqueda debe hacerse por índice y `WITH TARGET ACCURACY 95` para indicar la precisión de la búsqueda.

```sql
SELECT
  p.persona,
  VECTOR_DISTANCE(
    p.EMBEDDING,
    dbms_vector_chain.utl_to_embedding(
      TO_CLOB('una ingeniera de inteligencia artificial'),
      JSON('{
        "provider"       : "ocigenai",
        "credential_name": "OCI_CRED",
        "url"            : "https://inference.generativeai.us-chicago-1.oci.oraclecloud.com/20231130/actions/embedText",
        "model"          : "cohere.embed-multilingual-v3.0"
      }')
    ),
    COSINE
  ) AS cosine_distance
FROM personas100 p
ORDER BY cosine_distance
FETCH APPROX FIRST 10 ROWS ONLY
WITH TARGET ACCURACY 95;
```

La consulta anterior busca las 10 personas en la base de datos cuyo perfil es semánticamente 
más cercano a *"una ingeniera de inteligencia artificial"*.

- **`utl_to_embedding(...)`** — Convierte el texto de búsqueda en un vector
   numérico en tiempo real, usando Cohere vía OCI Generative AI.

- **`VECTOR_DISTANCE(..., COSINE)`** — Mide qué tan similar es ese vector
   con el embedding almacenado en cada fila de `personas100`. Cosine mide
   el ángulo entre vectores — a menor distancia, mayor similitud semántica.

- **`ORDER BY cosine_distance`** — Ordena de más cercano a más lejano.

- **`FETCH APPROX FIRST 10 ROWS ONLY`** — Trae los 10 más similares
   usando el índice IVF (el `APPROX` es lo que activa el índice).

- **`WITH TARGET ACCURACY 95`** — Le dice al índice que puede sacrificar
   un 5% de precisión a cambio de velocidad.


El tiempo de ejecución varía bastante

![image](./Búsqueda%20semántica%20en%20una%20base%20de%20datos%20utilizando/image%2013.png)

¿Qué sucede si lo ejecutamos con un TARGET ACCURACY de 80?


