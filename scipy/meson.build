# Platform detection
is_windows = host_machine.system() == 'windows'

if is_windows
  # For mingw-w64, link statically against the UCRT.
  gcc_link_args = ['-lucrt', '-static']
  if meson.get_compiler('c').get_id() == 'gcc'
    add_global_link_arguments(gcc_link_args, language: ['c', 'cpp'])
    # Force gcc to float64 long doubles for compatibility with MSVC
    # builds, for C only.
    add_global_arguments('-mlong-double-64', language: 'c')
    # Manual add of MS_WIN64 macro when not using MSVC.
    # https://bugs.python.org/issue28267
    bitness = run_command('_build_utils/gcc_build_bitness.py').stdout().strip()
    if bitness == '64'
      add_global_arguments('-DMS_WIN64', language: ['c', 'cpp', 'fortran'])
    endif
  endif
  if meson.get_compiler('fortran').get_id() == 'gcc'
    add_global_link_arguments(gcc_link_args, language: ['fortran'])
    # Flag needed to work around BLAS and LAPACK Gfortran dependence on
    # undocumented C feature when passing single character string
    # arguments.
    # https://gcc.gnu.org/bugzilla/show_bug.cgi?id=90329
    # https://github.com/wch/r-source/blob/838f9d5a7be08f2a8c08e47bcd28756f5d0aac90/src/gnuwin32/MkRules.rules#L121
    add_global_arguments('-fno-optimize-sibling-calls',
      language: ['fortran'])
  endif
endif

# NumPy include directory - needed in all submodules
incdir_numpy = run_command(py3,
  [
    '-c',
    'import os; os.chdir(".."); import numpy; print(numpy.get_include())'
  ],
  check: true
).stdout().strip()

inc_np = include_directories(incdir_numpy)

incdir_f2py = incdir_numpy / '..' / '..' / 'f2py' / 'src'
inc_f2py = include_directories(incdir_f2py)
fortranobject_c = incdir_f2py / 'fortranobject.c'

cc = meson.get_compiler('c')
npymath_path = incdir_numpy / '..' / 'lib'
npymath_lib = cc.find_library('npymath', dirs: npymath_path)
npyrandom_path = incdir_numpy / '..' / '..' / 'random' / 'lib'
npyrandom_lib = cc.find_library('npyrandom', dirs: npyrandom_path)

# pybind11 include directory - needed in several submodules
incdir_pybind11 = run_command(py3,
  [
    '-c',
    'import pybind11; print(pybind11.get_include())'
  ],
  check: true
).stdout().strip()

inc_pybind11 = include_directories(incdir_pybind11)

# Pythran include directory and build flags
use_pythran = run_command(py3,
  [
    '-c',
    'import os; print(os.environ.get("SCIPY_USE_PYTHRAN", 1))'
  ],
  check: true
).stdout().strip() == '1'

if use_pythran
  incdir_pythran = run_command(py3,
    [
      '-c',
      'import os; os.chdir(".."); import pythran; print(os.path.dirname(pythran.__file__));'
    ],
    check: true
  ).stdout().strip()
  inc_pythran = include_directories(incdir_pythran)
else
  inc_pythran = []
endif

cpp_args_pythran = [
  '-DENABLE_PYTHON_MODULE',
  '-D__PYTHRAN__=3',
  '-DPYTHRAN_BLAS_NONE'
]

# Don't use the deprecated NumPy C API. Define this to a fixed version instead of
# NPY_API_VERSION in order not to break compilation for released SciPy versions
# when NumPy introduces a new deprecation. Use in a meson.build file::
#
#   py3.extension_module('_name',
#     'source_fname',
#     numpy_nodepr_api)
#
numpy_nodepr_api = '-DNPY_NO_DEPRECATED_API=NPY_1_9_API_VERSION'


# TODO: 64-bit BLAS and LAPACK
blas = dependency('openblas')
lapack = dependency('openblas')

# TODO: deal with case where we do need the g77 ABI wrappers (see
#       _build_utils/_fortran.py)
g77_abi_wrappers = files('_build_utils' / 'src' / 'wrap_dummy_g77_abi.f')

generate_config = custom_target(
  'generate-config',
  install: true,
  build_always_stale: true,
  build_by_default: true,
  output: '__config__.py',
  input: '../tools/config_utils.py',
  command: [py3, '@INPUT@', '@OUTPUT@'],
  install_dir: py3.get_install_dir() + '/scipy'
)

#FIXME: the git revision is Unknown; script works when invoked directly, but
#       not when it's run by Ninja. See https://github.com/rgommers/scipy/pull/57
generate_version = custom_target(
  'generate-version',
  install: true,
  build_always_stale: true,
  build_by_default: true,
  output: 'version.py',
  input: '../tools/version_utils.py',
  command: [py3, '@INPUT@', '--source-root', '@SOURCE_ROOT@'],
  install_dir: py3.get_install_dir() + '/scipy'
)

python_sources = [
  '__init__.py',
  '_distributor_init.py',
  'conftest.py'
]

py3.install_sources(
  python_sources,
  pure: false,
  subdir: 'scipy'
)

py3.install_sources(
  ['_build_utils/tests/test_scipy_version.py'],
  pure: false,
  subdir: 'scipy/_lib/tests'
)

# Needed to trick Cython, it won't do a relative import outside a package
_cython_tree = custom_target('_cython_tree',
  output: [
    '__init__.py',
    'linalg.pxd',
    'special.pxd'
  ],
  input: [
    '__init__.py',
    'linalg.pxd',
    'special.pxd'
  ],
  command: [copier, '@INPUT@', '@OUTDIR@']
)
cython_tree = declare_dependency(sources: _cython_tree)

# Ordering of subdirs: special and linalg come first, because other submodules
# have dependencies on cython_special.pxd and cython_linalg.pxd. After those,
# subdirs with the most heavy builds should come first (that parallelizes
# better)
subdir('_lib')
subdir('special')
subdir('linalg')
subdir('sparse')
subdir('stats')
subdir('fft')
subdir('spatial')
subdir('cluster')
subdir('constants')
subdir('fftpack')
subdir('integrate')
subdir('signal')
subdir('interpolate')
subdir('ndimage')
subdir('odr')
subdir('optimize')
subdir('misc')
subdir('io')