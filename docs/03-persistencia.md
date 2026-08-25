# 03 - Identificación del mecanismo de persistencia con Process Monitor

## Objetivo

Las comprobaciones realizadas anteriormente no permitieron identificar qué mecanismo recreaba el directorio `FileOverlord` después de reiniciar Windows.

Las búsquedas en:

- claves `Run` y `RunOnce`;
- tareas programadas;
- servicios mediante referencias directas;
- Sysinternals Autoruns;

no mostraron una entrada evidente relacionada con `FileOverlord` o `print.exe`.

Por este motivo se cambió la estrategia de investigación.

En lugar de continuar buscando configuraciones que contuvieran esos nombres, se decidió observar directamente la actividad del sistema durante el arranque.

La herramienta utilizada fue **Sysinternals Process Monitor (Procmon)** mediante su funcionalidad **Boot Logging**.

---

## 1. Por qué utilizar Process Monitor

Process Monitor permite registrar en tiempo real diferentes tipos de actividad de Windows, incluyendo:

- acceso al sistema de archivos;
- creación y modificación de archivos;
- actividad del Registro;
- procesos e hilos;
- carga de componentes.

El objetivo era responder una pregunta concreta:

> **¿Qué proceso crea o escribe en `C:\ProgramData\FileOverlord-...` durante el arranque de Windows?**

Esta pregunta era diferente de las anteriores.

Ya no se buscaba necesariamente una configuración que contuviera la palabra `FileOverlord`, sino el proceso que **realmente realizaba la operación**.

---

## 2. Activación de Boot Logging

Process Monitor se ejecutó con privilegios de administrador.

Posteriormente se activó:

```text
Options → Enable Boot Logging
```

La función Boot Logging permite registrar actividad desde las primeras etapas del inicio de Windows, antes de que Process Monitor sea abierto manualmente después del inicio de sesión.

Una vez activado, se reinició el equipo.

---

## 3. Reaparición del comportamiento

Después del reinicio se permitió que Windows iniciara normalmente y que transcurriera suficiente tiempo para que el comportamiento investigado pudiera reproducirse.

Posteriormente se abrió nuevamente:

```text
Procmon64.exe
```

con privilegios de administrador.

Process Monitor detectó que existía un registro de actividad generado durante el arranque y permitió cargar los eventos recopilados.

---

## 4. Volumen del registro

El Boot Logging produjo una cantidad extremadamente grande de información.

En la investigación se registraron aproximadamente:

```text
14,5 millones de eventos
```

El archivo de captura llegó a ocupar varios gigabytes.

Esto demostró una característica importante de Process Monitor:

> Registrar toda la actividad del sistema puede generar enormes cantidades de información. El verdadero valor de la herramienta está en aplicar filtros que reduzcan los eventos a aquello que se está investigando.

No era práctico revisar manualmente millones de eventos.

Por ello se utilizó `FileOverlord` como indicador para reducir el conjunto de datos.

---

## 5. Filtrado de eventos

Se aplicó un filtro sobre la columna `Path` para mostrar únicamente operaciones relacionadas con:

```text
FileOverlord
```

El filtrado redujo drásticamente la cantidad de eventos visibles y permitió observar qué procesos estaban interactuando con el directorio.

En los resultados apareció repetidamente:

```text
svchost.exe
```

realizando operaciones sobre una ruta similar a:

```text
C:\ProgramData\FileOverlord-{GUID}\...
```

Entre las operaciones observadas se encontraban:

```text
CreateFile
WriteFile
CloseFile
QueryBasicInformationFile
FlushBuffersFile
```

Este fue uno de los hallazgos más importantes de toda la investigación.

---

## 6. Evidencia de creación de archivos

Los eventos mostraban a `svchost.exe` realizando operaciones de escritura dentro del nuevo directorio `FileOverlord`.

Entre ellas se observaron múltiples operaciones:

```text
WriteFile
```

con resultado:

```text
SUCCESS
```

Por tanto, ya no se trataba únicamente de una correlación temporal.

Process Monitor mostraba directamente que el proceso asociado estaba realizando operaciones de creación y escritura sobre los archivos del directorio.

El proceso observado utilizaba en esa sesión el PID:

```text
6608
```

> El PID no identifica permanentemente al proceso. Los identificadores de proceso son asignados dinámicamente por Windows y pueden cambiar después de cada ejecución o reinicio.

El PID fue útil únicamente para relacionar los eventos observados en esa captura con el servicio que se encontraba ejecutándose en ese momento.

---

## 7. De `svchost.exe` al servicio responsable

Encontrar `svchost.exe` todavía no resolvía completamente el problema.

`svchost.exe` es un proceso utilizado por Windows para hospedar servicios.

Por tanto, la nueva pregunta fue:

> **¿Qué servicio estaba ejecutándose dentro del `svchost.exe` con PID 6608?**

Se utilizó PowerShell:

```powershell
Get-CimInstance Win32_Service |
Where-Object { $_.ProcessId -eq 6608 } |
Select-Object Name, DisplayName, State, StartMode, PathName
```

También se realizó una comprobación mediante:

```cmd
tasklist /svc /fi "PID eq 6608"
```

Ambas consultas permitieron asociar el proceso observado con un servicio denominado:

```text
ADSISvc_c3960c
```

Su nombre visible era:

```text
ADSIAgnt_6c06b1
```

En ese momento el servicio se encontraba:

```text
Running
```

y configurado con inicio:

```text
Auto
```

Su `PathName` utilizaba:

```text
C:\Windows\SysWOW64\svchost.exe -k DcomLaunch
```

---

## 8. Por qué la búsqueda inicial de servicios no lo encontró

En la fase anterior se habían buscado servicios cuya propiedad `PathName` contuviera:

```text
FileOverlord
```

o:

```text
print.exe
```

Sin embargo, este servicio aparecía ejecutándose mediante:

```text
svchost.exe -k DcomLaunch
```

Por tanto, su `PathName` no contenía ninguno de los indicadores utilizados inicialmente.

Esto explica por qué una consulta como:

```powershell
Get-CimInstance Win32_Service |
Where-Object {
    $_.PathName -match "FileOverlord|print.exe"
}
```

podía devolver cero resultados aunque existiera un servicio involucrado.

Este fue un punto importante de la investigación:

> **Buscar únicamente el nombre del archivo problemático puede ser insuficiente cuando la ejecución se produce indirectamente mediante otro componente.**

---

## 9. Inspección de la configuración del servicio

Una vez identificado el servicio, se examinó su configuración:

```cmd
sc.exe qc ADSISvc_c3960c
```

También se consultó directamente su clave del Registro:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\ADSISvc_c3960c" /s
```

La configuración mostraba, entre otros datos:

```text
Type        : 0x20
Start       : 0x2
ObjectName  : LocalSystem
Description : Active Directory Service Interface
```

y una subclave:

```text
HKLM\SYSTEM\CurrentControlSet\Services\ADSISvc_c3960c\Parameters
```

que contenía:

```text
ServiceDll    REG_EXPAND_SZ    C:\Windows\SysWOW64\adsiedit.dll
```

Esto permitió identificar qué DLL estaba siendo cargada por el servicio hospedado mediante `svchost.exe`.

---

## 10. Inspección de `adsiedit.dll`

Antes de eliminar el servicio o modificar archivos del sistema se investigó:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

Se consultaron sus metadatos:

```powershell
(Get-Item "C:\Windows\SysWOW64\adsiedit.dll").VersionInfo |
Format-List CompanyName, FileDescription, ProductName, ProductVersion, OriginalFilename
```

Los metadatos observados indicaban:

```text
CompanyName      : Microsoft Corporation
FileDescription  : ADSI Edit
ProductName      : Microsoft Windows Operating System
OriginalFilename : adsiedit.dll
```

También se calculó su hash SHA-256:

```powershell
Get-FileHash "C:\Windows\SysWOW64\adsiedit.dll" -Algorithm SHA256
```

y se comprobó su firma mediante:

```powershell
Get-AuthenticodeSignature "C:\Windows\SysWOW64\adsiedit.dll" |
Format-List Status, StatusMessage, SignerCertificate
```

La comprobación de Authenticode mostró:

```text
Status : NotSigned
```

---

## 11. Interpretación prudente de la DLL

El resultado anterior debía interpretarse con precaución.

Que `Get-AuthenticodeSignature` mostrara:

```text
NotSigned
```

no era evidencia suficiente por sí sola para afirmar que `adsiedit.dll` fuera maliciosa.

Al mismo tiempo, sus metadatos indicaban que se trataba de un componente relacionado con Microsoft Windows.

Por ello, durante esta etapa **no se eliminó directamente la DLL**.

La evidencia más sólida obtenida hasta ese momento era otra:

1. Procmon observó a `svchost.exe` creando y escribiendo `FileOverlord`.
2. El PID de ese `svchost.exe` se relacionó con `ADSISvc_c3960c`.
3. El servicio estaba configurado para inicio automático.
4. El servicio utilizaba `adsiedit.dll` como `ServiceDll`.

Esto establecía una cadena observable entre el comportamiento y el servicio, pero no justificaba todavía afirmar que todos los componentes relacionados fueran necesariamente maliciosos.

---

## 12. Cadena de evidencia obtenida

La investigación permitió reconstruir la siguiente relación:

```text
Inicio de Windows
       │
       ▼
svchost.exe
PID observado: 6608
       │
       ▼
Servicio ADSISvc_c3960c
DisplayName: ADSIAgnt_6c06b1
       │
       ▼
ServiceDll
C:\Windows\SysWOW64\adsiedit.dll
       │
       ▼
Operaciones observadas por Procmon
       │
       ├── CreateFile
       ├── WriteFile
       ├── FlushBuffersFile
       └── CloseFile
       │
       ▼
C:\ProgramData\FileOverlord-{GUID}
```

Esta cadena proporcionó por primera vez un candidato concreto para explicar la regeneración de `FileOverlord`.

---

## 13. Por qué Procmon fue decisivo

Las herramientas utilizadas anteriormente analizaban principalmente configuraciones conocidas de inicio automático.

Procmon permitió abordar el problema desde otra perspectiva:

```text
No buscar quién dice que iniciará FileOverlord,
sino observar quién realmente crea FileOverlord.
```

Ese cambio fue decisivo.

Las búsquedas anteriores preguntaban esencialmente:

```text
¿Dónde aparece escrito "FileOverlord"?
```

Boot Logging permitió preguntar:

```text
¿Quién está modificando C:\ProgramData cuando FileOverlord aparece?
```

La segunda pregunta produjo la evidencia necesaria para continuar.

---

## 14. Limitaciones de la evidencia

Es importante diferenciar entre hechos observados e interpretaciones.

### Observado directamente

- `FileOverlord` reaparecía después del reinicio.
- Procmon registró operaciones sobre el directorio durante el arranque.
- Las operaciones fueron realizadas por `svchost.exe`.
- El PID observado fue `6608`.
- Ese PID estaba asociado en ese momento con `ADSISvc_c3960c`.
- El servicio estaba configurado con inicio automático.
- Su configuración indicaba `adsiedit.dll` como `ServiceDll`.

### No demostrado únicamente por estos datos

Estos datos, por sí solos, no permiten afirmar que:

- todos los archivos denominados `adsiedit.dll` sean maliciosos;
- cualquier servicio ADSI con nombre similar sea malicioso;
- cualquier proceso `print.exe` sea malicioso;
- cualquier directorio denominado `FileOverlord` tenga necesariamente el mismo origen.

La documentación describe únicamente el comportamiento observado en el sistema investigado.

---

## Conclusión de esta fase

Process Monitor permitió identificar el punto que las búsquedas estáticas no habían revelado.

El proceso:

```text
svchost.exe
```

fue observado creando y escribiendo el directorio `FileOverlord` durante el arranque.

El PID correspondiente fue relacionado con:

```text
ADSISvc_c3960c
```

configurado para inicio automático y utilizando:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

como `ServiceDll`.

Con un mecanismo concreto identificado, la siguiente fase consistió en **contenerlo de manera controlada**, verificar si `FileOverlord` dejaba de regenerarse y, solamente después de confirmar el resultado, proceder con su eliminación.

---

[← Investigación inicial](02-investigacion.md) | [Volver al README](../README.md) | [Siguiente: contención y eliminación →](04-contencion-y-eliminacion.md)
