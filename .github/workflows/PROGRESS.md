# Progreso del build Android — Blender 5.2

## Errores resueltos (en orden cronológico)
1. LFS pointers vacíos en startup.blend → fix: curl desde parent repo + fallback dummy
2. GCC insuficiente para host_tools (exige 14+) → fix: gcc-14/g++-14 + update-alternatives
3. host_tools sin deps de sistema (JPEG, PNG, Vulkan, etc.) → fix: apt install libs
4. shaderc.pc no disponible inicialmente → fix: descubierto que YA existe en x86_64 con -lshaderc
5. Python 3.13 no en PATH → fix: actions/setup-python@v5 con python-version: '3.13'
6. TEST_PYTHON_EXE eliminado al agregar flags → fix: restaurar -DTEST_PYTHON_EXE=$(which python3)
7. OpenImageIO REQUIRED sin instalar → fix: libopenimageio-dev
8. OpenImageIO::iconvert faltante → fix: openimageio-tools (provee /usr/bin/iconvert)
9. OpenColorIO 2.0.0 REQUIRED sin instalar → SIGUIENTE tras OIIO

## Estado actual
- Run ID: 34085879340
- Step donde cayó: Build and stage Gradle (host_tools configure)
- Esperando resultado de iconvert fix

## Arquitectura del problema
- CMake principal (Android): usa libs precompiladas lib-android_arm64 → OK
- host_tools (ExternalProject, host x86_64): busca deps del sistema
- platform_unix.cmake: OpenImageIO REQUIRED (495), OpenColorIO REQUIRED (501)

## Decisiones tomadas
- apt install para deps de sistema (lib-linux_x64 precompilado tenía LFS 404)
- actions/setup-python modifica PATH para steps posteriores
- openimageio-tools para iconvert bin (CMake targets reference it)

## NO HACER
- No reescribir código fuente de Blender sin autorización
- No asumir arquitectura sin verificar (sandbox ARM64 vs runner x86_64)
- No confundir CMake Warnings con CMake Errors bloqueantes

### 5. host_tools sub-build fails: OpenImageIO/OpenColorIO cmake configs broken (2026-09-07)

**Síntoma**: Configure CMake ✅, pero Build ❌ en `[32/7642] Performing configure step for 'host_tools'`. Errores:
- `Imported target "OpenImageIO::OpenImageIO" includes non-existent path`
- `Imported target "OpenColorIO::OpenColorIO" includes non-existent path`

**Causa raíz**: `host_tools` (makesdna/makesrna/datatoc/shader_tool) es un external project que ejecuta su propia configuración CMake heredando TODOS los flags `WITH_*` del build principal. Hereda `WITH_OPENIMAGEIO=ON` y `WITH_OPENCOLORIO=ON`. Los cmake configs del sistema (`/usr/lib/x86_64-linux-gnu/cmake/OpenImageIO/OpenImageIOConfig.cmake`) apuntan a `/usr/include/opencv4` (OpenCV, no instalado) y tienen paths rotos. El `CMAKE_PREFIX_PATH` del env solo afecta al configure principal, no a los sub-builds de CMake.

**Fix**: Tres flags a nivel del workflow, heredados por host_tools:
```yaml
-DWITH_OPENIMAGEIO=OFF
-DWITH_OPENCOLORIO=OFF
-DWITH_OPENIMAGEDENOISE=OFF
```

**Justificación**: host_tools NO necesita OIIO/OCIO para generar código (makesdna/makesrna/datatoc). Desactivarlas a nivel workflow evita que host_tools intente buscar system libs con cmake configs rotos, sin tocar código fuente de Blender. El Blender final sigue usando las libs precompiladas de Android normalmente vía `LIBDIR`.

**Si el build principal se queja** de que le faltan OIIO/OCIO para su compilación: ese caso ameritaría revisar en conjunto.

**Commits/workflow**: Solo `.github/workflows/build-android.yml` (android + main).


## F1 (en curso): swap backend SDL → GHOST NativeActivity de Wanderson

- **Estado**: 8 archivos GHOST de Wanderson copiados a `intern/ghost/intern/` (GHOST_AndroidMain.cc,
  GHOST_SystemAndroid.(c|h)h, GHOST_WindowAndroid.(c|h)h, GHOST_SystemPathsAndroid.(c|h)h,
  GHOST_AndroidMemoryTier.hh). Edits C++/CMake/gradle/Java/manifest aplicados (GShoWN arriba).
- **Cambios de la rama android** (commits F1):
  1. `CMakeLists.txt` (top): rama `ANDROID` → `WITH_GHOST_SDL=OFF` + `WITH_GHOST_ANDROID=ON` + `add_definitions`.
  2. `intern/ghost/CMakeLists.txt`: rama `elseif(WITH_GHOST_ANDROID)` con los 8 archivos, glue
     `android_native_app_glue.c` + `INC_SYS` desde `${ANDROID_NDK_ROOT}`, libs `android`+`log`.
  3. `source/creator/creator.cc`: `#ifdef WITH_GHOST_ANDROID` → `GHOST_android_launch(argc,argv)`
     (reemplaza `main`) con `WM_main_entry`→`GHOST_androidfinalize`; rama else sin cambios.
  4. `source/creator/CMakeLists.txt`: rama `ANDROID` → `add_library(blender SHARED ${SRC})` + `-Wl,-u,ANativeActivity_onCreate` (sin SDL, sin creator_android.cc).
  5. `source/blender/windowmanager/WM_api.hh` + `intern/wm.cc`: refactor `WM_main` →
     `WM_main_entry` + `WM_main_loop_body` (igual que Wanderson).
  6. `release/android/app/.../BlenderActivity.java`: `extends NativeActivity`, extrae
     `assets/extract/**` → `<filesDir>/blender/5.3` (stamp lastUpdateTime), IME bridge
     (nativeOnCommitText/nativeOnKey/nativeOpenMainFile), open-file resolve + BLENDER_ANDROID_OPEN_FILE,
     `openUrl`/`showKeyboard`/`hideKeyboard` (llamadas desde C++), python interpreter link +
     `PYTHONHOME`/`pyvenv.cfg`/`LD_LIBRARY_PATH`, hardware names, immersive + all-files access.
  7. `AndroidManifest.xml`: permiso `MANAGE_EXTERNAL_STORAGE` + `<meta-data android.app.lib_name=blender>`
     (NativeActivity). Se mantiene `launchMode="singleInstance"`, `configChanges` y orientation actuales.
  8. `build-android.yml`: `WITH_PYTHON_INSTALL` OFF→ON (alinea con rama main; la app crashea sin python en el runtime).
- **Contrato de rutas**: `GHOST_SystemPathsAndroid` lee `internalDataPath` (no envs). System dir =
  `<filesDir>/blender/5.3` (datafiles/scripts/python). getSystemLibsDir también apunta ahí.
- **Punto de control pendiente**: commit + push rama android; disparar build-android.yml; si compila,
  cerrar F1 y pasar a F2.

### BUG F1 — `undefined symbol: ANativeActivity_onCreate` (crash al arrancar en device)

- **Síntoma**: primer APK instalado en el device (moto g56 5G): `UnsatisfiedLinkError ... undefined
  symbol: ANativeActivity_onCreate` en `NativeActivity.onCreate` → `BlenderActivity` no llega a abrirse
  (el launcher vuelve al frente). `nm -D libblender.so` mostraba SOLO 12 exports (Python inits) y CERO
  símbolos del glue (`android_app`, `android_main`, `ANativeActivity_onCreate`).
- **Causa raíz REAL (la teoría del cache stale quedó FALSADA)**: el run `34307459241` SIN cache
  `blender/build` siguió produciendo el `.so` con SOLO 13 exports (12 PyInit + free/calloc). El
  culpable es el **version script de ocultación de símbolos**: el top-level `CMakeLists.txt` incluye
  `platform_unix.cmake` en Android (`if((UNIX AND NOT APPLE) OR ANDROID)`), que define
  `PLATFORM_LINKFLAGS_SYMBOL_HIDING = -Wl,--version-script='${PLATFORM_SYMBOLS_MAP}'`
  (`platform_unix.cmake:1088` · `symbols_unix.map` con `local: *`). `creator/CMakeLists.txt:2015`
  llama a `setup_platform_linker_symbol_hiding(blender)` (macros.cmake:597) y aplica el script al
  link → TODO queda oculto salvo las wildcards `global:` del map. Los 13 exports del `.so` casan al
  100% con esas wildcards (Py* / calloc* / free*): el glue (`ANativeActivity_onCreate`) no estaba en
  el map → no se exportaba. Wanderson NO padece esto porque su top-level usa
  `if(ANDROID) include(platform_android)` / `elseif(UNIX AND NOT APPLE) include(platform_unix)` y su
  `.so` release exporta las 176k symbols (sin version script en absoluto). El glue SÍ estaba linkeado
  en nuestro `.so` (el log mostró el .o compilado y link fresco `[7416/7660]`) — fallaba SOLO la
  exportación.
- **Fix F1** (commit `ae0de45e` rama android): `source/creator/symbols_unix.map` → añadir al bloque
  `global:` los entry points del glue NativeActivity:
  - `ANativeActivity_onCreate*` (lo busca el runtime vía dlopen al cargar libblender.so)
  - `android_main*` (el glue lo arranca desde ANativeActivity_onCreate)
  (mismo patrón que el `SDL_main*` que ya estaba en el map para la ruta SDLActivity). Con esto el
  ocultamiento de símbolos se mantiene para desktop y los símbolos del glue quedan exportados.
- **Lección**: en Android el version script `symbols_unix.map` (con `local: *`) se aplica igual que en
  desktop porque `platform_unix.cmake` se incluye para `ANDROID` también. Cualquier símbolo que el
  runtime de Android deba dlsym (entry points del glue) tiene que estar listado en el `global:` del
  map. La comprobación rápida: `nm -D libblender.so` y comparar contra las wildcards del map.
- **Verificado (run `34350750790`, release de `33404664373` + android code)**: `nm -D` del `.so`
  muestra `T ANativeActivity_onCreate` y `T android_main`. En device ya NO es `UnsatisfiedLinkError`;
  el glue corre (`main()` arranca, llega a WM init). F1 CERRADO → pasó a BUG F2.

### BUG F2 — `Unable to initialize GHOST, exiting!` (crash tras el permiso de almacenamiento)

- **Síntoma**: con F1 resuelto, la app pide permiso de almacenamiento y luego crashea:
  `blender: ghost.system | ERROR Unable to initialize GHOST, exiting!` a los 61ms del arranque, seguido
  de `FORTIFY: pthread_mutex_lock called on a destroyed mutex` y `signal 6 (Aborted)`.
- **Causa raíz**: `GHOST_ISystem::createSystem()` en la BASE (refactor multi-backend de Blender ≥4.x,
  `intern/ghost/intern/GHOST_ISystem.cc`) **no tenía rama ANDROID**: el `#if/#elif` solo contemplaba
  HEADLESS / X11+Wayland / X11 / Wayland / SDL / WIN32 / APPLE. El top-level `CMakeLists.txt`
  (`#if ANDROID`) fija `WITH_GHOST_SDL=OFF` + `WITH_GHOST_ANDROID=ON` (+ `-DWITH_GHOST_ANDROID`
  global), así que ninguna rama matcheaba → `system_` quedaba `nullptr` → `wm_window.cc:2299`
  imprimía el error y hacía `exit(EXIT_FAILURE)` antes de inicializar nada. En el árbol de Wanderson
  la rama SÍ existe:
  ```c
  #elif defined(WITH_GHOST_ANDROID)
      backends_attempted.push_back({"ANDROID"});
      CLOG_INFO(&LOG, "Create Android system");
      system_ = new GHOST_SystemAndroid();
  ```
- **Fix F2** (commit `…` rama android): añadir a `GHOST_ISystem.cc`:
  - include: `#elif defined(WITH_GHOST_ANDROID)` → `#include "GHOST_SystemAndroid.hh"`
  - createSystem: rama `#elif defined(WITH_GHOST_ANDROID)` → `system_ = new GHOST_SystemAndroid()`
  (idéntico al de Wanderson; la posición en la cadena es irrelevante porque los defines son mutuamente
  exclusivos).
- **Auditoría del resto del path Android** (diffs contra wanderson-audit): `GHOST_AndroidMain.cc`,
  `GHOST_SystemAndroid.cc/.hh`, `GHOST_WindowAndroid.cc` son **idénticos**. Las diferencias en
  `GHOST_ContextEGL.cc` y `creator.cc` son solo evolución del base (retry de config EGL alpha upstream,
  SYCL env, `BLI_task_scheduler_init(denormals)`) + prints `[BlenderAndroid]` cosméticos de Wanderson.
  Nada más bloqueante.
- **Verificación pendiente**: relanzar; en device GHOST debe arrancar y pasar a la inicialización de
  GPU/viewport. Próximo bug esperado: pipeline GLES/GPU backend (epoxy o draw).

