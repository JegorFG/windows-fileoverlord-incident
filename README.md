# Windows FileOverlord Incident

Documentación de la investigación, identificación, contención y eliminación de un incidente detectado en un equipo con Windows 11, relacionado con un proceso `print.exe` ejecutado desde un directorio denominado `FileOverlord` dentro de `C:\ProgramData`.

> [!WARNING]
> Este repositorio documenta un incidente real con fines educativos y de diagnóstico.
>
> **No contiene ni distribuye los ejecutables encontrados durante la investigación.**
>
> Los nombres, rutas y configuraciones documentados corresponden al sistema analizado y no deben utilizarse como criterio único para eliminar archivos o servicios en otros equipos.

## Resumen del incidente

El problema se detectó inicialmente debido a un consumo anormal de memoria RAM por un proceso que aparecía en el Administrador de tareas como:

```text
Utilidad de impresión
```

El proceso llegó a utilizar aproximadamente **2,3 GB de memoria RAM**, provocando un consumo total de memoria cercano al límite disponible del sistema.

Durante la investigación se determinó que el proceso correspondía a:

```text
print.exe
```

y estaba siendo ejecutado desde una ruta similar a:

```text
C:\ProgramData\FileOverlord-{GUID}\print.exe
```

Finalizar manualmente el proceso o eliminar el directorio no solucionaba permanentemente el problema.

Después de reiniciar Windows, `FileOverlord` podía volver a ser creado y `print.exe` volvía a ejecutarse.

---

## Hallazgo principal

Las comprobaciones iniciales de mecanismos habituales de persistencia no mostraron el origen del comportamiento.

Se revisaron, entre otros:

* aplicaciones de inicio;
* claves `Run` y `RunOnce`;
* tareas programadas;
* servicios mediante búsquedas directas;
* Sysinternals Autoruns.

Al no encontrar una referencia evidente, la investigación cambió de estrategia.

En lugar de continuar buscando configuraciones que contuvieran el nombre `FileOverlord`, se utilizó **Sysinternals Process Monitor (Procmon)** con **Boot Logging** para observar directamente qué proceso creaba los archivos durante el arranque.

Procmon registró operaciones de creación y escritura realizadas por:

```text
svchost.exe
```

sobre:

```text
C:\ProgramData\FileOverlord-{GUID}
```

El PID observado durante esa captura fue posteriormente relacionado con el servicio:

```text
ADSISvc_c3960c
```

cuyo nombre visible era:

```text
ADSIAgnt_6c06b1
```

El servicio se encontraba configurado para inicio automático.

Después de deshabilitarlo y reiniciar Windows, `FileOverlord` dejó de regenerarse durante las comprobaciones realizadas.

Posteriormente se respaldó la configuración del servicio y se eliminó su registro.

---

## Cadena de investigación

El incidente puede resumirse mediante la siguiente secuencia:

```text
"Utilidad de impresión"
        │
        ▼
     print.exe
        │
        ▼
C:\ProgramData\FileOverlord-{GUID}
        │
        ▼
El directorio reaparece después del reinicio
        │
        ▼
Búsquedas tradicionales sin coincidencias concluyentes
        │
        ▼
Process Monitor + Boot Logging
        │
        ▼
svchost.exe escribiendo FileOverlord
        │
        ▼
PID asociado con ADSISvc_c3960c
        │
        ▼
Servicio deshabilitado
        │
        ▼
FileOverlord deja de regenerarse
        │
        ▼
Respaldo y eliminación del servicio
        │
        ▼
Reparación y verificación del sistema
```

---

## Documentación completa

La investigación se encuentra dividida en cinco etapas.

### 1. [Detección inicial](docs/01-deteccion-inicial.md)

Describe el síntoma original, el consumo anormal de memoria y cómo se identificó que la aparente **"Utilidad de impresión"** correspondía a `print.exe` ejecutándose desde `FileOverlord`.

### 2. [Investigación del mecanismo de persistencia](docs/02-investigacion.md)

Documenta las búsquedas realizadas mediante PowerShell, tareas programadas, Registro, servicios y Sysinternals Autoruns.

También explica por qué los resultados negativos fueron importantes para cambiar la estrategia de investigación.

### 3. [Identificación de la persistencia con Process Monitor](docs/03-persistencia.md)

Documenta el uso de **Procmon Boot Logging**, el filtrado de millones de eventos y la identificación de `svchost.exe` realizando operaciones de escritura sobre `FileOverlord`.

Posteriormente relaciona el PID observado con `ADSISvc_c3960c`.

### 4. [Contención y eliminación](docs/04-contencion-y-eliminacion.md)

Describe la estrategia gradual utilizada:

```text
identificar
→ deshabilitar
→ comprobar
→ respaldar
→ eliminar
```

También documenta la eliminación de los directorios residuales después de comprobar que no existían procesos ejecutándose desde ellos.

### 5. [Reparación y verificación](docs/05-reparacion-y-verificacion.md)

Documenta las comprobaciones realizadas mediante DISM y System File Checker, así como la verificación posterior de:

* procesos;
* directorios;
* servicio;
* comportamiento después del reinicio.

---

## Herramientas utilizadas

Durante la investigación se utilizaron:

* Windows Task Manager
* PowerShell
* Windows Service Control (`sc.exe`)
* Registry Query (`reg.exe`)
* Sysinternals Autoruns
* Sysinternals Process Monitor
* DISM
* System File Checker (`sfc`)

---

## Resultado

Después de la intervención:

* `ADSISvc_c3960c` fue identificado y relacionado con el proceso observado por Procmon.
* El servicio fue deshabilitado antes de eliminarlo.
* La configuración del servicio fue respaldada antes de su eliminación.
* Los directorios residuales de `FileOverlord` fueron eliminados.
* No se encontraron posteriormente procesos ejecutándose desde rutas `FileOverlord`.
* `FileOverlord` dejó de regenerarse durante las comprobaciones realizadas.
* DISM completó correctamente su operación.
* SFC informó haber encontrado y reparado archivos dañados.
* No se eliminó manualmente `svchost.exe`.
* No se eliminó manualmente `C:\Windows\SysWOW64\adsiedit.dll`.

---

## Observaciones vs. conclusiones

Este repositorio intenta diferenciar cuidadosamente entre **hechos observados** e **interpretaciones**.

La investigación proporciona evidencia de que:

1. `FileOverlord` se regeneraba después del reinicio.
2. Procmon observó un `svchost.exe` escribiendo sus archivos.
3. El PID observado estaba asociado en ese momento con `ADSISvc_c3960c`.
4. Al deshabilitar el servicio, el comportamiento dejó de reproducirse durante las comprobaciones realizadas.
5. El servicio fue posteriormente eliminado.

Sin embargo, estos resultados **no permiten generalizar** que cualquier archivo o servicio con nombres similares tenga necesariamente el mismo comportamiento.

---

## Importante

No utilice comandos como:

```text
Remove-Item -Recurse -Force
sc.exe delete
reg delete
```

únicamente porque encuentre nombres similares en otro equipo.

Antes de eliminar componentes del sistema debe determinarse:

* qué proceso está realizando la actividad;
* qué servicio o mecanismo lo inicia;
* qué archivos utiliza;
* si pertenece a Windows o a software legítimamente instalado;
* y qué evidencia existe para justificar su eliminación.

Especialmente, **no debe eliminarse `svchost.exe` basándose en este caso**.

`svchost.exe` es un componente fundamental utilizado por Windows para hospedar servicios.

---

## Principal aprendizaje

La parte decisiva de la investigación no fue preguntar únicamente:

> **¿Dónde está registrado FileOverlord?**

sino:

> **¿Qué proceso está creando FileOverlord?**

El cambio de una investigación basada en nombres a una basada en **comportamiento observado** permitió identificar el mecanismo que las búsquedas iniciales no habían revelado.

En este caso, **Process Monitor con Boot Logging fue la herramienta decisiva**.

---

## Estado

**Incidente contenido, mecanismo de persistencia identificado y eliminado, y sistema verificado posteriormente.**

La documentación se conserva con fines educativos, técnicos y como referencia para investigaciones de comportamiento similar.
