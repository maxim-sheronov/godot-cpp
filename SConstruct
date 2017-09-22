#!python
import os
import sys
import methods
import subprocess

# Local dependency paths, adapt them to your setup
godot_headers_path = ARGUMENTS.get("headers", "../godot_headers/")
godot_bin_path = ARGUMENTS.get("godotbinpath", "../godot_fork/bin/")
# for Windows
godot_lib_path = ARGUMENTS.get("godotlibpath", godot_bin_path)

target = ARGUMENTS.get("target", "debug")
platform = ARGUMENTS.get("platform", ARGUMENTS.get("p", ""))
godot_name = "godot." + ("x11" if platform == "linux" else platform) + ".tools.64"

cpppath = ['.', 'include', 'include/core', godot_headers_path]
output_name = 'godot_cpp_bindings'


def add_sources(sources, directory):
    for file in os.listdir(directory):
        if file.endswith('.cpp'):
            sources.append(directory + '/' + file)


def common_setup(env):
    opts = Variables([], ARGUMENTS)

    # Target build options
    opts.Add('p', "Platform (alias for 'platform')", '')
    opts.Add('target', "Compilation target (debug/release_debug/release)", 'debug')

    # Advanced options
    opts.Add('extra_suffix', "Custom extra suffix added to the base filename of all generated binary files", '')
    opts.Add('warnings', "Set the level of warnings emitted during compilation (extra/all/moderate/no)", 'no')

    # Android
    opts.Add('ANDROID_NDK_ROOT', 'Path to the Android NDK', os.environ.get("ANDROID_NDK_ROOT", 0))
    opts.Add('ndk_platform', 'Target platform (android-<api>, e.g. "android-18")', "android-18")
    opts.Add('android_arch', 'Target architecture (armv7/armv6/arm64v8/x86)', "armv7")
    opts.Add('android_neon', 'Enable NEON support (armv7 only)', "yes")
    opts.Add('android_stl', 'Enable Android STL support (for modules)', "yes")

    opts.Update(env)  # update environment
    Help(opts.GenerateHelpText(env))  # generate help

    env.Append(CPPPATH=cpppath)

    if ARGUMENTS.get("use_llvm", "no") == "yes":
        env["CXX"] = "clang++"


def build(env):
    if ARGUMENTS.get("generate_bindings", "no") == "yes":
        godot_executable = godot_bin_path + godot_name
        if platform == "windows":
            godot_executable += ".exe"
        if env["CXX"] == "clang++":
            godot_executable += ".llvm"

        # TODO Generating the API should be done only if the Godot build is more recent than the JSON file
        json_api_file = 'godot_api.json'
        subprocess.call([godot_executable, '--gdnative-generate-json-api', json_api_file])

        # actually create the bindings here

        import binding_generator
        binding_generator.generate_bindings(json_api_file)

    output_path = 'bin/' + platform + '/'
    if 'eabi' in env and env['eabi'] != '':
        output_path += env['eabi'] + '/'
    output_path += output_name

    sources = []
    add_sources(sources, "src/core")
    add_sources(sources, "src")

    library = env.StaticLibrary(target=output_path, source=sources)
    Default(library)


if platform == 'linux':
    env = Environment()
    common_setup(env)

    if target == 'debug':
        env.Append(CCFLAGS = ['-fPIC', '-g','-O3', '-std=c++14'])
    else:
        env.Append(CCFLAGS = ['-fPIC', '-O3', '-std=c++14'])
        env.Append(LINKFLAGS = ['-s'])

    build(env)

elif platform == 'windows':
    env = Environment(ENV = os.environ)
    common_setup(env)

    if os.getenv("VCINSTALLDIR") != None:
        if env['target'] == "debug":
            env.Append(CCFLAGS = ['-EHsc', '-D_DEBUG', '/MDd'])
        else:
            env.Append(CCFLAGS = ['-O2', '-EHsc', '-DNDEBUG', '/MD'])
    else:
        if target == 'debug':
            env.Append(CCFLAGS = ['-fPIC', '-g','-O3', '-std=c++14'])
        else:
            env.Append(CCFLAGS = ['-fPIC', '-O3', '-std=c++14'])
            env.Append(LINKFLAGS = ['-s'])

    env.Append(LIBPATH=[godot_lib_path])
    env.Append(LIBS=[godot_name])

    build(env)

elif platform == "osx":
    env = Environment()
    common_setup(env)

    env.Append(CCFLAGS = ['-g','-O3', '-std=c++14', '-arch', 'x86_64'])
    env.Append(LINKFLAGS = ['-arch', 'x86_64', '-framework', 'Cocoa', '-Wl,-undefined,dynamic_lookup'])

    build(env)

elif platform == 'android':
    env = Environment()
    common_setup(env)

    tools = env['TOOLS']
    if "mingw" in tools:
        tools.remove('mingw')
    if "applelink" in tools:
        tools.remove("applelink")
        env.Tool('gcc')

    env.Replace(tools=tools)

    # Workaround for MinGW. See:
    # http://www.scons.org/wiki/LongCmdLinesOnWin32
    if (os.name == "nt"):

        import subprocess

        def mySubProcess(cmdline, env):
            # print("SPAWNED : " + cmdline)
            startupinfo = subprocess.STARTUPINFO()
            startupinfo.dwFlags |= subprocess.STARTF_USESHOWWINDOW
            proc = subprocess.Popen(cmdline, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
                                    stderr=subprocess.PIPE, startupinfo=startupinfo, shell=False, env=env)
            data, err = proc.communicate()
            rv = proc.wait()
            if rv:
                print("=====")
                print(err)
                print("=====")
            return rv

        def mySpawn(sh, escape, cmd, args, env):

            newargs = ' '.join(args[1:])
            cmdline = cmd + " " + newargs

            rv = 0
            if len(cmdline) > 32000 and cmd.endswith("ar"):
                cmdline = cmd + " " + args[1] + " " + args[2] + " "
                for i in range(3, len(args)):
                    rv = mySubProcess(cmdline + args[i], env)
                    if rv:
                        break
            else:
                rv = mySubProcess(cmdline, env)

            return rv

        env['SPAWN'] = mySpawn

    ## Architecture

    if env['android_arch'] not in ['armv7', 'armv6', 'arm64v8', 'x86']:
        env['android_arch'] = 'armv7'

    neon_text = ""
    if env["android_arch"] == "armv7" and env['android_neon'] == 'yes':
        neon_text = " (with NEON)"
    print("Building for Android (" + env['android_arch'] + ")" + neon_text)

    can_vectorize = True
    if env['android_arch'] == 'x86':
        env['ARCH'] = 'arch-x86'
        env['extra_suffix'] = ".x86" + env['extra_suffix']
        target_subpath = "x86-4.9"
        abi_subpath = "i686-linux-android"
        arch_subpath = "x86"
        env['eabi'] = arch_subpath
        env["x86_libtheora_opt_gcc"] = True
    elif env['android_arch'] == 'armv6':
        env['ARCH'] = 'arch-arm'
        env['extra_suffix'] = ".armv6" + env['extra_suffix']
        target_subpath = "arm-linux-androideabi-4.9"
        abi_subpath = "arm-linux-androideabi"
        arch_subpath = "armeabi"
        env['eabi'] = arch_subpath
        can_vectorize = False
    elif env["android_arch"] == "armv7":
        env['ARCH'] = 'arch-arm'
        target_subpath = "arm-linux-androideabi-4.9"
        abi_subpath = "arm-linux-androideabi"
        arch_subpath = "armeabi-v7a"
        env['eabi'] = arch_subpath
        if env['android_neon'] == 'yes':
            env['extra_suffix'] = ".armv7.neon" + env['extra_suffix']
        else:
            env['extra_suffix'] = ".armv7" + env['extra_suffix']
    elif env["android_arch"] == "arm64v8":
        env['ARCH'] = 'arch-arm64'
        target_subpath = "aarch64-linux-android-4.9"
        abi_subpath = "aarch64-linux-android"
        arch_subpath = "arm64-v8a"
        env['eabi'] = arch_subpath
        env['extra_suffix'] = ".armv8" + env['extra_suffix']

    ## Build type

    if (env["target"].startswith("release")):
        env.Append(LINKFLAGS=['-O2'])
        env.Append(CPPFLAGS=['-O2', '-DNDEBUG', '-ffast-math', '-funsafe-math-optimizations', '-fomit-frame-pointer'])
        if (can_vectorize):
            env.Append(CPPFLAGS=['-ftree-vectorize'])
        if (env["target"] == "release_debug"):
            env.Append(CPPFLAGS=['-DDEBUG_ENABLED'])
    elif (env["target"] == "debug"):
        env.Append(LINKFLAGS=['-O0'])
        env.Append(CPPFLAGS=['-O0', '-D_DEBUG', '-UNDEBUG', '-DDEBUG_ENABLED',
                             '-DDEBUG_MEMORY_ENABLED', '-g', '-fno-limit-debug-info'])

    ## Compiler configuration

    env['SHLIBSUFFIX'] = '.so'

    if env['PLATFORM'] == 'win32':
        env.Tool('gcc')
        env.use_windows_spawn_fix()

    mt_link = True
    if (sys.platform.startswith("linux")):
        host_subpath = "linux-x86_64"
    elif (sys.platform.startswith("darwin")):
        host_subpath = "darwin-x86_64"
    elif (sys.platform.startswith('win')):
        if (platform.machine().endswith('64')):
            host_subpath = "windows-x86_64"
            if env["android_arch"] == "arm64v8":
                mt_link = False
        else:
            mt_link = False
            host_subpath = "windows"

    compiler_path = env["ANDROID_NDK_ROOT"] + "/toolchains/llvm/prebuilt/" + host_subpath + "/bin"
    gcc_toolchain_path = env["ANDROID_NDK_ROOT"] + "/toolchains/" + target_subpath + "/prebuilt/" + host_subpath
    tools_path = gcc_toolchain_path + "/" + abi_subpath + "/bin"

    # For Clang to find NDK tools in preference of those system-wide
    env.PrependENVPath('PATH', tools_path)

    env['CC'] = compiler_path + '/clang'
    env['CXX'] = compiler_path + '/clang++'
    env['AR'] = tools_path + "/ar"
    env['RANLIB'] = tools_path + "/ranlib"
    env['AS'] = tools_path + "/as"

    sysroot = env["ANDROID_NDK_ROOT"] + "/platforms/" + env['ndk_platform'] + "/" + env['ARCH']
    common_opts = ['-fno-integrated-as', '-gcc-toolchain', gcc_toolchain_path]

    ## Compile flags

    env.Append(CPPFLAGS=["-isystem", sysroot + "/usr/include"])
    env.Append(CPPFLAGS='-fpic -ffunction-sections -funwind-tables -fstack-protector-strong -fvisibility=hidden -fno-strict-aliasing'.split())
    env.Append(CPPFLAGS='-DNO_STATVFS -DGLES2_ENABLED'.split())

    env['neon_enabled'] = False
    if env['android_arch'] == 'x86':
        target_opts = ['-target', 'i686-none-linux-android']
        # The NDK adds this if targeting API < 21, so we can drop it when Godot targets it at least
        env.Append(CPPFLAGS=['-mstackrealign'])

    elif env["android_arch"] == "armv6":
        target_opts = ['-target', 'armv6-none-linux-androideabi']
        env.Append(CPPFLAGS='-D__ARM_ARCH_6__ -march=armv6 -mfpu=vfp -mfloat-abi=softfp'.split())

    elif env["android_arch"] == "armv7":
        target_opts = ['-target', 'armv7-none-linux-androideabi']
        env.Append(CPPFLAGS='-D__ARM_ARCH_7__ -D__ARM_ARCH_7A__ -march=armv7-a -mfloat-abi=softfp'.split())
        if env['android_neon'] == 'yes':
            env['neon_enabled'] = True
            env.Append(CPPFLAGS=['-mfpu=neon', '-D__ARM_NEON__'])
        else:
            env.Append(CPPFLAGS=['-mfpu=vfpv3-d16'])

    elif env["android_arch"] == "arm64v8":
        target_opts = ['-target', 'aarch64-none-linux-android']
        env.Append(CPPFLAGS=['-D__ARM_ARCH_8A__'])
        env.Append(CPPFLAGS=['-mfix-cortex-a53-835769'])

    env.Append(CPPFLAGS=target_opts)
    env.Append(CPPFLAGS=common_opts)

    if (env['android_stl'] == 'yes'):
        env.Append(CPPPATH=[env["ANDROID_NDK_ROOT"] + "/sources/cxx-stl/gnu-libstdc++/4.9/include"])
        env.Append(CPPPATH=[env["ANDROID_NDK_ROOT"] + "/sources/cxx-stl/gnu-libstdc++/4.9/libs/" + arch_subpath + "/include"])
        env.Append(LIBPATH=[env["ANDROID_NDK_ROOT"] + "/sources/cxx-stl/gnu-libstdc++/4.9/libs/" + arch_subpath])
        env.Append(LIBS=["gnustl_static"])
    else:
        env.Append(CXXFLAGS=['-fno-rtti', '-fno-exceptions', '-DNO_SAFE_CAST'])

    ## Link flags

    env['LINKFLAGS'] = ['-shared', '--sysroot=' + sysroot, '-Wl,--warn-shared-textrel']
    if env["android_arch"] == "armv7":
        env.Append(LINKFLAGS='-Wl,--fix-cortex-a8'.split())
    env.Append(LINKFLAGS='-Wl,--no-undefined -Wl,-z,noexecstack -Wl,-z,relro -Wl,-z,now'.split())
    env.Append(LINKFLAGS='-Wl,-soname,libgodot_android.so -Wl,--gc-sections'.split())
    if mt_link:
        env.Append(LINKFLAGS=['-Wl,--threads'])
    env.Append(LINKFLAGS=target_opts)
    env.Append(LINKFLAGS=common_opts)

    env.Append(LIBPATH=[env["ANDROID_NDK_ROOT"] + '/toolchains/arm-linux-androideabi-4.9/prebuilt/' +
                        host_subpath + '/lib/gcc/' + abi_subpath + '/4.9.x'])
    env.Append(LIBPATH=[env["ANDROID_NDK_ROOT"] +
                        '/toolchains/arm-linux-androideabi-4.9/prebuilt/' + host_subpath + '/' + abi_subpath + '/lib'])

    env.Append(CPPPATH=['#platform/android'])
    env.Append(CPPFLAGS=['-DANDROID_ENABLED', '-DUNIX_ENABLED', '-DNO_FCNTL', '-DMPC_FIXED_POINT'])
    env.Append(LIBS=['OpenSLES', 'EGL', 'GLESv3', 'android', 'log', 'z', 'dl'])

    # TODO: Move that to opus module's config
    if("module_opus_enabled" in env and env["module_opus_enabled"] != "no"):
        if (env["android_arch"] == "armv6" or env["android_arch"] == "armv7"):
            env.Append(CFLAGS=["-DOPUS_ARM_OPT"])
        env.opus_fixed_point = "yes"

    build(env)

else:
    print("No valid target platform selected.")
    print("Please run scons again with argument: platform=<string>")




