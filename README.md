# Virtual Threads en Java

## 1. Explicación General de los Virtual Threads y Conceptos Fundamentales

Los Virtual Threads, también conocidos como "hilos ligeros" o "fibras", son una nueva característica introducida en Java para abordar los desafíos de escalabilidad y rendimiento en aplicaciones concurrentes. A diferencia de los hilos tradicionales del sistema operativo (OS threads), que son mapeados uno a uno a hilos del sistema operativo y conllevan una sobrecarga significativa en términos de memoria y tiempo de conmutación de contexto, los Virtual Threads son implementados en el espacio de usuario de la Java Virtual Machine (JVM). Esto permite que se puedan crear y gestionar un número mucho mayor de hilos con una huella de memoria reducida.

La propuesta de los Virtual Threads busca simplificar la programación concurrente, permitiendo a los desarrolladores escribir código concurrente de una manera más similar a cómo escribirían código secuencial bloqueante, pero logrando la eficiencia de las operaciones no bloqueantes.

Esto permite:

- Crear y gestionar un número **mucho mayor de hilos**.
- Utilizar **menos memoria** por hilo.
- Simplificar la programación concurrente.
- Escribir código concurrente con estilo secuencial bloqueante, pero con la eficiencia de operaciones no bloqueantes.

## 2. Carrier Threads vs. Virtual Threads

### Carrier Threads (Hilos Portadores)
La distinción entre "Carrier Threads" (hilos portadores) y "Virtual Threads" es fundamental para entender el funcionamiento de esta nueva arquitectura.

- Carrier Threads (Hilos Portadores): Son los hilos del sistema operativo subyacentes que realmente ejecutan el código de los Virtual Threads. Son hilos tradicionales de la JVM que actúan como "portadores" o "ejecutores" para los Virtual Threads. Cuando un Virtual Thread realiza una operación bloqueante (por ejemplo, una operación de E/S), el Virtual Thread se desvincula de su Carrier Thread, liberándolo para que pueda ejecutar otro Virtual Thread. Una vez que la operación bloqueante se completa, el Virtual Thread se reanuda, posiblemente en un Carrier Thread diferente.
- Virtual Threads (Hilos Virtuales): Son instancias de java.lang.Thread que son ligeras y no se mapean directamente a hilos del sistema operativo. Son gestionados por la JVM y pueden ser muy numerosos, permitiendo la creación de miles o incluso millones de Virtual Threads. La JVM programa estos Virtual Threads para que se ejecuten en un grupo más pequeño de Carrier Threads.
  

### Comparación entre Hilos Tradicionales y Virtuales

| Característica        | Hilos Tradicionales (Carrier Threads) | Hilos Virtuales             |
|------------------------|----------------------------------------|-----------------------------|
| **Creados por**        | Sistema operativo                      | JVM                         |
| **Uso de memoria**     | Alto (~1MB por hilo)                   | Bajo (~pocos KB)            |
| **Bloqueo**            | Bloquea el hilo del SO                 | Suspendido sin bloquear     |
| **Creación**           | Costosa y lenta                        | Muy rápida                  |
| **Escalabilidad**      | Limitada                               | Altísima                    |

---

## 3. Bloqueo y Escalabilidad

El bloqueo es un problema inherente en la programación concurrente tradicional. Cuando un hilo tradicional realiza una operación bloqueante (como esperar por una respuesta de red o una lectura de disco), ese hilo permanece ocupado y no puede realizar ningún otro trabajo hasta que la operación se complete. En sistemas con muchos hilos bloqueados, esto puede llevar a un consumo excesivo de recursos (memoria y CPU) y limitar significativamente la escalabilidad de la aplicación.

Los Virtual Threads abordan este problema de manera elegante. Cuando un Virtual Thread se bloquea en una operación de E/S, la JVM detecta este bloqueo y "desmonta" el Virtual Thread de su Carrier Thread. El Carrier Thread se libera y puede ser utilizado para ejecutar otro Virtual Thread. Una vez que la operación de E/S se completa, la JVM "monta" el Virtual Thread de nuevo en un Carrier Thread disponible, y el Virtual Thread continúa su ejecución desde donde se quedó. Esta capacidad de descargar y recargar Virtual Threads de los Carrier Threads permite que un número limitado de Carrier Threads maneje un número mucho mayor de Virtual Threads, lo que mejora drásticamente la escalabilidad de las aplicaciones intensivas en E/S. La imagen del documento ilustra cómo los "Traditional Threads" requieren 10,000 unidades de OS (sistema operativo), mientras que los "Virtual Threads" solo consumen "Low MB" (baja cantidad de megabytes), lo que indica su menor huella de memoria y mayor eficiencia.


## 4. Cambios o Extensiones para Crear y Gestionar Virtual Threads

La belleza de los Virtual Threads radica en su diseño para ser casi indistinguibles de los hilos tradicionales desde la perspectiva del programador. La API para crear y gestionar Virtual Threads es muy similar a la de los hilos tradicionales, lo que facilita la migración de código existente y la adopción por parte de los desarrolladores.
Para crear un Virtual Thread, se utiliza un Thread.Builder para especificar el tipo de hilo:

```java
Thread virtualThread = Thread.ofVirtual().start(runnable);
```

También se pueden crear Virtual Threads a través de un ExecutorService:

```java
try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(() -> {
        // Código a ejecutar en el Virtual Thread
    });
}
```

La JVM se encarga automáticamente de la programación, la conmutación de contexto y la multiplexación de los Virtual Threads sobre los Carrier Threads. Esto significa que los desarrolladores pueden centrarse en la lógica de negocio sin preocuparse por los detalles de bajo nivel de la gestión de hilos.



## 5. Breve Mención al Concepto Structured Concurrency
Structured Concurrency es un concepto emergente que busca mejorar la legibilidad, mantenibilidad y depuración del código concurrente al imponer una estructura jerárquica a las operaciones concurrentes. En esencia, todas las operaciones concurrentes lanzadas dentro de un ámbito (scope) deben completarse antes de que el ámbito termine. Esto evita fugas de hilos y facilita la gestión de errores y la cancelación.
Los Virtual Threads se integran muy bien con el concepto de Structured Concurrency, ya que su naturaleza ligera permite la creación de muchos hilos de vida corta para tareas específicas. La combinación de Virtual Threads y Structured Concurrency ofrece un modelo de programación concurrente más robusto y comprensible.

## 6. Análisis de Ventajas y Desventajas de la Propuesta
Ventajas:
- 	Escalabilidad Mejorada: Permiten la creación de millones de hilos concurrentes con una sobrecarga mínima, lo que es ideal para aplicaciones con un gran número de conexiones simultáneas, como servidores web o microservicios. La imagen lo demuestra al mostrar que 10,000 "Traditional Threads" utilizan mucho espacio en el sistema operativo, mientras que los "Virtual Threads" utilizan "Low MB".
- Programación Simplificada: Al permitir el uso de código bloqueante sin sacrificar la eficiencia, los Virtual Threads simplifican la escritura de aplicaciones concurrentes, eliminando la necesidad de APIs reactivas complejas para muchos casos de uso.
- Menor Consumo de Memoria: Debido a su naturaleza ligera, los Virtual Threads consumen mucha menos memoria por instancia que los hilos tradicionales del sistema operativo.
- Mayor Rendimiento: Reducen la sobrecarga de conmutación de contexto del sistema operativo y mejoran la utilización de la CPU al multiplexar eficientemente los Virtual Threads sobre un conjunto más pequeño de Carrier Threads.
- Depuración y Monitoreo Más Sencillos: Dado que se presentan como instancias de java.lang.Thread, las herramientas de depuración y monitoreo existentes pueden ser utilizadas con poca o ninguna modificación.
Desventajas:
- No aptos para tareas intensivas en CPU: Aunque son excelentes para operaciones intensivas en E/S, los Virtual Threads no ofrecen mejoras significativas para tareas que son principalmente intensivas en CPU, ya que un Virtual Thread intensivo en CPU bloqueará su Carrier Thread hasta que se complete la operación.
- Curva de aprendizaje inicial: Aunque la API es similar, comprender las implicaciones de rendimiento y las diferencias entre hilos virtuales y tradicionales requiere una comprensión inicial.
- Posibles problemas de "pinning": Si un Virtual Thread realiza una operación bloqueante que no es manejada por la JVM (por ejemplo, llamadas a código nativo que no son reconocidas), podría "atar" un Carrier Thread, reduciendo la eficiencia.
- Compatibilidad con código nativo: Las operaciones que implican el uso de JNI (Java Native Interface) pueden no liberar el Carrier Thread, lo que podría limitar los beneficios de los Virtual Threads en ciertos escenarios.


## 7. Escenarios Ideales para el Uso de Esta Propuesta

Los Virtual Threads son particularmente adecuados para los siguientes escenarios:

- Aplicaciones de Servidor Web y Microservicios: Donde un gran número de solicitudes simultáneas, a menudo bloqueadas por operaciones de E/S (acceso a bases de datos, llamadas a servicios externos), necesitan ser manejadas de manera eficiente.
- Procesamiento de Eventos y Mensajería: Aplicaciones que procesan flujos de eventos o mensajes y que a menudo esperan por respuestas o la disponibilidad de recursos externos.
- Servicios de Red de Baja Latencia: Donde es crucial mantener un alto rendimiento bajo una carga pesada de conexiones.
- Aplicaciones de E/S Intensivas: Cualquier aplicación que pase una cantidad significativa de tiempo esperando operaciones de entrada/salida.
- Migración de Código Sincrónico Bloqueante: Permiten que el código existente que utiliza operaciones bloqueantes se beneficie de una mayor escalabilidad sin una reescritura compleja a un estilo de programación asíncrono.


## 8. Otros Lenguajes de Programación que Implementen Ideas Similares

La idea de hilos ligeros o "fibras" no es nueva y ha sido implementada en otros lenguajes de programación con diferentes enfoques:

- Go (Goroutines): Go es quizás el ejemplo más prominente de un lenguaje que utiliza un modelo de concurrencia basado en hilos ligeros (goroutines) multiplexados sobre hilos del sistema operativo. Al igual que los Virtual Threads, las goroutines son muy baratas de crear y gestionar, y el runtime de Go se encarga de su programación.
- Kotlin (Coroutines): Kotlin ofrece coroutines, que son un concepto similar a los Virtual Threads, permitiendo la programación asíncrona de manera secuencial y bloqueante. Las coroutines de Kotlin se ejecutan en un grupo de hilos subyacentes.
- C# (async/await): Si bien no son hilos ligeros en el mismo sentido que los Virtual Threads o Goroutines, las palabras clave async y await en C# proporcionan un mecanismo para escribir código asíncrono que parece sincrónico, gestionando el desvío de las operaciones bloqueantes sin atar hilos del sistema operativo.
- Erlang (Procesos Ligeros): Erlang es conocido por su modelo de concurrencia basado en procesos muy ligeros y aislados que se comunican a través de mensajes. Estos procesos son gestionados por la máquina virtual de Erlang y son extremadamente eficientes para aplicaciones distribuidas y tolerantes a fallos.
  
## Datos curiosos
- Los virtual threads pueden iniciar en menos de 1ms.
- En pruebas internas, una JVM pudo ejecutar 10 millones de hilos virtuales sin error.
- La JVM maneja internamente las stacks de los virtual threads como objetos livianos y extensibles.
- El proyecto Loom comenzó en 2017 y se lanzó oficialmente en Java 21 (septiembre 2023).


## 9. Opinión Personal sobre el Futuro de la Propuesta

En lo personal, creo que los hilos Virtual Threads son una gran mejora para Java. Lo mejor es que nos permiten escribir código como siempre, de forma sencilla, pero ahora podemos hacer muchas tareas al mismo tiempo sin que el rendimiento se vea afectado.

Creo que esto va a facilitar mucho el trabajo, sobre todo para los que recién empezamos a aprender sobre programación con hilos. Ya no hará falta usar herramientas complicadas para manejar muchas tareas a la vez.

Seguramente en el futuro los Virtual Threads se usen cada vez más en aplicaciones grandes y en frameworks populares. Aunque no van a reemplazar todo, sí van a ser la opción más común para la mayoría de los casos.


## Implementación de código
### Clase ContadorURLsVirtualThreads

```java
package org.example;

import java.util.List;
import java.util.Map;
import java.util.concurrent.*;

public class ContadorURLsVirtualThreads {

    public static void main(String[] args) {
        List<String> urls = AnalizadorURL.leerURLsDesdeArchivo("urls_parcial1-1.txt");
        Map<String, Integer> resultados = new ConcurrentHashMap<>();

        ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

        for (String url : urls) {
            executor.submit(new AnalizadorURL(url, resultados));
        }

        executor.shutdown();
        try {
            executor.awaitTermination(5, TimeUnit.MINUTES);
        } catch (InterruptedException e) {
            System.err.println("Tiempo de espera excedido para la ejecución de los hilos.");
        }

        AnalizadorURL.guardarResultados(resultados, "resultado_urls.csv");
    }
}

```
### Clase AnalizadorURL

```java

package org.example;

import java.io.*;
import java.net.*;
import java.nio.file.*;
import java.util.*;
import java.util.regex.*;

public class AnalizadorURL implements Runnable {
    private final String url;
    private final Map<String, Integer> resultados;

    public AnalizadorURL(String url, Map<String, Integer> resultados) {
        this.url = url;
        this.resultados = resultados;
    }

    @Override
    public void run() {
        try {
            String contenido = descargarContenido(url);
            int cantidad = contarURLsInternas(contenido, url);
            resultados.put(url, cantidad);
        } catch (IOException e) {
            System.err.println("Error procesando URL: " + url);
            resultados.put(url, 0);
        }
    }

    private String descargarContenido(String urlString) throws IOException {
        StringBuilder contenido = new StringBuilder();
        URL urlObj = new URL(urlString);
        HttpURLConnection conn = (HttpURLConnection) urlObj.openConnection();
        conn.setRequestProperty("User-Agent", "Java URL Analyzer");
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(conn.getInputStream()))) {
            String linea;
            while ((linea = reader.readLine()) != null) {
                contenido.append(linea);
            }
        }
        return contenido.toString();
    }

    private int contarURLsInternas(String html, String baseURL) {
        Pattern pattern = Pattern.compile("href=\\\"(.*?)\\\"");
        Matcher matcher = pattern.matcher(html);
        URI baseURI = URI.create(baseURL);
        int contador = 0;
        while (matcher.find()) {
            try {
                String link = matcher.group(1);
                URI uri = baseURI.resolve(link);
                if (uri.getHost() != null && uri.getHost().equalsIgnoreCase(baseURI.getHost())) {
                    contador++;
                }
            } catch (IllegalArgumentException e) {
                // Ignorar links malformados
            }
        }
        return contador;
    }

    // --- Métodos estáticos utilitarios ---
    public static List<String> leerURLsDesdeArchivo(String nombreArchivo) {
        try {
            return Files.readAllLines(Path.of(nombreArchivo));
        } catch (IOException e) {
            System.err.println("Error al leer el archivo: " + nombreArchivo);
            return List.of();
        }
    }

    public static void guardarResultados(Map<String, Integer> resultados, String nombreArchivo) {
        try (BufferedWriter writer = Files.newBufferedWriter(Path.of(nombreArchivo))) {
            writer.write("URL,CantidadURLsInternas\n");
            for (Map.Entry<String, Integer> entry : resultados.entrySet()) {
                writer.write(entry.getKey() + "," + entry.getValue() + "\n");
            }
        } catch (IOException e) {
            System.err.println("Error al guardar el archivo de resultados.");
        }
    }
}

```
### POM.XML

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.example</groupId>
    <artifactId>contador-urls</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>24</maven.compiler.source>
        <maven.compiler.target>24</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <build>
        <plugins>
            <!-- Plugin para ejecutar tu clase main -->
            <plugin>
                <groupId>org.codehaus.mojo</groupId>
                <artifactId>exec-maven-plugin</artifactId>
                <version>3.1.0</version>
                <configuration>
                    <mainClass>ContadorURLsVirtualThreads</mainClass>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

```
