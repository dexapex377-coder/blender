# Progreso del build Android — Blender 5.2

## Errores resueltos (en orden cronológico)
1. LFS pointers vacíos en startup.blend → fix: curl desde parent repo + fallback dummy si 404/tamaño insuficiente
2. GCC 13 insuficiente para host_tools (exige 14+) → fix: instalar gcc-14/g++-14 + update-alternatives
3. host_tools sin deps de sistema (JPEG, PNG, Vulkan, etc.) → fix: apt install libs necesarias
4. libshaderc-dev no proveía shaderc.pc correcto → fix: shaderc_combined.pc links a -lshaderc_combined (la shared no existía), SE HALLA que en x86_64 YA existe shaderc.pc con -lshaderc (OK)
5. Python 3.13 no en PATH para host_tools → fix: actions/setup-python@v5 con python-version: '3.13'
6. TEST_PYTHON_EXE eliminado al agregar flags Python → fix: restaurar -DTEST_PYTHON_EXE=$(which python3)
7. OpenImageIO REQUIRED en host_tools sin instalar → EN PROGRESO
8. OpenColorIO 2.0.0 REQUIRED en host_tools sin instalar → SIGUIENTE tras OIIO

## Estado actual
- Run ID: 34084016456 (Configure ✅, host_tools build ❌ en OpenImageIO)
- Step donde falla: Build and stage Gradle → host_tools configure
- Error real: `find_package_wrapper(OpenImageIO REQUIRED)` — CMake Error (no warning)
- Deps ya resueltas: shaderc ✅, Python 3.13 ✅, Vulkan ✅

## Arquitectura del problema
- El CMake principal (Android) usa libs precompiladas de lib-android_arm64 → OK
- host_tools (ExternalProject que corre en host x86_64) busca deps del sistema → FALTA
- platform_unix.cmake línea 495: OpenImageIO REQUIRED
- platform_unix.cmake línea 501: OpenColorIO 2.0.0 REQUIRED
- warnings "disabling" no bloquean (OpenJPEG, OpenAL, FFmpeg, etc.)

## NO HACER (descartados, no repetir)
- No reescribir código fuente de Blender (build_files/, source/) sin autorización
- No asumir arquitectura del runner sin verificar (sandbox era ARM64, runner real x86_64)
- No confundir warnings con errores bloqueantes
- No eliminar TEST_PYTHON_EXE al modificar flags
- No asumir que un repo de libs precompiladas tiene LFS funcional sin verificar

## Decisiones tomadas
- Se usa apt install de deps de sistema para host_tools (lib-linux_x64 precompilado tenía LFS 404)
- Se mantiene `-DWITH_OPENCOLORIO=OFF` para CMake principal (libs precompiladas); host_tools usa defaults propios
- actions/setup-python modifica PATH solo para steps posteriores
