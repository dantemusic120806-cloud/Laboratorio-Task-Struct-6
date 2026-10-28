# Laboratorio-Task-Struct-6
# Informe de Laboratorio: Hilos en Linux (pthreads y el Modelo 1:1)
## De la Teoría Arquitectónica al Paralelismo Real (Cierre de la Unidad 1)

## 1. Introducción y Modelos de Hilos
En los sistemas operativos modernos, la concurrencia y el paralelismo permiten dividir una aplicación en múltiples flujos de ejecución que comparten un mismo espacio de direcciones. Históricamente, la implementación de hilos ha transitado por diferentes modelos arquitectónicos[cite: 40]:
* **Modelo N:1 (Hilos de Usuario):** Múltiples hilos de usuario se gestionan en espacio de usuario y se mapean a un único hilo de kernel. Su ventaja era la ligereza, pero si un hilo realizaba una operación de E/S bloqueante, **todo el proceso se bloqueaba**[cite: 40].
* **Modelo M:N:** Multiplexa un número arbitrario de hilos de usuario sobre un número menor o igual de hilos de kernel, requiriendo un planificador masivo complejo (como en *Go routines* o *Java Virtual Threads*)[cite: 40].
* **Modelo 1:1 (Paralelismo Real):** Cada hilo creado a nivel de usuario se corresponde directamente con una entidad de planificación administrada por el kernel[cite: 40, 41]. 

Linux adoptó el **Modelo 1:1** mediante la biblioteca POSIX Threads (`pthreads`) y la llamada al sistema subyacente `clone()` utilizando banderas como `CLONE_THREAD`, donde el planificador del kernel (CFS) evalúa cada hilo de forma independiente (`TID`) compartiendo el mismo identificador de grupo de procesos (`TGID / PID`)[cite: 41].

---

## 2. Anatomía y Comparativa Cuantitativa: Procesos vs. Hilos
A nivel de recursos, los hilos comparten el espacio de memoria global (`Heap`, `Data`, `Text`), los descriptores de archivos (`files_struct`) y las credenciales del proceso, pero mantienen de forma **privada** su propio stack virtual (por defecto 8 MB), los registros de CPU (`PC`) y su máscara de señales (`sigmask`)[cite: 43].

| Criterio | Proceso (`fork`) | Hilo (`pthread`) |
| :--- | :--- | :--- |
| **Costo de creación** | ~100–300 $\mu s$ (copia de tablas de páginas vía CoW)[cite: 39] | ~20–50 $\mu s$ (solo crea un nuevo `task_struct` y stack)[cite: 39] |
| **Cambio de contexto** | ~1–3 $\mu s$ (requiere *Flush* de TLB por cambio de memoria)[cite: 39] | ~0.5–1 $\mu s$ (mantiene el mismo mapa de memoria)[cite: 39] |
| **Comunicación** | IPC / Pipes (requiere llamadas al kernel)[cite: 39] | Directa a través de la memoria compartida (*heap*)[cite: 39] |
| **Aislamiento** | Fuerte (un fallo crashea solo al proceso)[cite: 39] | Nulo (un fallo como `SIGSEGV` termina a todo el grupo)[cite: 39] |

---

## 3. El Puente de APIs: De Procesos a Hilos
Existe una correspondencia conceptual directa entre las llamadas de control de procesos y la interfaz de gestión de hilos POSIX[cite: 42]:
* `fork()` $\rightarrow$ `pthread_create()` (Creación de la entidad de ejecución)[cite: 42].
* `getpid()` $\rightarrow$ `pthread_self()` (Identificación del flujo actual)[cite: 42].
* `waitpid()` $\rightarrow$ `pthread_join()` (Sincronización y recolección de recursos)[cite: 42].
* `exit()` $\rightarrow$ `pthread_exit()` (Terminación del flujo activo)[cite: 42].

> **Nota crítica:** A diferencia de las llamadas clásicas de POSIX que retornan `-1` y configuran la variable global `errno`, las funciones de la librería `pthreads` **retornan el código de error directamente como un número positivo** en caso de fallo[cite: 42]. Asimismo, un hilo que finaliza sin un `pthread_join()` o sin haber sido desacoplado (`pthread_detach()`) retiene sus recursos en el sistema, constituyendo el equivalente funcional a un **proceso zombie**[cite: 42].

---

## 4. El Problema de la Concurrencia y el Riesgo de *Race Conditions*
Debido a que los hilos comparten el segmento de datos y el *heap*, las operaciones de escritura concurrente sobre variables globales (como un contador incremental `contador++`) generan resultados no deterministas e incorrectos[cite: 47]. Esto ocurre porque una instrucción elemental en C se traduce a nivel de ensamblador en múltiples pasos atómicos no divisibles (**Read $\rightarrow$ Modify $\rightarrow$ Write**)[cite: 47]. 

Este fenómeno se denomina **Race Condition** (Condición de Carrera)[cite: 47], y su resolución mediante mecanismos de exclusión mutua (*Mutex* y *Semáforos*) constituye el núcleo central de estudio de la próxima unidad del curso[cite: 47, 51].

---

## 5. Conclusión
El dominio de la API de `pthreads` y la comprensión del modelo 1:1 en Linux cierran conceptualmente la Unidad 1, conectando la estructura fundamental del núcleo (`task_struct`), la observabilidad a través de `/proc`, la gestión de procesos mediante el ciclo `fork-exec-wait` y, finalmente, el paralelismo de grano fino basado en hilos nativos.
