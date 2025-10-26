project('aubio', 'c',
  version: '0.5.0-alpha',
  license: 'GPL-3.0-or-later',
  default_options: [
    'warning_level=2',
    'c_std=c99',
  ],
  meson_version: '>=0.58.0',
)

# Parse version from VERSION file
fs = import('fs')
version_file_path = meson.project_source_root() / 'VERSION'
version_contents = run_command(
  find_program('python3', 'python'),
  '-c',
  'import sys; print(open(sys.argv[1]).read())',
  version_file_path,
  check: true
).stdout().strip()

version_data = {}
foreach line : version_contents.split('\n')
  if line.contains('=')
    parts = line.split('=')
    key = parts[0]
    value = parts[1]
    if key == 'AUBIO_MAJOR_VERSION'
      version_data += {'major': value.to_int()}
    elif key == 'AUBIO_MINOR_VERSION'
      version_data += {'minor': value.to_int()}
    elif key == 'AUBIO_PATCH_VERSION'
      version_data += {'patch': value.to_int()}
    elif key == 'AUBIO_VERSION_STATUS'
      version_data += {'status': value.strip().strip('\'"')}
    elif key == 'LIBAUBIO_LT_CUR'
      version_data += {'lt_cur': value.to_int()}
    elif key == 'LIBAUBIO_LT_REV'
      version_data += {'lt_rev': value.to_int()}
    elif key == 'LIBAUBIO_LT_AGE'
      version_data += {'lt_age': value.to_int()}
    endif
  endif
endforeach

aubio_version = '@0@.@1@.@2@@3@'.format(
  version_data.get('major'),
  version_data.get('minor'),
  version_data.get('patch'),
  version_data.get('status', '')
)

# Library version (soversion)
lib_version = '@0@.@1@.@2@'.format(
  version_data.get('lt_cur'),
  version_data.get('lt_rev'),
  version_data.get('lt_age')
)

message('Building aubio version: ' + aubio_version)
message('Library version: ' + lib_version)

# Compiler setup
cc = meson.get_compiler('c')
host_system = host_machine.system()

# vcpkg integration for cross-platform dependency management
vcpkg_exe = find_program('vcpkg', required: false)
if vcpkg_exe.found()
  message('vcpkg found, installing dependencies from vcpkg.json manifest...')
  
  # Determine vcpkg triplet based on host system and architecture
  if host_system == 'windows'
    if host_machine.cpu_family() == 'x86_64'
      vcpkg_triplet = 'x64-windows'
    elif host_machine.cpu_family() == 'x86'
      vcpkg_triplet = 'x86-windows'
    elif host_machine.cpu_family() == 'aarch64'
      vcpkg_triplet = 'arm64-windows'
    else
      vcpkg_triplet = 'x64-windows'
    endif
  elif host_system == 'darwin'
    if host_machine.cpu_family() == 'aarch64'
      vcpkg_triplet = 'arm64-osx'
    else
      vcpkg_triplet = 'x64-osx'
    endif
  elif host_system == 'linux'
    if host_machine.cpu_family() == 'aarch64'
      vcpkg_triplet = 'arm64-linux'
    else
      vcpkg_triplet = 'x64-linux'
    endif
  endif
  
  message('Using vcpkg triplet: ' + vcpkg_triplet)
  
  # Install vcpkg dependencies
  vcpkg_result = run_command(
    vcpkg_exe,
    'install',
    '--triplet=' + vcpkg_triplet,
    '--x-wait-for-lock',
    '--x-manifest-root=' + meson.project_source_root(),
    '--x-install-root=' + meson.project_source_root() / 'vcpkg_installed',
    check: false
  )
  
  if vcpkg_result.returncode() == 0
    message('vcpkg dependencies installed successfully')
    
    # Set up pkg-config and cmake paths for vcpkg
    vcpkg_installed_dir = meson.project_source_root() / 'vcpkg_installed' / vcpkg_triplet
    
    # Add vcpkg paths to environment
    add_project_arguments(
      '-I' + vcpkg_installed_dir / 'include',
      language: 'c'
    )
    add_project_link_arguments(
      '-L' + vcpkg_installed_dir / 'lib',
      language: 'c'
    )
    
    # Set PKG_CONFIG_PATH for dependency discovery
    pkg_config_env = environment()
    pkg_config_env.prepend('PKG_CONFIG_PATH', vcpkg_installed_dir / 'lib' / 'pkgconfig')
    pkg_config_env.prepend('PKG_CONFIG_PATH', vcpkg_installed_dir / 'share' / 'pkgconfig')
    
    # Store vcpkg paths for later use
    meson.add_devenv(pkg_config_env)
  else
    warning('vcpkg install failed, falling back to system dependencies')
    message(vcpkg_result.stderr())
  endif
else
  message('vcpkg not found, using system dependencies')
endif

# Configuration data
conf_data = configuration_data()
conf_data.set('HAVE_CONFIG_H', 1)
conf_data.set_quoted('AUBIO_PREFIX', get_option('prefix'))
conf_data.set_quoted('PACKAGE', 'aubio')

# Build type
if get_option('buildtype') == 'debug'
  conf_data.set('DEBUG', 1)
else
  conf_data.set('NDEBUG', 1)
endif

# Check for standard headers
check_headers = [
  'stdlib.h',
  'stdio.h',
  'math.h',
  'string.h',
  'errno.h',
  'limits.h',
  'stdarg.h',
  'getopt.h',
  'unistd.h',
]

foreach h : check_headers
  if cc.has_header(h)
    conf_data.set('HAVE_' + h.to_upper().underscorify(), 1)
  endif
endforeach

# Platform-specific settings
if host_system == 'windows'
  conf_data.set('HAVE_WIN_HACKS', 1)
endif

# Check for math library
math_dep = cc.find_library('m', required: false)

# Double precision mode
enable_double = get_option('double')
if enable_double
  conf_data.set('HAVE_AUBIO_DOUBLE', 1)
  message('Building in double precision mode')
else
  message('Building in single precision mode')
endif

# Complex number support
if get_option('complex')
  if cc.has_header('complex.h')
    conf_data.set('HAVE_COMPLEX_H', 1)
  endif
endif

# Check for C99 __VA_ARGS__ macros
va_args_code = '''
#include <stdio.h>
#define AUBIO_ERR(...) fprintf(stderr, __VA_ARGS__)
int main(void) { return 0; }
'''
if cc.compiles(va_args_code, name: 'C99 __VA_ARGS__ macro')
  conf_data.set('HAVE_C99_VARARGS_MACROS', 1)
endif

# Memcpy hacks
if get_option('memcpy')
  conf_data.set('HAVE_MEMCPY_HACKS', 1)
endif

# Wavread/wavwrite support
if get_option('wavread')
  conf_data.set('HAVE_WAVREAD', 1)
endif
if get_option('wavwrite')
  conf_data.set('HAVE_WAVWRITE', 1)
endif

# Dependencies
dependencies = [math_dep]

# FFT implementation
fftw3_dep = dependency('', required: false)
fftw3f_dep = dependency('', required: false)

if get_option('fftw3f').enabled()
  fftw3f_dep = dependency('fftw3f', version: '>=3.0.0', required: get_option('fftw3f'))
  if fftw3f_dep.found()
    conf_data.set('HAVE_FFTW3F', 1)
    conf_data.set('HAVE_FFTW3', 1)
    dependencies += fftw3f_dep
    message('FFT implementation: fftw3f')
  endif
elif get_option('fftw3').enabled()
  fftw3_dep = dependency('fftw3', version: '>=3.0.0', required: get_option('fftw3'))
  if fftw3_dep.found()
    conf_data.set('HAVE_FFTW3', 1)
    dependencies += fftw3_dep
    message('FFT implementation: fftw3')
  endif
endif

# Intel IPP
if get_option('intelipp').enabled()
  ipp_headers = ['ippcore.h', 'ippvm.h', 'ipps.h']
  has_ipp_headers = true
  foreach h : ipp_headers
    if not cc.has_header(h)
      has_ipp_headers = false
    endif
  endforeach
  
  if has_ipp_headers
    ipp_dep = cc.find_library('ippcore', required: false)
    ippvm_dep = cc.find_library('ippvm', required: false)
    ipps_dep = cc.find_library('ipps', required: false)
    
    if ipp_dep.found() and ippvm_dep.found() and ipps_dep.found()
      conf_data.set('HAVE_INTEL_IPP', 1)
      dependencies += [ipp_dep, ippvm_dep, ipps_dep]
      message('Intel IPP support: enabled')
      
      if cc.get_id() == 'msvc'
        conf_data.set('_IPP_SEQUENTIAL_STATIC', 1)
      endif
    endif
  endif
endif

# Accelerate framework (macOS)
accelerate_dep = dependency('', required: false)
if host_system == 'darwin' and get_option('accelerate').auto()
  accelerate_dep = dependency('Accelerate', required: false)
  if accelerate_dep.found()
    conf_data.set('HAVE_ACCELERATE', 1)
    dependencies += accelerate_dep
    message('Accelerate framework: enabled')
  endif
endif

# Apple Audio (macOS/iOS)
apple_audio_deps = []
if host_system == 'darwin' and get_option('apple-audio').auto()
  corefoundation_dep = dependency('CoreFoundation', required: false)
  audiotoolbox_dep = dependency('AudioToolbox', required: false)
  
  if corefoundation_dep.found() and audiotoolbox_dep.found()
    conf_data.set('HAVE_SOURCE_APPLE_AUDIO', 1)
    conf_data.set('HAVE_SINK_APPLE_AUDIO', 1)
    apple_audio_deps = [corefoundation_dep, audiotoolbox_dep]
    dependencies += apple_audio_deps
    message('Apple Audio support: enabled')
  endif
endif

# libsndfile
sndfile_dep = dependency('sndfile', version: '>=1.0.4', required: get_option('sndfile'))
if sndfile_dep.found()
  conf_data.set('HAVE_SNDFILE', 1)
  dependencies += sndfile_dep
endif

# libsamplerate (not compatible with double precision)
samplerate_dep = dependency('', required: false)
if not enable_double
  samplerate_dep = dependency('samplerate', version: '>=0.0.15', required: get_option('samplerate'))
  if samplerate_dep.found()
    conf_data.set('HAVE_SAMPLERATE', 1)
    dependencies += samplerate_dep
  endif
else
  if get_option('samplerate').enabled()
    error('libsamplerate is not compatible with double precision mode')
  endif
endif

# rubberband
rubberband_dep = dependency('rubberband', version: '>=1.3', required: get_option('rubberband'))
if rubberband_dep.found()
  conf_data.set('HAVE_RUBBERBAND', 1)
  dependencies += rubberband_dep
endif

# JACK
jack_dep = dependency('jack', required: get_option('jack'))
if jack_dep.found()
  conf_data.set('HAVE_JACK', 1)
  dependencies += jack_dep
endif

# libav (avcodec, avformat, avutil, swresample)
avcodec_dep = dependency('libavcodec', version: '>=54.35.0', required: get_option('avcodec'))
avformat_dep = dependency('libavformat', version: '>=52.3.0', required: get_option('avcodec'))
avutil_dep = dependency('libavutil', version: '>=52.3.0', required: get_option('avcodec'))
swresample_dep = dependency('libswresample', version: '>=1.2.0', required: false)

if avcodec_dep.found() and avformat_dep.found() and avutil_dep.found() and swresample_dep.found()
  conf_data.set('HAVE_LIBAV', 1)
  conf_data.set('HAVE_AVCODEC', 1)
  conf_data.set('HAVE_AVFORMAT', 1)
  conf_data.set('HAVE_AVUTIL', 1)
  conf_data.set('HAVE_SWRESAMPLE', 1)
  dependencies += [avcodec_dep, avformat_dep, avutil_dep, swresample_dep]
endif

# vorbis
vorbisenc_dep = dependency('vorbisenc', required: get_option('vorbis'))
vorbis_dep = dependency('vorbis', required: get_option('vorbis'))
ogg_dep = dependency('ogg', required: get_option('vorbis'))
if vorbisenc_dep.found() and vorbis_dep.found() and ogg_dep.found()
  conf_data.set('HAVE_VORBISENC', 1)
  dependencies += [vorbisenc_dep, vorbis_dep, ogg_dep]
endif

# FLAC
flac_dep = dependency('flac', required: get_option('flac'))
if flac_dep.found()
  conf_data.set('HAVE_FLAC', 1)
  dependencies += flac_dep
endif

# BLAS
blas_dep = dependency('blas', required: get_option('blas'))
if blas_dep.found()
  conf_data.set('HAVE_BLAS', 1)
  dependencies += blas_dep
  
  # Check for BLAS headers
  blas_headers = ['cblas.h', 'atlas/cblas.h', 'openblas/cblas.h']
  foreach h : blas_headers
    if cc.has_header(h)
      break
    endif
  endforeach
endif

# Generate config.h
configure_file(
  output: 'config.h',
  configuration: conf_data,
  install: false,
)

# Add configuration include
config_inc = include_directories('.')

# Subdirectories
subdir('src')

if get_option('examples')
  subdir('examples')
endif

if get_option('tests')
  subdir('tests')
endif

# Python bindings
if get_option('python')
  subdir('python')
endif

# pkg-config file
pkg = import('pkgconfig')
pkg.generate(
  libaubio,
  name: 'aubio',
  description: 'a library for audio labelling',
  version: aubio_version,
  filebase: 'aubio',
  subdirs: 'aubio',
)
