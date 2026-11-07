# 🖥️ HDMIB — HDMI en DDI B para Intel Ice Lake en macOS

[![Release Candidate]([suspicious link removed])](#-estado-del-proyecto)
[![macOS Support](https://img.shields.io/badge/macOS-15_%7C_26-blue.svg)](#-versiones-compatibles)
[![License](https://img.shields.io/badge/license-BSD--3--Clause-green.svg)](LICENSE)

Módulo complementario (header-only) para [WhateverGreen](https://github.com/acidanthera/WhateverGreen) que soluciona el problema de los puertos HDMI inactivos en portátiles con procesadores **Intel Ice Lake** (Gen 11).

---

## 🧐 ¿Qué problema resuelve este módulo?

El controlador de framebuffer de Apple para Ice Lake (`AppleIntelICLLPGraphicsFramebuffer`) únicamente reconoce receptores **DisplayPort** en el conector DDI B. La detección original de monitores depende de lecturas DPCD a través del canal auxiliar (AUX), una característica ausente en conexiones HDMI nativas. 

Por ello, en portátiles donde la salida HDMI está conectada a DDI B, el sistema genera en los registros el error:
```text
Hotplug detected on unsupported ddi 1
```
dejando el puerto de video completamente muerto.

### ✨ Solución implementada
Este módulo integra en tiempo de ejecución las funciones necesarias para operar como HDMI/TMDS:
* 🔌 **Detección basada únicamente en HPD** (Hot-Plug Detect), sin requerir canal AUX.
* 📖 **Lector de EDID propio** para identificar pantallas externas correctamente.
* 🛠️ **Inicialización de rutas de pantalla y recursos** que Apple omite en framebuffers inyectados.
* ⚡ **Alimentación automática de DBUF slice-2** para la salida pipe B.
* 🖱️ **Cursor por hardware real** en la pantalla secundaria conectada por HDMI.
* 🛡️ **Parches de estabilidad**: previenen bloqueos o kernel panics comunes por hot-plugging (doble activación de DC6, tiempos de espera continuos en AUX y fallos de VRR).

---

## 🚦 Estado del proyecto

> ⚠️ **Estado: Release Candidate**.  
> Este módulo fue desarrollado y se utiliza a diario en un portátil **ASUS X415JA** (Intel Core i5-1035G1 / Iris Plus G1). Funciona de manera estable en este equipo, pero **no ha sido probado en otros modelos**.  
> Dado que parchea estructuras internas no documentadas del controlador de Apple, cualquier desviación en los offsets puede provocar un kernel panic; por seguridad, el módulo **permanece deliberadamente inactivo** en versiones de macOS no verificadas.

### 🚫 Limitaciones actuales
* 🔇 **Audio por HDMI**: aún no implementado.
* 💻 **Exclusivo para Ice Lake LP**: compatible con `AppleIntelICLLPGraphicsFramebuffer` (i3/i5/i7-10x5G*, Gen 11). No compatible con Tiger Lake, Comet Lake ni ICL-HP.
* 🖥️ **Un solo puerto HDMI activo**: diseñado para DDI B de forma predeterminada (aunque `igfxhdmib-ddi=` permite elegir otro DDI, solo DDI B ha sido probado).
* 📐 **Resoluciones**: probado únicamente con monitor 1360x768 (mediante adaptador VGA→HDMI). No se han realizado pruebas con frecuencias de reloj elevadas o pantallas 4K.

---

## 🍏 Versiones compatibles

| macOS | Darwin | Estado del Layout | Notas |
| :--- | :---: | :---: | :--- |
| **macOS 26 Tahoe** | 25 | **Verificado** ✅ | Versión base sobre la que se desarrolló el módulo. |
| **macOS 15 Sequoia** | 24 | **Verificado** ✅ | Verificado en 15.8.2 (24H232) mediante análisis estático y pruebas en hardware. |
| *Cualquier otra versión* | — | No verificado ⏸️ | El módulo se mantendrá inactivo por seguridad. |

---

## 📦 Instalación paso a paso

### 1️⃣ Compilación
> ⚠️ **Requisito:** Se requiere la suite completa de **Xcode** (no basta con las Command Line Tools).

Ejecuta el script de construcción automatizado:
```bash
bash build_weg_hdmib.sh
```
El script descarga las fuentes de Lilu, MacKernelSDK y WhateverGreen, integra el módulo con los ajustes requeridos en `kern_igfx.cpp` y compila el binario resultante `WhateverGreen-HDMIB.kext`.

> 📌 **Importante:** Utiliza este kext en sustitución del WhateverGreen oficial (**nunca instales ambos simultáneamente**). Para fijar versiones específicas puedes usar `WEG_REF=<commit>` y `LILU_REF=<commit>`.

---

### 2️⃣ Argumentos de arranque (Boot-args)
Agrega el argumento principal en tu `config.plist` de OpenCore (`NVRAM -> Add -> 7C436110-AB2A-4BBB-A880-FE41995C9F82 -> boot-args`):

```text
-igfxhdmib
```
*Sin este argumento, el módulo permanece completamente inactivo, por lo que el kext es seguro de conservar instalado.*

---

### 3️⃣ Parche de DeviceProperties en la iGPU
En la ruta PCI de tu gráfica integrada (`PciRoot(0x0)/Pci(0x2,0x0)`), debes reescribir un conector no utilizado de la tabla de la plataforma para convertirlo en HDMI en DDI B con pin GMBUS 2.

Ejemplo aplicado en la ASUS X415JA (`AAPL,ig-platform-id`: `01005c8a`), donde se reemplaza la 4.ª entrada (DDI D):

| Propiedad | Tipo | Valor |
| :--- | :--- | :--- |
| `framebuffer-patch-enable` | Data | `01000000` |
| `framebuffer-patch0-enable` | Data | `01000000` |
| `framebuffer-patch0-find` | Data | `03000000 0a000000 01000000 01000000 00040000 c1020000` |
| `framebuffer-patch0-replace` | Data | `01000000 02000000 02000000 00000000 00080000 c1020000` |

> 💡 **Nota:** La secuencia de bytes en `find` debe coincidir con la tabla de conectores de tu plataforma específica. Puedes examinarla previamente con Hackintool (*Patch → Connectors*) o usando una versión DEBUG de WhateverGreen con el boot-arg `-igfxfbdump`.

---

### 4️⃣ Reiniciar
Reinicia el equipo. Al conectar el cable HDMI, la pantalla externa debería encenderse inmediatamente.

---

## ⚙️ Configuración de referencia (ASUS X415JA)

Datos del equipo de pruebas: **Intel Core i5-1035G1** (device-id reportado `0x8A56`, spoofeado a `0x8A5C`), DVMT preasignado en BIOS de 64MB, OpenCore 1.0.8 en macOS 15/26.

Las propiedades completas de la iGPU se encuentran en el archivo [`example_iGPU_DeviceProperties.plist`](example_iGPU_DeviceProperties.plist):

| Clave | Valor | Propósito |
| :--- | :--- | :--- |
| `framebuffer-patch-enable` / `framebuffer-patch0-enable` | `01000000` | Activa el motor de parches de conectores. |
| `framebuffer-patch0-find` / `-replace` | *Ver paso 3* | Crea el conector HDMI (DDI B, pin GMBUS 2, clase 2, tipo 0x800). |
| `AAPL,ig-platform-id` | `01005c8a` | ID de plataforma del procesador. |
| `device-id` | `5c8a0000` | Identificador spoofeado para habilitar aceleración gráfica completa. |
| `framebuffer-stolenmem` / `-fbmem` / `-unifiedmem` | `00003001` / `00009000` / `000000c0` | Ajustes de memoria asignada dependientes del DVMT del BIOS. |
| `enable-dbuf-early-optimizer` | `01000000` | Corrige la pantalla negra en la ventana de inicio de sesión de este equipo. |

---

## 🎛️ Tabla de Boot-args

| Boot-arg | Valor por defecto | Descripción |
| :--- | :---: | :--- |
| `-igfxhdmib` | *Desactivado* | Activa las funciones del módulo. |
| `igfxhdmib-ddi=N` | `1` | Define el índice DDI del puerto HDMI (`1` = DDI B). |
| `igfxhdmib-fb=N` | `auto` | Índice del framebuffer asignado al puerto. |
| `igfxhdmib-bootdetect=S` | `3` | Tiempo de espera en segundos para reintentar hot-plug si el cable ya estaba conectado al arrancar (`0` = apagado). |
| `igfxhdmib-hwcursor=0\|1` | `1` | Cursor por hardware en la pantalla HDMI (`0` = cursor por software). |
| `igfxhdmib-swcursor=1` | `0` | Fuerza cursor por software en todos los framebuffers. |
| `igfxhdmib-aux=0` | `1` | Evita interceptar las lecturas AUX que el acelerador intenta emitir. |
| `igfxhdmib-dc6fix=0` | `1` | No omite el segundo `enableHWDC6()` (evita el pánico *DC6 is already enabled*). |
| `igfxhdmib-edid-fallback=1` | `0` | Si la lectura del EDID falla, inyecta un EDID genérico 1024x768@60 en su lugar. |
| `igfxhdmib-force=1` | `0` | Fuerza el funcionamiento en versiones de macOS no verificadas (**Bajo tu propio riesgo**). |

---

## 🔍 Diagnóstico y comprobación

Para verificar si el módulo está cargado y funcionando, ejecuta en la Terminal:

```bash
ioreg -l | grep HDMIB
```

### Interpretación de resultados
* **`HDMIB-layout = "unverified: idle"`**: la versión actual de macOS no está en la tabla de layouts verificados; el módulo no actúa.
* **`HDMIB-hooks = "symbols-missing: idle"`**: el kext del sistema carece de funciones requeridas por el módulo.
* **`HDMIB-hooks = "core-route-failed"`**: Lilu no pudo interceptar los hooks esenciales; el puerto no funcionará.
* **`HDMIB-hooks = "installed"`**: los ganchos se instalaron correctamente en memoria.
* **`HDMIB-status = "hotplugs=1 edid-ok=1 ..."`**: la conexión se detectó y el EDID se leyó sin errores. *(Si `edid-fail > 0`, revisa el cable/adaptador o prueba con `igfxhdmib-edid-fallback=1`)*.

---

## ⚠️ Problemas conocidos y consejos

* 🔄 **Conexión recomendada**: Conectar el cable HDMI **después de iniciar sesión** es el escenario con mayor margen de estabilidad. Si arrancas con el cable conectado, asegúrate de mantener activo `igfxhdmib-bootdetect`.
* 🖥️ **Resolución nativa del panel interno**: Si la pantalla integrada no se encuentra en su resolución nativa al enchufar el HDMI, la pantalla externa puede permanecer en negro. Vuelve a poner la pantalla integrada en su resolución nativa y reconecta el cable.
* 🪵 **Avisos inocuos en logs**: Es normal observar mensajes de tipo `VRR_ERR` o fallos aislados en escrituras `kHDCP1_DPCD_AN` en el registro de sistema; no provocan ningún impacto visual perceptible.

---

## 🧪 Lista de verificación para pruebas

Al probar una nueva máquina o versión de macOS, sigue esta secuencia de validación:
1. Iniciar con `-igfxhdmib` sin cable conectado: verificar `HDMIB-hooks = installed`, sin panics y con la pantalla interna funcionando normalmente.
2. Conectar el cable tras iniciar sesión: la pantalla debe encenderse en pocos segundos y `HDMIB-status` debe mostrar lecturas de EDID exitosas.
3. Conectar y desconectar 5 veces seguidas para comprobar que no existan bloqueos ni congelamientos.
4. Iniciar con el cable conectado desde el encendido: la pantalla debe activarse pocos segundos después de cargar el escritorio.
5. Probar ciclos de reposo y reactivación (Sleep/Wake) con el cable conectado y desconectándolo durante el reposo.
6. Mover el cursor entre ambas pantallas sobre transparencias y con salvapantallas para descartar fallos de cursor duplicado o sobredimensionado.

---

## 🧭 Adaptación a nuevas versiones de macOS

Para verificar la compatibilidad de una nueva versión de macOS, puedes ejecutar el script automatizado:
```bash
bash verify_layout.sh
```
El script extrae el framebuffer de la colección de kernels, analiza los desplazamientos en memoria con ayuda de `capstone` y genera un informe de diferencias en tu Escritorio (`~/Desktop/hdmib_verify.txt`) frente a las referencias base.

---

## 📂 Estructura del repositorio

* `kern_igfx_hdmib.hpp`: Código fuente del módulo.
* `example_iGPU_DeviceProperties.plist`: Ejemplo funcional de DeviceProperties para OpenCore.
* `build_weg_hdmib.sh`: Script de descarga y compilación automática con WhateverGreen.
* `verify_layout.sh`: Script de verificación rápida de layouts en el sistema local.
* `tools/`: Utilidades de extracción y huellas de memoria (`kcx.pl`, `layout_fingerprint.py`, esquemas de referencia).
* `LICENSE`: Licencia BSD de 3 cláusulas.

---

## ⚖️ Descargo de responsabilidad
Este proyecto no está respaldado ni afiliado con Apple, Intel ni Acidanthera. El software se distribuye tal como está, bajo tu propia responsabilidad.
