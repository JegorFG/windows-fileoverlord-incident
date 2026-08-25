# 05 - Reparación y verificación del sistema

## Objetivo

Después de contener y eliminar el servicio relacionado con la regeneración de `FileOverlord`, la siguiente fase consistió en comprobar que:

1. Windows mantuviera su integridad.
2. El servicio eliminado no reapareciera.
3. El directorio `FileOverlord` no volviera a generarse.
4. No existieran procesos ejecutándose desde rutas relacionadas con `FileOverlord`.
5. El comportamiento anormal observado inicialmente no volviera a presentarse después de reiniciar Windows.

Esta fase fue especialmente importante porque eliminar un componente no demuestra, por sí solo, que el mecanismo de persistencia haya sido neutralizado.

La ausencia del comportamiento después de varios reinicios proporcionaría evidencia adicional para evaluar el resultado de la intervención.

---

## 1. Comprobación de la imagen de Windows con DISM

Durante la investigación se habían examinado servicios y componentes ubicados dentro de directorios del sistema.

Por precaución se utilizó **Deployment Image Servicing and Management (DISM)** para comprobar y reparar la imagen de Windows.

Desde una terminal con privilegios de administrador se ejecutó:

```cmd
DISM /Online /Cleanup-Image /RestoreHealth
```

El proceso alcanzó:

```text
100.0%
```

y finalizó con:

```text
The restore operation completed successfully.
The operation completed successfully.
```

Esto confirmó que DISM pudo completar correctamente la operación de reparación de la imagen.

---

## 2. Comprobación mediante System File Checker

Después de finalizar DISM se ejecutó:

```cmd
sfc /scannow
```

System File Checker realizó la verificación de los archivos protegidos del sistema.

El análisis finalizó al:

```text
100%
```

y Windows informó:

```text
Protección de recursos de Windows encontró archivos dañados y los reparó correctamente.
```

Por tanto, durante esta comprobación Windows determinó que existían archivos protegidos que requerían reparación.

SFC informó que pudo realizarla correctamente.

---

## 3. Interpretación prudente del resultado de SFC

El hecho de que SFC encontrara archivos dañados **no permite determinar automáticamente cuál fue la causa del daño**.

Tampoco demuestra por sí mismo que esos archivos estuvieran relacionados con:

```text
FileOverlord
```

o:

```text
ADSISvc_c3960c
```

Por este motivo, el resultado se documenta como una comprobación de integridad realizada después de la contención y no como evidencia de una relación causal con el incidente.

Esta distinción es importante:

> Que dos hallazgos aparezcan durante una misma investigación no significa necesariamente que uno haya provocado al otro.

---

## 4. Consulta del registro CBS

Los resultados detallados de System File Checker quedan registrados en:

```text
C:\Windows\Logs\CBS\CBS.log
```

Para consultar eventos relacionados con reparación se utilizó:

```powershell
Select-String -Path "$env:windir\Logs\CBS\CBS.log" `
-Pattern "\[SR\].*Repairing|\[SR\].*Repaired|\[SR\].*Cannot repair" |
Select-Object -Last 50
```

La consulta mostró actividad de SFC, incluyendo una entrada:

```text
[SR] Repairing 0 components
```

Este resultado no permitió atribuir una reparación específica a `adsiedit.dll` ni a otro componente investigado.

Por tanto, no se estableció una relación entre la reparación reportada por SFC y el mecanismo de persistencia identificado anteriormente.

---

## 5. Nueva comprobación de adsiedit.dll

Después de DISM y SFC se volvió a comprobar la existencia del archivo:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

mediante:

```powershell
Get-Item "C:\Windows\SysWOW64\adsiedit.dll" |
Select-Object FullName, Length, LastWriteTime
```

También se consultaron nuevamente sus metadatos:

```powershell
(Get-Item "C:\Windows\SysWOW64\adsiedit.dll").VersionInfo |
Format-List CompanyName, FileDescription, ProductName, ProductVersion
```

Los valores observados continuaban identificándolo como:

```text
CompanyName     : Microsoft Corporation
FileDescription : ADSI Edit
ProductName     : Microsoft Windows Operating System
ProductVersion  : 10.0.19041.1
```

Por esta razón el archivo:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

**no fue eliminado manualmente**.

La intervención se mantuvo limitada al servicio cuya actividad había sido relacionada mediante Process Monitor con la creación de `FileOverlord`.

---

## 6. Verificación del servicio después de la contención

Antes de su eliminación definitiva, el servicio había quedado:

```text
Status    : Stopped
StartType : Disabled
```

Posteriormente se eliminó mediante:

```cmd
sc.exe delete ADSISvc_c3960c
```

Después de la eliminación se utilizó:

```powershell
Get-Service ADSISvc_c3960c -ErrorAction SilentlyContinue
```

La consulta no devolvió resultados.

También se comprobó directamente su antigua ubicación en el Registro:

```cmd
reg query "HKLM\SYSTEM\CurrentControlSet\Services\ADSISvc_c3960c"
```

Windows informó que la clave especificada no podía encontrarse.

Esto confirmó que el registro del servicio había sido eliminado.

---

## 7. Verificación de procesos FileOverlord

Para comprobar que ningún proceso relacionado permaneciera activo se utilizó:

```powershell
Get-CimInstance Win32_Process |
Where-Object {
    $_.ExecutablePath -like "*FileOverlord*" -or
    $_.CommandLine -like "*FileOverlord*"
} |
Select-Object ProcessId, Name, ExecutablePath
```

La consulta no devolvió resultados.

Por tanto, en el momento de la comprobación no existían procesos detectables ejecutándose desde rutas relacionadas con:

```text
FileOverlord
```

---

## 8. Verificación de directorios

También se comprobó nuevamente `C:\ProgramData`:

```powershell
Get-ChildItem "C:\ProgramData" -Force |
Where-Object { $_.Name -like "*FileOverlord*" } |
Select-Object Name, CreationTime, LastWriteTime
```

Después de eliminar los directorios aislados durante la investigación, la consulta no devolvió resultados.

Esto permitió confirmar que ya no permanecían directorios `FileOverlord` dentro de:

```text
C:\ProgramData
```

---

## 9. Verificación combinada

Para simplificar las comprobaciones se utilizó posteriormente una consulta combinada:

```powershell
Write-Host "`n=== PROCESOS FILEOVERLORD ==="
Get-CimInstance Win32_Process |
Where-Object {
    $_.ExecutablePath -like "*FileOverlord*" -or
    $_.CommandLine -like "*FileOverlord*"
} |
Select-Object ProcessId, Name, ExecutablePath

Write-Host "`n=== CARPETAS FILEOVERLORD ==="
Get-ChildItem "C:\ProgramData" -Force |
Where-Object { $_.Name -like "*FileOverlord*" } |
Select-Object Name, CreationTime, LastWriteTime

Write-Host "`n=== SERVICIO ADSI IDENTIFICADO ==="
Get-Service ADSISvc_c3960c -ErrorAction SilentlyContinue
```

El resultado esperado después de una eliminación correcta era que las tres secciones permanecieran vacías:

```text
=== PROCESOS FILEOVERLORD ===

=== CARPETAS FILEOVERLORD ===

=== SERVICIO ADSI IDENTIFICADO ===
```

La ausencia de resultados debía interpretarse como:

* ningún proceso encontrado en rutas `FileOverlord`;
* ningún directorio `FileOverlord` encontrado en `C:\ProgramData`;
* servicio `ADSISvc_c3960c` no encontrado.

---

## 10. Importancia del reinicio

Una comprobación realizada inmediatamente después de eliminar un proceso o servicio no es suficiente para demostrar que la persistencia haya desaparecido.

Durante las primeras etapas de esta investigación ya se había observado que:

```text
finalizar print.exe
```

o:

```text
eliminar FileOverlord
```

podía producir una aparente solución temporal.

Sin embargo, el comportamiento reaparecía después de reiniciar Windows.

Por este motivo, el reinicio constituyó una prueba fundamental.

La comprobación debía realizarse nuevamente después de iniciar Windows para determinar si el sistema reconstruía:

```text
C:\ProgramData\FileOverlord-{GUID}
```

o volvía a ejecutar:

```text
print.exe
```

---

## 11. Comparación antes y después

El comportamiento observado puede resumirse de la siguiente manera.

### Antes de identificar el mecanismo

```text
Windows inicia
      │
      ▼
FileOverlord aparece
      │
      ▼
print.exe se ejecuta
      │
      ▼
"Utilidad de impresión"
      │
      ▼
Consumo aproximado de 2,3 GB de RAM
```

Eliminar únicamente `print.exe` o `FileOverlord` no impedía su reaparición.

### Después de la intervención

```text
ADSISvc_c3960c eliminado
      │
      ▼
Windows reinicia
      │
      ▼
No se detecta FileOverlord
      │
      ▼
No se detectan procesos desde FileOverlord
      │
      ▼
El comportamiento investigado no reaparece
```

Esta diferencia antes/después fue una de las evidencias más importantes para evaluar la efectividad de la contención.

---

## 12. Qué puede afirmarse

A partir de las observaciones realizadas durante esta investigación puede afirmarse que:

* `print.exe` se ejecutaba desde un directorio `FileOverlord` en `C:\ProgramData`;
* el directorio reaparecía después de haber sido eliminado o aislado;
* Process Monitor registró un `svchost.exe` realizando operaciones de creación y escritura sobre `FileOverlord`;
* el PID observado fue asociado con `ADSISvc_c3960c`;
* el servicio estaba configurado para inicio automático;
* después de deshabilitarlo, `FileOverlord` dejó de regenerarse durante las comprobaciones realizadas;
* posteriormente el servicio fue eliminado;
* los directorios residuales fueron eliminados;
* las comprobaciones posteriores no encontraron procesos ni directorios relacionados con `FileOverlord`;
* DISM completó correctamente su operación;
* SFC informó haber encontrado y reparado archivos dañados.

---

## 13. Qué no puede afirmarse

La evidencia recopilada **no permite afirmar de forma concluyente** que:

* `adsiedit.dll` fuera un archivo malicioso;
* todos los servicios con nombres similares a `ADSISvc` sean maliciosos;
* cualquier proceso llamado `print.exe` tenga relación con este incidente;
* cualquier directorio llamado `FileOverlord` corresponda al mismo comportamiento;
* los archivos reparados por SFC fueran dañados por `FileOverlord`;
* exista una relación causal entre otros hallazgos de seguridad encontrados posteriormente y el incidente documentado aquí.

Esta última consideración es especialmente importante.

Durante una revisión de seguridad pueden aparecer otros elementos que merezcan atención. Sin evidencia que los vincule técnicamente con `FileOverlord`, deben tratarse como **hallazgos independientes** y no incorporarse artificialmente a la cadena causal de este incidente.

---

## 14. Lección técnica principal

Uno de los aprendizajes más importantes de este caso fue que eliminar el proceso visible no necesariamente elimina la causa.

La investigación comenzó con:

```text
"Utilidad de impresión"
```

y:

```text
print.exe
```

pero ninguno de ellos explicó por sí solo por qué el comportamiento reaparecía.

La pregunta decisiva terminó siendo:

> **¿Quién vuelve a crear los archivos?**

Process Monitor permitió responder esa pregunta observando directamente las operaciones realizadas durante el arranque.

Esto llevó de:

```text
síntoma
```

a:

```text
proceso
```

después a:

```text
actividad del sistema
```

y finalmente a:

```text
mecanismo de persistencia
```

---

## 15. Estado final del incidente

Después de las medidas documentadas:

```text
FileOverlord
```

no volvió a ser detectado durante las comprobaciones realizadas.

El servicio:

```text
ADSISvc_c3960c
```

fue eliminado después de haber sido previamente identificado, deshabilitado, probado y respaldado.

No se eliminaron manualmente componentes legítimos de Windows como:

```text
svchost.exe
```

ni:

```text
C:\Windows\SysWOW64\adsiedit.dll
```

La estrategia utilizada fue eliminar únicamente el mecanismo cuya relación con el comportamiento había sido establecida mediante las observaciones realizadas durante la investigación.

---

## Conclusión

La verificación posterior fue tan importante como la eliminación.

Sin comprobar el comportamiento después de reiniciar Windows, una desaparición temporal de `print.exe` podría haberse interpretado erróneamente como una solución definitiva, tal como ocurrió durante las primeras etapas.

La combinación de:

```text
observación
→ identificación
→ contención
→ reinicio
→ verificación
```

permitió evaluar de manera mucho más fiable el resultado de la intervención.

---

[← Contención y eliminación](04-contencion-y-eliminacion.md) | [Volver al README](../README.md)
