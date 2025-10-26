# Python bindings for aubio

py = import('python').find_installation('python3', required: true)

# Try to get numpy dependency
numpy_dep = dependency('numpy', required: false)

# Get numpy include directory
numpy_inc_result = run_command(py, 
  ['-c', 'import numpy; print(numpy.get_include())'],
  check: false
)

if numpy_inc_result.returncode() == 0
  numpy_inc = include_directories(numpy_inc_result.stdout().strip())
else
  error('NumPy is required for building Python bindings. Please install numpy: pip install numpy')
endif

# Generate Python extension sources
gen_external = files('lib/gen_external.py')

gen_sources = custom_target('generate_python_sources',
  output: ['aubio-generated.c', 'aubio-generated.h'],
  command: [
    py,
    '@SOURCE_DIR@' / 'lib' / 'gen_external.py',
    '@CURRENT_SOURCE_DIR@' / '..' / 'src',
    '@OUTPUT_DIR@',
  ],
  depend_files: gen_external,
)

# Python extension sources (static files only)
pyaubio_sources = files(
  'ext/aubiomodule.c',
  'ext/aubioproxy.c',
  'ext/py-cvec.c',
  'ext/py-fft.c',
  'ext/py-filter.c',
  'ext/py-filterbank.c',
  'ext/py-musicutils.c',
  'ext/py-phasevoc.c',
  'ext/py-sink.c',
  'ext/py-source.c',
  'ext/ufuncs.c',
)

# Add pre-generated wrapper sources (these exist in the source tree)
pyaubio_sources += files(
  'gen/gen-dct.c',
  'gen/gen-mfcc.c',
  'gen/gen-notes.c',
  'gen/gen-onset.c',
  'gen/gen-pitch.c',
  'gen/gen-pitchshift.c',
  'gen/gen-sampler.c',
  'gen/gen-specdesc.c',
  'gen/gen-tempo.c',
  'gen/gen-tss.c',
  'gen/gen-wavetable.c',
)

# Add the generated source from custom_target
pyaubio_sources += gen_sources

# Include directories for Python extension
pyaubio_inc = include_directories('ext', 'gen', '../src')

# Determine if we should build using installed libaubio or embed it
# For now, we'll link against the libaubio we just built
pyaubio_c_args = [
  '-DHAVE_CONFIG_H',
  '-DUSE_LOCAL_AUBIO',
  '-DAUBIO_VERSION="' + aubio_version + '"',
]

# Platform-specific args
if host_system == 'darwin'
  pyaubio_link_args = [
    '-framework', 'CoreFoundation',
    '-framework', 'AudioToolbox',
  ]
elif host_system == 'windows'
  # On Windows with MSVC, ignore missing debug Python library
  pyaubio_link_args = ['/NODEFAULTLIB:python312_d.lib']
else
  pyaubio_link_args = []
endif

# Build Python extension module
# On Windows, link against the static library since we don't export symbols from the DLL
if host_system == 'windows'
  pyaubio_link_with = libaubio_static
  pyaubio_deps = [py.dependency()]
  # Add aubio's dependencies directly since we're not using aubio_dep
  pyaubio_deps += dependencies
else
  pyaubio_link_with = []
  pyaubio_deps = [aubio_dep, py.dependency()]
endif

py.extension_module('_aubio',
  pyaubio_sources,
  include_directories: [pyaubio_inc, numpy_inc, config_inc],
  dependencies: pyaubio_deps,
  c_args: pyaubio_c_args,
  link_args: pyaubio_link_args,
  link_with: pyaubio_link_with,
  install: true,
  subdir: 'aubio',
)

# Install Python package files
py.install_sources(
  'lib/aubio/__init__.py',
  'lib/aubio/cmd.py',
  'lib/aubio/cut.py',
  'lib/aubio/midiconv.py',
  'lib/aubio/slicing.py',
  subdir: 'aubio',
  pure: false,
)
