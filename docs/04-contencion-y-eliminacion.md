# 04 - Contención y eliminación del mecanismo de persistencia

## Objetivo

Después de relacionar mediante Process Monitor el comportamiento de `FileOverlord` con el servicio:

```text
ADSISvc_c3960c
```

se inició una fase de contención controlada.

El objetivo inmediato no era eliminar archivos indiscriminadamente, sino comprobar una hipótesis:

> **Si el servicio identificado participa en la regeneración de `FileOverlord`, impedir su ejecución debería detener ese comportamiento.**

Por este motivo se siguió una estrategia gradual:

1. intentar detener el servicio;
2. deshabilitar su inicio automático;
3. verificar su estado;
4. comprobar si `FileOverlord` continuaba ejecutándose o regenerándose;
5. conservar una copia de seguridad de la configuración;
6. eliminar finalmente el servicio identificado.

---

## 1. Estado inicial del servicio

Durante la investigación anterior se había determinado que:

```text
ADSISvc_c3960c
```

se encontraba:

```text
Running
```

y configurado con:

```text
StartMode : Auto
```

El servicio estaba asociado en la captura analizada con un proceso `svchost.exe` y utilizaba:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

como `ServiceDll`.

Antes de realizar cambios se había recopilado información sobre su configuración mediante herramientas como:

```powershell
Get-CimInstance Win32_Service
```

```cmd
sc.exe qc ADSISvc_c3960c
```

y:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\ADSISvc_c3960c" /s
```

---

## 2. Intento de detener el servicio

Desde PowerShell ejecutado como administrador se intentó detener el servicio:

```powershell
Stop-Service -Name "ADSISvc_c3960c" -Force
```

La operación produjo un error indicando que el servicio no podía detenerse mediante esa solicitud.

El mensaje observado fue equivalente a:

```text
No se puede detener el servicio ADSIAgnt_6c06b1 (ADSISvc_c3960c)
```

Este resultado era importante.

El hecho de que `Stop-Service` no pudiera detenerlo inmediatamente no justificaba eliminar archivos o finalizar indiscriminadamente procesos `svchost.exe`, ya que estos procesos pueden hospedar componentes legítimos de Windows.

---

## 3. Deshabilitación del inicio automático

A continuación se modificó el tipo de inicio del servicio:

```powershell
Set-Service -Name "ADSISvc_c3960c" -StartupType Disabled
```

Posteriormente se comprobó su estado:

```powershell
Get-Service ADSISvc_c3960c |
Select-Object Name, Status, StartType
```

El resultado observado fue:

```text
Name             Status    StartType
----             ------    ---------
ADSISvc_c3960c   Stopped   Disabled
```

Esto proporcionó una primera medida de contención.

El servicio ya no se encontraba en ejecución y había quedado configurado para no iniciarse automáticamente.

---

## 4. Comprobación de procesos relacionados con FileOverlord

Una vez detenido y deshabilitado el servicio se buscó cualquier proceso cuyo ejecutable procediera de una ruta relacionada con `FileOverlord`.

Se utilizó:

```powershell
Get-CimInstance Win32_Process |
Where-Object { $_.ExecutablePath -like "*FileOverlord*" } |
Select-Object ProcessId, Name, ExecutablePath
```

La consulta no devolvió resultados.

Esto indicaba que, en ese momento, no existían procesos detectables ejecutándose desde una ruta que contuviera:

```text
FileOverlord
```

Esta comprobación se realizó nuevamente en etapas posteriores de la investigación.

---

## 5. Localización de directorios residuales

También se buscaron directorios relacionados con `FileOverlord` dentro de:

```text
C:\ProgramData
```

mediante:

```powershell
Get-ChildItem "C:\ProgramData" -Force |
Where-Object { $_.Name -like "*FileOverlord*" } |
Select-Object Name, CreationTime, LastWriteTime
```

Se localizaron tres directorios residuales:

```text
FileOverlord-QUARANTINE
FileOverlord-QUARANTINE-2
FileOverlord-QUARANTINE-3
```

Las fechas observadas mostraban que habían sido creados durante diferentes momentos de la investigación.

Estos directorios correspondían a elementos conservados o renombrados durante las etapas de diagnóstico y contención.

---

## 6. Eliminación de directorios residuales

Después de comprobar que no existían procesos ejecutándose desde esas rutas, se eliminaron los directorios residuales:

```powershell
Remove-Item "C:\ProgramData\FileOverlord-QUARANTINE" -Recurse -Force
Remove-Item "C:\ProgramData\FileOverlord-QUARANTINE-2" -Recurse -Force
Remove-Item "C:\ProgramData\FileOverlord-QUARANTINE-3" -Recurse -Force
```

Posteriormente se repitió la búsqueda:

```powershell
Get-ChildItem "C:\ProgramData" -Force |
Where-Object { $_.Name -like "*FileOverlord*" }
```

La consulta no devolvió resultados.

Esto confirmó que los directorios residuales habían sido eliminados.

> La eliminación de estos directorios se realizó después de haber recopilado la información necesaria para la investigación y comprobar que no existían procesos activos ejecutándose desde ellos.

---

## 7. Reparación de la imagen de Windows

Debido a que la investigación había involucrado componentes ubicados dentro de directorios del sistema y existía incertidumbre sobre posibles modificaciones, se decidió comprobar posteriormente la integridad de Windows.

Se ejecutó:

```cmd
DISM /Online /Cleanup-Image /RestoreHealth
```

DISM completó el proceso al:

```text
100.0%
```

y mostró:

```text
The restore operation completed successfully.
The operation completed successfully.
```

Esto indicó que la operación de reparación de la imagen de Windows finalizó correctamente.

---

## 8. Comprobación mediante System File Checker

Después de DISM se ejecutó:

```cmd
sfc /scannow
```

El análisis alcanzó:

```text
100%
```

y Windows informó:

```text
Protección de recursos de Windows encontró archivos dañados y los reparó correctamente.
```

Este resultado confirmó que existían archivos del sistema que requerían reparación y que SFC pudo repararlos.

Los detalles quedaron registrados por Windows en:

```text
C:\Windows\Logs\CBS\CBS.log
```

---

## 9. Consulta del registro de reparación

Para revisar información relevante del registro de SFC se utilizó PowerShell:

```powershell
Select-String -Path "$env:windir\Logs\CBS\CBS.log" `
-Pattern "\[SR\].*Repairing|\[SR\].*Repaired|\[SR\].*Cannot repair" |
Select-Object -Last 50
```

Entre los registros apareció actividad de reparación identificada mediante:

```text
[SR] Repairing
```

La finalidad de esta comprobación era verificar que la reparación informada por SFC también estuviera reflejada en el registro CBS.

---

## 10. Revisión adicional de adsiedit.dll

Después de ejecutar DISM y SFC se volvió a consultar:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

mediante:

```powershell
Get-Item "C:\Windows\SysWOW64\adsiedit.dll" |
Select-Object FullName, Length, LastWriteTime
```

El archivo continuaba presente.

También se revisaron nuevamente sus metadatos:

```powershell
(Get-Item "C:\Windows\SysWOW64\adsiedit.dll").VersionInfo |
Format-List CompanyName, FileDescription, ProductName, ProductVersion
```

Los datos observados indicaban:

```text
CompanyName     : Microsoft Corporation
FileDescription : ADSI Edit
ProductName     : Microsoft Windows Operating System
ProductVersion  : 10.0.19041.1
```

Por esta razón se mantuvo la decisión de **no eliminar manualmente `adsiedit.dll`**.

La investigación se concentró en eliminar la configuración del servicio que había sido relacionada con el comportamiento observado, no en borrar directamente un archivo situado dentro de Windows cuya legitimidad no se había descartado de forma concluyente.

---

## 11. Revisión de la configuración del servicio

Antes de eliminar el servicio se volvió a consultar su configuración en el Registro:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\ADSISvc_c3960c" /s
```

Entre los valores observados se encontraban:

```text
Start        REG_DWORD      0x4
ImagePath    REG_EXPAND_SZ  %SystemRoot%\SysWOW64\svchost.exe -k DcomLaunch
DisplayName  REG_SZ         ADSIAgnt_6c06b1
ObjectName   REG_SZ         LocalSystem
Description  REG_SZ         Active Directory Service Interface
```

El valor:

```text
Start = 0x4
```

era consistente con el estado:

```text
Disabled
```

También permanecía la configuración:

```text
Parameters\ServiceDll
```

apuntando a:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

---

## 12. Copia de seguridad de la configuración

Antes de eliminar definitivamente el servicio se exportó su clave del Registro.

Se utilizó:

```cmd
reg export "HKLM\SYSTEM\CurrentControlSet\Services\ADSISvc_c3960c" "%USERPROFILE%\Desktop\ADSISvc_c3960c-backup.reg"
```

Windows confirmó:

```text
La operación se completó correctamente.
```

Esto creó una copia de la configuración existente antes de proceder con la eliminación.

> La copia `.reg` contiene configuración del sistema y debe conservarse únicamente como evidencia o respaldo técnico. No debe importarse en otro equipo ni utilizarse como procedimiento de instalación.

---

## 13. Eliminación del servicio

Después de:

- identificar el servicio;
- documentar su configuración;
- deshabilitarlo;
- comprobar que estaba detenido;
- verificar la ausencia de procesos `FileOverlord`;
- y conservar una copia de seguridad;

se procedió a eliminar su registro como servicio mediante:

```cmd
sc.exe delete ADSISvc_c3960c
```

Windows respondió:

```text
[SC] DeleteService CORRECTO
```

---

## 14. Verificación de la eliminación

Después de eliminarlo se realizó una comprobación mediante:

```powershell
Get-Service ADSISvc_c3960c -ErrorAction SilentlyContinue
```

La consulta no devolvió el servicio.

También se consultó directamente el Registro:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\ADSISvc_c3960c"
```

Windows respondió:

```text
ERROR: El sistema no ha podido encontrar la clave o el valor del Registro especificados.
```

Esto confirmó que la entrada correspondiente al servicio ya no estaba presente en:

```text
HKLM\SYSTEM\CurrentControlSet\Services
```

---

## 15. Cadena de contención aplicada

El procedimiento seguido puede resumirse así:

```text
Servicio identificado
        │
        ▼
Intento de Stop-Service
        │
        ▼
Deshabilitación
        │
        ▼
Stopped / Disabled
        │
        ▼
Verificación de procesos FileOverlord
        │
        ▼
Eliminación de directorios residuales
        │
        ▼
DISM / RestoreHealth
        │
        ▼
SFC /scannow
        │
        ▼
Respaldo de configuración del servicio
        │
        ▼
sc.exe delete ADSISvc_c3960c
        │
        ▼
Verificación de ausencia
```

---

## 16. Por qué no se eliminó inmediatamente el servicio

Una parte importante del procedimiento fue evitar pasar directamente de:

```text
"servicio sospechoso"
```

a:

```text
"eliminar servicio"
```

La secuencia utilizada permitió obtener evidencia adicional.

Primero se deshabilitó el mecanismo identificado y posteriormente se comprobó el comportamiento del sistema.

Esta metodología reduce el riesgo de eliminar accidentalmente un componente legítimo y permite comprobar si la hipótesis planteada durante la investigación era correcta.

---

## 17. Consideraciones de seguridad

Los comandos documentados en esta sección modifican configuraciones importantes de Windows.

En particular:

```powershell
Set-Service
```

```powershell
Remove-Item -Recurse -Force
```

```cmd
reg export
```

y:

```cmd
sc.exe delete
```

pueden producir consecuencias graves si se utilizan sobre servicios o rutas incorrectas.

Por ello, este documento **no debe interpretarse como una lista genérica de comandos para eliminar servicios desconocidos**.

Antes de ejecutar acciones equivalentes en otro sistema debe determinarse:

- qué proceso genera el comportamiento;
- qué servicio está relacionado con ese proceso;
- qué archivos carga;
- qué configuración utiliza;
- si pertenece legítimamente al sistema o a software instalado;
- y qué evidencia existe para justificar su contención.

Los nombres documentados corresponden únicamente al equipo investigado.

---

## Resultado de la fase

Al finalizar esta etapa:

- `ADSISvc_c3960c` había sido detenido y deshabilitado;
- su configuración había sido respaldada;
- posteriormente el servicio fue eliminado;
- su clave dejó de existir en `HKLM\SYSTEM\CurrentControlSet\Services`;
- no se encontraron procesos ejecutándose desde rutas `FileOverlord`;
- los directorios residuales de `FileOverlord` fueron eliminados;
- DISM completó correctamente la reparación de la imagen;
- SFC detectó archivos dañados y los reparó correctamente;
- `adsiedit.dll` no fue eliminado manualmente.

La siguiente fase consistió en comprobar el estado del sistema después de la contención, revisar los resultados de Microsoft Defender Offline y verificar que el mecanismo observado no volviera a aparecer.

---

[← Identificación de persistencia](03-persistencia.md) | [Volver al README](../README.md) | [Siguiente: reparación y análisis de seguridad →](05-reparacion-y-seguridad.md)
