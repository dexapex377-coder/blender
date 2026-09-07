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

