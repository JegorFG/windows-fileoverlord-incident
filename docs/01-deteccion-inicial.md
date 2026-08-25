# 01 - Detección inicial

## Objetivo

Este documento describe cómo se detectó inicialmente el comportamiento anómalo que dio origen a la investigación de `FileOverlord`.

En esta etapa todavía no se conocía el origen del proceso, su mecanismo de ejecución ni la causa de su reaparición después de reiniciar Windows.

El objetivo inicial era determinar qué estaba provocando un consumo anormalmente alto de memoria RAM.

---

## 1. Síntoma inicial

El incidente comenzó al observar un consumo elevado de memoria en el **Administrador de tareas de Windows 11**.

Un proceso identificado visualmente como:

```text
Utilidad de impresión
```

estaba utilizando aproximadamente:

```text
2,3 GB de memoria RAM
```

El consumo era considerablemente superior al esperado para una utilidad aparentemente relacionada con impresión y provocó que el uso total de memoria del equipo alcanzara aproximadamente el **93 %**.

Esto motivó la investigación del proceso.

---

## 2. Primera hipótesis

Por el nombre mostrado por el Administrador de tareas, inicialmente era razonable considerar que el proceso pudiera estar relacionado con:

- el sistema de impresión de Windows;
- algún controlador de impresora;
- el servicio Print Spooler;
- software instalado por el fabricante de alguna impresora.

Sin embargo, finalizar el proceso no solucionaba definitivamente el problema.

El proceso volvía a aparecer posteriormente.

Esto indicaba que probablemente existía otro componente responsable de iniciarlo.

---

## 3. Identificación del ejecutable

Para obtener información más precisa se investigó el proceso desde PowerShell.

Durante la investigación se determinó que el proceso mostrado como **"Utilidad de impresión"** correspondía realmente a:

```text
print.exe
```

Posteriormente se obtuvo su ruta de ejecución, que seguía un patrón similar a:

```text
C:\ProgramData\FileOverlord-{GUID}\print.exe
```

El directorio incluía un identificador variable, por lo que el nombre exacto podía cambiar entre distintas recreaciones.

Este fue el primer indicio importante de que no se trataba simplemente de la utilidad de impresión estándar que inicialmente sugería el Administrador de tareas.

---

## 4. Inspección del directorio

Se examinó el directorio `FileOverlord` ubicado dentro de:

```text
C:\ProgramData
```

El directorio contenía numerosos archivos necesarios para la ejecución de la aplicación, incluyendo DLL, bibliotecas de runtime y otros componentes.

Entre los archivos observados se encontraban componentes relacionados con Qt y runtimes de Microsoft.

También se identificó un ejecutable denominado:

```text
FileOverlord.exe
```

además del proceso:

```text
print.exe
```

Esto sugería que `print.exe` formaba parte de una aplicación o conjunto de archivos considerablemente mayor.

En esta etapa todavía no se conocía qué componente estaba creando el directorio ni qué mecanismo provocaba su ejecución.

---

## 5. Primer intento de contención

Como prueba inicial se procedió a finalizar el proceso `print.exe`.

Posteriormente se eliminó o aisló el directorio `FileOverlord`.

Esto parecía solucionar temporalmente el problema.

Sin embargo, después de reiniciar Windows se observó nuevamente la creación de un directorio con el patrón:

```text
FileOverlord-{GUID}
```

y `print.exe` volvió a ejecutarse.

Por tanto, eliminar únicamente el proceso y sus archivos **no eliminaba la causa del problema**.

---

## 6. Cambio en la investigación

La reaparición después del reinicio modificó la pregunta principal de la investigación.

Ya no se trataba únicamente de determinar:

> ¿Qué es `print.exe`?

La pregunta más importante pasó a ser:

> **¿Qué componente de Windows está reconstruyendo `FileOverlord` y ejecutando nuevamente `print.exe`?**

Esto llevó a investigar posibles mecanismos de persistencia.

Entre ellos:

- aplicaciones de inicio;
- claves `Run` y `RunOnce` del Registro;
- servicios de Windows;
- tareas programadas;
- otros mecanismos de inicio automático.

La investigación de estos mecanismos se documenta en la siguiente etapa.

---

## Conclusión de esta fase

Al finalizar la detección inicial se habían establecido los siguientes hechos:

1. Existía un proceso denominado `print.exe` consumiendo una cantidad anormal de memoria.
2. El Administrador de tareas lo mostraba como **"Utilidad de impresión"**.
3. El ejecutable se encontraba dentro de un directorio `FileOverlord` en `C:\ProgramData`.
4. Finalizar el proceso no solucionaba permanentemente el problema.
5. Eliminar el directorio tampoco impedía su reaparición.
6. El directorio era reconstruido después de reiniciar Windows.
7. Existía, por tanto, algún mecanismo externo responsable de recrear y ejecutar nuevamente sus componentes.

En este punto todavía **no se había identificado el mecanismo responsable**.

Ese hecho es importante para mantener separadas las observaciones comprobadas de las hipótesis que surgieron durante la investigación.

---

[Volver al README](../README.md)
