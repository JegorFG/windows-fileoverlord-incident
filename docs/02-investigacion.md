# 02 - Investigación del mecanismo de persistencia

## Objetivo

Después de comprobar que finalizar `print.exe` y eliminar el directorio `FileOverlord` no solucionaban permanentemente el problema, la investigación se orientó a identificar qué componente estaba provocando su reaparición.

La hipótesis principal era la existencia de algún mecanismo de inicio automático o persistencia.

En esta fase se revisaron procesos, tareas programadas, servicios, claves comunes del Registro y entradas de inicio mediante Sysinternals Autoruns.

---

## 1. Confirmación del proceso y su ruta

Se utilizó PowerShell con privilegios de administrador para localizar procesos cuya ruta de ejecución contuviera el término `FileOverlord`.

```powershell
Get-CimInstance Win32_Process |
Where-Object {
    $_.ExecutablePath -like "*FileOverlord*" -or
    $_.CommandLine -like "*FileOverlord*"
} |
Select-Object ProcessId, Name, ExecutablePath, CommandLine
```

La consulta permitió identificar un proceso similar a:

```text
ProcessId      : 13656
Name           : print.exe
ExecutablePath : C:\ProgramData\FileOverlord-{GUID}\print.exe
```

> El PID es temporal y puede cambiar en cada ejecución.  
> El `{GUID}` se utiliza en esta documentación para representar el identificador presente en el nombre del directorio.

Esto confirmó que el proceso observado en el Administrador de tareas estaba siendo ejecutado directamente desde `C:\ProgramData\FileOverlord-...`.

---

## 2. Inspección de directorios FileOverlord

Se buscaron directorios relacionados dentro de `C:\ProgramData`:

```powershell
Get-ChildItem "C:\ProgramData" -Force |
Where-Object { $_.Name -like "*FileOverlord*" } |
Select-Object Name, CreationTime, LastWriteTime
```

Durante distintas etapas de la investigación aparecieron directorios `FileOverlord` creados nuevamente después de haber eliminado o aislado versiones anteriores.

Esto reforzó la hipótesis de que otro componente estaba reconstruyendo los archivos.

---

## 3. Investigación del proceso padre

Una de las siguientes preguntas fue:

> ¿Quién inició `print.exe`?

Se obtuvo información del proceso y de su `ParentProcessId`:

```powershell
$p = Get-CimInstance Win32_Process -Filter "ProcessId=13656"

$p |
Select-Object ProcessId, ParentProcessId, CreationDate, ExecutablePath, CommandLine

Get-CimInstance Win32_Process -Filter "ProcessId=$($p.ParentProcessId)" |
Select-Object ProcessId, ParentProcessId, Name, CreationDate, ExecutablePath, CommandLine
```

En la observación realizada, el proceso padre correspondía a:

```text
explorer.exe
```

Este dato, por sí solo, **no identificaba todavía el mecanismo original de persistencia**.

El proceso padre observado en un momento concreto no necesariamente explica qué componente creó previamente los archivos o qué mecanismo provocó toda la cadena de ejecución.

Por ello fue necesario continuar investigando.

---

## 4. Búsqueda en tareas programadas

Se revisaron las tareas programadas buscando referencias a `FileOverlord` o `print.exe`.

```powershell
Get-ScheduledTask |
Where-Object {
    $_.Actions.Execute -match "FileOverlord|print.exe" -or
    $_.Actions.Arguments -match "FileOverlord|print.exe"
} |
Select-Object TaskName, TaskPath, State, Actions
```

### Resultado

No se encontraron coincidencias relevantes.

Esto redujo la probabilidad de que la persistencia estuviera implementada mediante una tarea programada convencional.

Sin embargo, un resultado negativo no demuestra por sí mismo que no exista otro mecanismo de ejecución.

---

## 5. Búsqueda entre servicios

También se revisaron servicios cuya ruta contuviera referencias directas a `FileOverlord` o `print.exe`.

```powershell
Get-CimInstance Win32_Service |
Where-Object {
    $_.PathName -match "FileOverlord|print.exe"
} |
Select-Object Name, DisplayName, State, StartMode, PathName
```

### Resultado inicial

No aparecieron coincidencias directas.

Esto resultaría particularmente importante más adelante.

El mecanismo finalmente investigado **no necesitaba mostrar `FileOverlord` directamente en el `PathName` del servicio**, por lo que una búsqueda basada únicamente en esas palabras podía no detectarlo.

---

## 6. Búsqueda en claves Run y RunOnce

Se revisaron las ubicaciones habituales utilizadas para iniciar aplicaciones durante el inicio de sesión:

```powershell
$paths = @(
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKCU:\Software\Microsoft\Windows\CurrentVersion\RunOnce",
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\Run",
    "HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce"
)

foreach ($path in $paths) {
    if (Test-Path $path) {
        Get-ItemProperty $path | Format-List
    }
}
```

Se observaron aplicaciones legítimas configuradas para iniciarse con Windows, pero no se encontró una referencia evidente a:

```text
FileOverlord
```

ni a:

```text
print.exe
```

---

## 7. Búsqueda adicional en el Registro

También se realizaron búsquedas por texto directamente mediante `reg query`.

Ejemplos:

```cmd
reg query HKCU /f "FileOverlord" /s
reg query HKLM /f "FileOverlord" /s
```

También se buscó el identificador concreto utilizado por uno de los directorios observados:

```cmd
reg query HKCU /f "{GUID}" /s
reg query HKLM /f "{GUID}" /s
```

### Resultado

Las consultas no encontraron coincidencias útiles.

Esto indicaba que el nombre `FileOverlord` y el identificador del directorio no estaban apareciendo directamente en las ubicaciones del Registro examinadas.

---

## 8. Uso de Sysinternals Autoruns

Para ampliar la investigación se utilizó **Sysinternals Autoruns** con privilegios de administrador.

Autoruns permite visualizar numerosos mecanismos de inicio automático, entre ellos:

- Logon;
- Explorer;
- Scheduled Tasks;
- Services;
- Drivers;
- Winlogon;
- AppInit;
- Image Hijacks;
- WMI;
- otros mecanismos de autoarranque.

Se realizaron búsquedas utilizando:

```text
FileOverlord
```

y posteriormente otros indicadores conocidos durante la investigación.

### Resultado

Las búsquedas no mostraron una entrada evidente que explicara la recreación de `FileOverlord`.

Este fue otro resultado negativo importante.

---

## 9. Lo que sabíamos hasta este momento

Después de estas comprobaciones existía una situación aparentemente contradictoria:

- `print.exe` podía ser finalizado;
- el directorio `FileOverlord` podía eliminarse;
- no había una tarea programada evidente;
- las claves habituales `Run` y `RunOnce` no mostraban el origen;
- las búsquedas directas entre servicios no mostraban `FileOverlord`;
- Autoruns tampoco revelaba una entrada evidente;
- pero después de reiniciar Windows, `FileOverlord` podía volver a aparecer.

Por tanto, buscar únicamente **dónde estaba registrado el nombre `FileOverlord`** no estaba siendo suficiente.

Era necesario observar directamente **qué proceso creaba el directorio y escribía sus archivos durante el arranque**.

---

## 10. Cambio de estrategia

En lugar de continuar buscando únicamente configuraciones estáticas, se decidió observar el comportamiento del sistema durante el inicio de Windows.

Para ello se utilizó:

```text
Sysinternals Process Monitor (Procmon)
```

con su funcionalidad:

```text
Boot Logging
```

La pregunta que debía responder la siguiente fase era mucho más concreta:

> **¿Qué proceso accede, crea o escribe en `C:\ProgramData\FileOverlord-...` durante el arranque de Windows?**

Este cambio —de buscar configuraciones a observar comportamiento— fue el que permitió avanzar en la investigación.

---

## Conclusión de esta fase

Las comprobaciones iniciales no identificaron el mecanismo responsable mediante los métodos habituales de persistencia.

Esto no significaba que el mecanismo no existiera.

Significaba que las búsquedas realizadas estaban basadas principalmente en indicadores conocidos como:

```text
FileOverlord
print.exe
{GUID}
```

y el componente responsable podía no contener ninguno de esos indicadores en su propia configuración.

La investigación pasó entonces de un enfoque basado en **búsqueda de nombres** a uno basado en **observación de eventos del sistema**.

La siguiente fase documenta cómo Process Monitor permitió observar la creación de `FileOverlord` durante el arranque.

---

[← Detección inicial](01-deteccion-inicial.md) | [Volver al README](../README.md) | [Siguiente: análisis con Procmon →](03-persistencia.md)
