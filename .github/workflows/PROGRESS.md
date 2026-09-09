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
- **Causa raíz**: el cache `blender/build` (cache/restore+save en `build-android.yml`, key
  `blender-build-<libs-arm-commit>`) restauraba el árbol ninja con `build.ninja` y los CMakeLists con
  **mtimes iguales** (todos recién extraídos del tar) → la regla `RERUN_CMAKE` de ninja no disparaba
  reconfigure → el plan de link seguía siendo el pre-F1 (SDL, sin glue, sin `-Wl,-u`). El `.so`
  resultante no contenía el glue.
- **Fix** (dos partes, commits `33a67258` rama android + `73886ad7` rama main):
  1. `source/creator/CMakeLists.txt`: el glue `android_native_app_glue.c` se compila **directo** en el
     target `blender` (`target_sources` + `target_include_directories`), ya no vía la lib estática
     ghost; se mantiene `-Wl,-u,ANativeActivity_onCreate`. Object directo ⇒ link + export garantizados
     (no depende de scan de static lib ni de gc-sections). `intern/ghost/CMakeLists.txt` conserva solo
     el `INC_SYS` del glue (headers para GHOST_AndroidMain.cc).
  2. `build-android.yml`: **se elimina el cache `blender/build`** (restore + save). ccache se mantiene.
- **Lección**: no cachear árboles ninja/CMake vía `actions/cache` — la restauración con mtimes frescos
  rompe la regeneración de `build.ninja` y el build usa planes stale. ccache es la vía correcta de
  acelerar (keyed por contenido de fuente) sin el riesgo.
- **Verificación pendiente**: relanzar build (run `34307459241`), extraer el nuevo `gradle-stage`,
  `nm -D | grep ANativeActivity_onCreate` debe dar la función, re-instalar y probar en device.

