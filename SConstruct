"""
SCons build script for Ember

Basic usage:

    'scons help' - print a description of user-specifiable options

    'scons build' - Compile Ember

    'scons test' - Run the test suite

    'scons install' - Install the Ember Python module

    'scons msi' - Create a MSI installer for Windows
"""

from __future__ import print_function

import os
import platform
from distutils.sysconfig import get_config_var
from distutils.version import StrictVersion
from Cython.Build import cythonize
import numpy as np
from buildutils import *

if not COMMAND_LINE_TARGETS:
    print(__doc__)
    sys.exit(0)

VariantDir('build/core','src', duplicate=0)
VariantDir('build/test','test', duplicate=0)

extraEnvArgs = {}
opts = Variables('ember.conf')
class defaults: pass

defaults.env_vars = 'PYTHONPATH,CANTERA_DATA,PATH'

if os.name == 'nt':
    windows_compiler_options = []
    # On Windows, target the same architecture as the current copy of Python,
    # unless the user specified another option.
    if '64 bit' in sys.version:
        target_arch = 'amd64'
    else:
        target_arch = 'x86'

    # Make an educated guess about the right default compiler
    if which('g++') and not which('cl.exe'):
        defaultToolchain = 'mingw'
    else:
        defaultToolchain = 'msvc'

    windows_compiler_options.extend([
        ('msvc_version',
         """Version of Visual Studio to use. The default is the newest
            installed version. Specify '9.0' for VS2008, '10.0' for VS2010,
            '11.0' for VS2012, '12.0' for VS2013, etc. This must be the same
            version of Visual Studio used to compile Cantera.""",
         ''),
        ('target_arch',
         """Target architecture. The default is the same
            architecture as the installed version of Python. Should be one of
            'amd64' or 'x86'.""",
         target_arch)])
    opts.AddVariables(*windows_compiler_options)

    pickCompilerEnv = Environment()
    opts.Update(pickCompilerEnv)

    if pickCompilerEnv['msvc_version']:
        defaultToolchain = 'msvc'

    windows_compiler_options.append(EnumVariable(
        'toolchain',
        """The preferred compiler toolchain. This must be the same toolchain
           used to compile Cantera.""",
        defaultToolchain, ('msvc', 'mingw', 'intel')))
    opts.AddVariables(windows_compiler_options[-1])
    opts.Update(pickCompilerEnv)

    if pickCompilerEnv['toolchain'] == 'msvc':
        toolchain = ['default']
        msvc = pickCompilerEnv['msvc_version'] or pickCompilerEnv['MSVC_VERSION']
        extraEnvArgs['MSVC_VERSION'] = msvc
        tbbCompiler = 'vc' + msvc.split('.')[0] # e.g. vc9
        print('INFO: Compiling with MSVC', msvc)

    elif pickCompilerEnv['toolchain'] == 'mingw':
        toolchain = ['mingw', 'f90']
        extraEnvArgs['F77'] = None
        # Next line fixes http://scons.tigris.org/issues/show_bug.cgi?id=2683
        extraEnvArgs['WINDOWS_INSERT_DEF'] = 1

    elif pickCompilerEnv['toolchain'] == 'intel':
        toolchain = ['intelc'] # note: untested

    target_arch = pickCompilerEnv['target_arch']
    extraEnvArgs['TARGET_ARCH'] = target_arch
    if target_arch == 'amd64':
        tbbArch = 'intel64'
    elif target_arch == 'x86':
        tbbArch = 'ia32'

    defaults.blas_lapack = ''
    print('INFO: Compiling for architecture:', pickCompilerEnv['target_arch'])
    print('INFO: Compiling using the following toolchain(s):', repr(toolchain))

elif platform.system() == 'Darwin':
    defaults.blas_lapack = ''
    defaults.env_vars += ',DYLD_LIBRARY_PATH'
else:
    defaults.env_vars += ',LD_LIBRARY_PATH'
    defaults.blas_lapack = 'lapack,blas'

env = Environment(tools=['default', 'subst'], **extraEnvArgs)

opts.AddVariables(
    ('CXX',
     'The C++ compiler to use.',
     env['CXX']),
    PathVariable(
        'cantera',
        'Location of the Cantera header and library files.',
        '', PathVariable.PathAccept),
    ('cantera_libs',
     """Comma-separated list of extra libraries required to link with
        Cantera. If Cantera was not built with the "single_library=y" option,
        this may include some of 'execstream', 'ctmath', 'ctlapack', 'ctblas',
        and 'ctf2c'""",
     ''),
    PathVariable(
        'sundials',
        'Location of the Sundials header and library files.',
        '', PathVariable.PathAccept),
    PathVariable(
        'eigen',
        'Location of the Eigen header files',
        '', PathVariable.PathAccept),
    PathVariable(
        'boost',
        'Location of the Boost header files.',
        '', PathVariable.PathAccept),
    ('include',
     'Comma-separated List of additional include directories',
     ''),
    ('cxx_flags',
     """Flags to pass to the C++ compiler (separate multiple options with
        spaces. If not specified, compiler-specific defaults will be used.""",
     ''),
    ('libdirs',
     'Comma-separated List of additional library directories',
     ''),
    ('env_vars',
     """Environment variables to propagate through to SCons. Either the
        string "all" or a comma separated list of variable names, e.g.
        'LD_LIBRARY_PATH,HOME'.""",
     defaults.env_vars),
    PathVariable(
        'tbb',
        'Location of the Thread Building Blocks (TBB) header and library files',
        '', PathVariable.PathAccept),
    BoolVariable(
      'use_tbb',
      """Enable or disable use of the TBB library. Setting this to "False"
         will prevent Ember from utilizing multiple processors.
      """,
      True),
    PathVariable(
        'python_cmd',
        """Path to the Python interpreter to be used for building the Python module,
           if different from the interpreter being used by SCons.""",
        sys.executable, PathVariable.PathAccept),
    BoolVariable(
        'debug_symbols',
        'Include debug symbols in the compiled module',
        True),
    BoolVariable(
        'debug',
        """Enable asserts, bounds checking and other debugging code; disable
           compiler optimizations.""",
        False),
    ('blas_lapack',
     'Comma-separated list of libraries to include for BLAS/LAPACK support',
     defaults.blas_lapack),
    ('install_args',
     'Command-line arguments passed to "python setup.py install"',
     '--user')
    )

opts.Update(env)
opts.Save('ember.conf', env)

if os.name == 'nt' and 'g++' in env.subst('$CXX'):
    # Compile using mingw
    env = Environment(tools=['mingw', 'subst'], **extraEnvArgs)
    env.Append(LINKFLAGS=['-static-libgcc', '-static-libstdc++'])
    opts.Update(env)

if 'help' in COMMAND_LINE_TARGETS:
    # Print help about configuration options and exit
    print("""
        ************************************************
        *   Configuration options for building Ember   *
        ************************************************

The following options can be passed to SCons to customize the Ember build
process. They should be given in the form:

    scons build option1=value1 option2=value2

Variables set in this way will then be stored in the 'ember.conf' file and
reusd automatically on subsequent invocations of scons. Alternatively, the
configuration variables can be entered directly into 'ember.conf' before
running 'scons build'. The format of this file is:

    option1 = 'value1'
    option2 = 'value2'

        ************************************************
""")
    for opt in opts.options:
        print('\n'.join(formatOption(env, opt)))
    sys.exit(0)

# Print values of all build options:
print("Configuration variables read from 'ember.conf' and command line:")
for line in open('ember.conf'):
    print('   ', line.strip())
print()

env['OS'] = platform.system()

# Copy in external environment variables
if env['env_vars'] == 'all':
    env['ENV'].update(os.environ)
    if 'PYTHONHOME' in env['ENV']:
        del env['ENV']['PYTHONHOME']
elif env['env_vars']:
    for name in env['env_vars'].split(','):
        if name in os.environ:
            env['ENV'][name] = os.environ[name]
        elif name not in defaults.env_vars:
            print('WARNING: failed to propagate environment variable', name)

cantera = ['cantera'] + env['cantera_libs'].split(',')
sundials = 'sundials_nvecserial sundials_ida sundials_cvode'.split()
lastlibs = ['tbb'] if env['use_tbb'] else []

if os.name == 'nt':
    if 'g++' in env.subst('$CXX'):
        lastlibs += ['python27']
        tbbCompiler = 'mingw'
        if target_arch == 'amd64':
            env.Append(CPPDEFINES='MS_WIN64')
    tbbLibDir = env['tbb']+'/lib/%s/%s' % (tbbArch, tbbCompiler)
else:
    tbbLibDir = env['tbb']+'/lib'

if platform.system() == 'Darwin':
    env.Append(FRAMEWORKS=['Accelerate'])

lastlibs += env['blas_lapack'].split(',')
include_dirs = env['include'].split(',')
library_dirs = [env.Dir('lib').abspath] + env['libdirs'].split(',')

if os.name == 'nt':
    library_dirs.append(get_config_var('prefix') + '/libs')

if env['cantera']:
    include_dirs.append(env['cantera'] + '/include')
    library_dirs.append(env['cantera'] + '/lib')

if env['sundials']:
    include_dirs.append(env['sundials'] + '/include')
    library_dirs.append(env['sundials'] + '/lib')

if env['eigen']:
    include_dirs.append(env['eigen'])

if env['boost']:
    include_dirs.append(env['boost'] + '/include')
    library_dirs.append(env['boost'] + '/lib')

if env['use_tbb'] and env['tbb']:
    include_dirs.append(env['tbb'] + '/include')
    library_dirs.append(tbbLibDir)

include_dirs.extend([get_config_var('INCLUDEPY'),
                     np.get_include()])

if ('g++' in env.subst('$CXX')
    or 'clang++' in env.subst('$CXX')
    or 'icpc' in env.subst('$CXX')):
    flags = ['-ftemplate-depth-128', '-std=c++0x', '-fPIC', '-g', '-Wall', '-pthread',
             '-Wno-deprecated-declarations']
    linkflags = ['-pthread']
    defines = []
    if env['debug_symbols']:
        flags.append('-g')

    if 'cygwin' in platform.system().lower():
        flags.remove('-std=c++0x')
        flags.append('-std=gnu++0x')

    if env['OS'] == 'nt' or 'cygwin' in platform.system().lower():
        flags.remove('-fPIC')

    if env['debug']:
        flags.extend(['-O0','-fno-inline'])
    else:
        flags.extend(['-O3', '-finline-functions', '-Wno-inline'])
        defines.append('NDEBUG')

elif env.subst('$CXX') == 'cl':
    flags = ['/nologo', '/W3', '/Zc:wchar_t', '/Zc:forScope', '/EHsc', '/MD']
    linkflags=[]
    defines = ['_SCL_SECURE_NO_WARNINGS', '_CRT_SECURE_NO_WARNINGS']
    if env['debug_symbols']:
        env.Append(LINKFLAGS='/DEBUG')
        flags.append('/Zi')

    if env['debug']:
        flags.extend(['/Od', '/Ob0', '/MD'])
    else:
        flags.append('/O2')
        defines.append('NDEBUG')

else:
    print('error: unknown c++ compiler: "%s"' % env['CXX'])
    sys.exit(1)

env.Append(CPPPATH=include_dirs,
           LIBPATH=library_dirs,
           CXXFLAGS=env['cxx_flags'] or flags,
           CPPDEFINES=defines,
           LINKFLAGS=linkflags,
           LIBS=sundials + cantera + lastlibs)

def CheckMemberFunction(context, function, includes=""):
    context.Message('Checking for %s... ' % function)
    src = """
%(include)s
int main(int argc, char** argv) {
    &%(func)s;
    return 0;
}
""" % {'func':function,
       'include':includes}
    result = context.TryLink(src, '.cpp')
    context.Result(result)
    return result

def get_expression_value(includes, expression):
    s = ['#include ' + i for i in includes]
    s.extend(('#include <iostream>',
              'int main(int argc, char** argv) {',
              '    std::cout << %s << std::endl;' % expression,
              '    return 0;',
              '}\n'))
    return '\n'.join(s)

configInfo = {}
if env['use_tbb']:
    configInfo['EMBER_USE_TBB'] = 1

import SCons.Conftest, SCons.SConf
tests = {}
conf = Configure(env, custom_tests={'CheckMemberFunction': CheckMemberFunction})
context = SCons.SConf.CheckContext(conf)

# Check for required headers
fail = False
required_headers = [('cantera/thermo/IdealGasPhase.h', '""'),
                    ('cvode/cvode.h', '<>'),
                    ('Eigen/Dense', '<>')]
if env['use_tbb']:
    required_headers.append(('tbb/parallel_for.h', '""'))
for header, quotes in required_headers:
    fail |= SCons.Conftest.CheckHeader(context, header, language='C++',
                                     include_quotes=quotes)
if fail:
    raise EnvironmentError("Failed to a required header file. "
                           "See config.log for details.")

# Check for required libraries
src = get_expression_value(["<cmath>"], "sin(3.14)")
retcode, retval = conf.TryRun(src, '.cpp')
if not retcode:
    raise EnvironmentError("Failed to a required library."
                           "See config.log for details.")

cantera_version_source = get_expression_value(['"cantera/base/config.h"'],
                                              'CANTERA_VERSION')
retcode, cantera_version = conf.TryRun(cantera_version_source, '.cpp')

min_cantera_version = '2.3.0a3'
if StrictVersion(cantera_version.strip()) < StrictVersion(min_cantera_version):
    raise EnvironmentError("Ember requires Cantera {} or newer, but the "
        "installed version of Cantera is {}.".format(
        min_cantera_version, cantera_version.strip()))

# Determine Sundials version
sundials_version_source = get_expression_value(['"sundials/sundials_config.h"'],
                                                   'SUNDIALS_PACKAGE_VERSION')
retcode, sundials_version = conf.TryRun(sundials_version_source, '.cpp')
if retcode == 0:
    print("Failed to determine Sundials version.")
    print("See 'config.log' for details.")
    sys.exit(1)

# Ignore the minor version and convert to integer, e.g. 2.4.x -> 24
configInfo['EMBER_SUNDIALS_VERSION'] = ''.join(sundials_version.strip().split('.')[:2])
print("""INFO: Using Sundials version %s""" % sundials_version.strip())

config_h = env.Command('src/config.h',
                       'src/config.h.in',
                       ConfigBuilder(configInfo))
env.AlwaysBuild(config_h)

common_objects = env.SharedObject(Glob('build/core/*.cpp'))

corelib = env.Library('build/core/ember', common_objects)
env.Alias('build', corelib)

if os.name == 'nt':
    if env['use_tbb']:
        for dest in ['python/ember/TBB.dll', 'bin/TBB.dll']:
            tbb = env.Command(dest,
                              env['tbb']+'/bin/%s/%s/TBB.dll' % (tbbArch, tbbCompiler),
                              Copy('$TARGET', '$SOURCE'))
            env.Alias('build', tbb)
    py_gui1 = env.Command('python/scripts/ember-script.pyw', 'python/scripts/ember',
                          Copy('$TARGET', '$SOURCE'))
    script = ('''"from pkg_resources import resource_string; open('$TARGET', 'wb').write(resource_string('setuptools', 'gui.exe'))"''')
    py_gui2 = env.Command('python/scripts/ember.exe', py_gui1, '$python_cmd -c ' + script)
    env.Alias('build', [py_gui1, py_gui2])

# Workaround for https://bugs.python.org/issue11566
if os.name == 'nt' and env.subst('$CXX') != 'cl':
    env.Append(CPPDEFINES='_hypot=hypot')

# The Python module
make_setup = env.SubstFile('#python/setup.py', '#python/setup.py.in')
script = ('from distutils.sysconfig import *\n'
          'print(get_config_var("EXT_SUFFIX") or get_config_var("SO"))\n'
          'print(get_config_var("INCLUDEPY"))\n'
          'print(get_config_var("LIBRARY") or get_config_var("LDLIBRARY"))\n'
          'print(get_config_var("LIBDIR"))\n'
          'print(get_python_version())\n')

suffix, includepy, pylib, pylibdir, target_py_version = [s.strip()
    for s in getCommandOutput(env['python_cmd'], '-c', script).split()]

if pylibdir != 'None':
    env.Append(LIBPATH=pylibdir)

def compile_cython(target, source, env):
    cythonize([f.abspath for f in source])

# extract 'pythonX.Y' from 'libpythonX.Y.dll.a' or 'libpythonX.Y.a'
if pylib != 'None':
    pylib = pylib[3:]
    if pylib.endswith('.a'):
        pylib = pylib[:-2]
    if pylib.endswith('.dll'):
        pylib = pylib[:-4]
    python_libs = [pylib]
else:
    python_libs = []

env.Command('python/ember/_ember.cpp', ['python/ember/_ember.pyx'], compile_cython)
env.Depends('python/ember/_ember.cpp', 'python/ember/_ember.pxd')

cyenv = env.Clone() # environment for compiling the Cython module
cyenv.Prepend(CPPPATH='src', LIBPATH='build/core', LIBS=corelib)
cyenv.Append(CPPPATH=includepy)

if env['OS'] == 'Darwin':
    cyenv.Append(LINKFLAGS='-undefined dynamic_lookup')
elif 'cygwin' in platform.system().lower():
    cyenv.Append(LIBS=python_libs)

# Suppress warnings from Cython-generated code
if 'g++' in env.subst('$CXX'):
    cyenv.Append(CXXFLAGS=['-w'])
elif os.name == 'nt' and env.subst('$CXX') == 'cl':
    cyenv.Append(CXXFLAGS=['/w'])

py_ext = cyenv.LoadableModule('#build/python/ember/_ember%s' % suffix, '#python/ember/_ember.cpp',
                              LIBPREFIX='', SHLIBPREFIX='', SHLIBSUFFIX=suffix, LIBSUFFIXES=[suffix])

setup_cmd = 'cd python && $python_cmd setup.py build --build-lib=../build/python '

if os.name == 'nt':
    env.Depends(make_setup, [py_gui1, py_gui2])

py_build = env.Command('build/python/ember/__init__.py', py_ext, setup_cmd)
env.Alias('build', py_build)
env.Depends(py_build, [py_ext, make_setup])
env.Depends(py_build, mglob(env, 'python/ember', 'py'))
env.Depends(py_build, mglob(env, 'python/ember/test', 'py'))

py_install = env.Command('dummy_target', py_build, setup_cmd + 'install $install_args')
env.Alias('install', py_install)

py_msi = env.Command('dummy_target2', py_build,
                     setup_cmd + 'bdist_msi --dist-dir=..' +
                     ' --target-version=' + target_py_version)
env.Alias('msi', py_msi)

# GoogleTest tests
testenv = env.Clone()
testenv.Append(LIBS=['gtest'],
               CPPPATH=['ext/gtest/include'])

if pylibdir != 'None':
    testenv.AppendENVPath('LD_LIBRARY_PATH', pylibdir)

if os.name == 'nt':
    testenv['ENV']['PATH'] += ';' + Dir('lib').abspath
    if env.subst('CXX') != 'cl':
        testenv.Append(LIBS=python_libs)
else:
    testenv.Append(LIBS=python_libs)

test_program = testenv.Program('bin/unittest',
                               Glob('build/test/*.cpp') + common_objects)
run_test = testenv.Command('test_dummy', test_program[0].abspath, '$SOURCE $TARGET')
test_alias = Alias('test', run_test)

# Google Test
Export('env')
VariantDir('ext/build', 'ext', duplicate=0)
SConscript('ext/build/SConscript')

# Python unittest tests
def unittestRunner(target, source, env):
    workDir = Dir('build/work').abspath
    if not os.path.isdir(workDir):
        os.mkdir(workDir)

    environ = dict(env['ENV'])
    if 'PYTHONPATH' in environ:
        environ['PYTHONPATH'] += os.path.pathsep + Dir('build/python').abspath
    else:
        environ['PYTHONPATH'] = Dir('build/python').abspath
    return subprocess.call([env.subst('$python_cmd'), '-m','unittest',
                            'discover', '-b', '-v', '-s', 'build/python'],
                           env=environ)

py_tests = testenv.Command('pytest_dummy', py_build, unittestRunner)
Alias('test', py_tests)
