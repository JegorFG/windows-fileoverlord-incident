# Windows FileOverlord Incident

Documentación de la investigación, contención y eliminación de un incidente detectado en un equipo con Windows 11, relacionado con un proceso `print.exe` ejecutado desde un directorio denominado `FileOverlord` dentro de `C:\ProgramData`.

> [!WARNING]
> Este repositorio documenta un incidente real con fines educativos y de diagnóstico.
> No contiene ni distribuye los ejecutables encontrados durante la investigación.

## Resumen del incidente

El problema se detectó inicialmente debido a un consumo anormal de memoria RAM por un proceso que aparecía en el Administrador de tareas como **"Utilidad de impresión"**.

El proceso llegó a consumir aproximadamente **2,3 GB de memoria RAM**, comportamiento que motivó una investigación para determinar su origen.

Posteriormente se identificó que el proceso correspondía a:

```text
print.exe

y que estaba siendo ejecutado desde una ruta similar a:

```text
C:\ProgramData\FileOverlord-{GUID}\print.exe
```

Finalizar manualmente el proceso o eliminar el directorio no solucionaba el problema de forma permanente: después de reiniciar Windows, el directorio `FileOverlord` podía volver a ser creado y `print.exe` volvía a ejecutarse.

## Hallazgo principal

Las comprobaciones iniciales de los mecanismos habituales de persistencia —claves `Run`/`RunOnce`, tareas programadas y búsqueda mediante Sysinternals Autoruns— no mostraron inicialmente el origen del problema.

Para observar lo que ocurría durante el arranque se utilizó **Sysinternals Process Monitor (Procmon)** con **Boot Logging**.

El registro permitió observar que un proceso:

```text
svchost.exe
```

creaba nuevamente el directorio `FileOverlord` y escribía sus archivos durante el inicio del sistema.

El PID observado fue relacionado posteriormente con un servicio denominado:

```text
ADSISvc_c3960c
```

con nombre visible:

```text
ADSIAgnt_6c06b1
```

El servicio estaba configurado para iniciarse automáticamente y utilizaba:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

como `ServiceDll`.

El servicio fue deshabilitado y posteriormente eliminado después de realizar una copia de seguridad de su configuración.

## Herramientas utilizadas

- Windows Task Manager
- PowerShell
- Microsoft Defender Offline
- Sysinternals Autoruns
- Sysinternals Process Monitor
- DISM
- System File Checker (`sfc`)

## Resultado

Después de la contención y eliminación:

- El servicio identificado quedó eliminado.
- Los directorios `FileOverlord` dejaron de regenerarse.
- No volvió a encontrarse un proceso ejecutándose desde una ruta `FileOverlord`.
- Microsoft Defender Offline puso en cuarentena elementos adicionales detectados durante el análisis.
- DISM completó correctamente la reparación de la imagen de Windows.
- SFC detectó archivos del sistema dañados y los reparó correctamente.

## Documentación

La investigación completa se documentará progresivamente:

1. Detección inicial
2. Investigación del proceso `print.exe`
3. Identificación del mecanismo de persistencia
4. Contención y eliminación
5. Reparación de Windows
6. Verificación final

## Importante

Los nombres, rutas, identificadores y resultados documentados corresponden al equipo analizado.

No debe asumirse que otro sistema que presente un proceso llamado `print.exe`, un directorio denominado `FileOverlord` o nombres similares tenga necesariamente el mismo origen.

Antes de eliminar servicios, archivos o claves del Registro se recomienda identificar su procedencia y conservar una copia de seguridad cuando sea posible.

## Estado

**Incidente contenido y mecanismo de persistencia identificado y eliminado.**

La documentación forense detallada continúa en elaboración.
