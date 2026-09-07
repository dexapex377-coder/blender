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
