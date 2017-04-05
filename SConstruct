import shutil
import os
import excons


env = excons.MakeBaseEnv()

deps = []


staticlib = (excons.GetArgument("libraw-static", 1, int) != 0)
out_incdir = excons.OutputBaseDirectory() + "/include"
out_libdir = excons.OutputBaseDirectory() + "/lib"


#LibRaw-cmake
if not os.path.isfile("CMakeLists.txt"):
    shutil.copy("LibRaw-cmake/CMakeLists.txt", "CMakeLists.txt")

if not os.path.isdir("cmake"):
    shutil.copytree("LibRaw-cmake/cmake", "cmake")


excons.PrintOnce("Build jpeg from sources ...")
excons.Call("libjpeg-turbo", imp=["RequireLibjpeg", "LibjpegName"], overrides={"WITH_JPEG8": 1})
jpeg_incdir, jpeg_libdir, jpeg_path = out_incdir, out_libdir, LibjpegName(static=True)
deps.append("libjpeg")


prjs = []


prjs.append({"name": "libraw",
             "type": "cmake",
             "cmake-opts": {"BUILD_SHARED_LIBS": (0 if staticlib else 1),
                            "INSTALL_CMAKE_MODULE_PATH": "modules",
                            "CMAKE_VERVOSE_MAKEFILE": 1,
                            "JPEG_INCLUDE_DIR": os.path.abspath(jpeg_incdir),
                            "JPEG_LIBRARY": os.path.abspath(jpeg_path),
                            "JPEG_LIB_VERSION": 82},
             "cmake-srcs": excons.CollectFiles("libraw", patterns=["*.c"], recursive=True),
             "deps": deps})


excons.DeclareTargets(env, prjs)


excons.AddHelpOptions(libraw="""LIBRAW OPTIONS
    libraw-static=0|1  : Toggle between static and shared library build [1]""")


def RequireLibRaw(env):
    env.Append(CPPPATH=[out_incdir])
    env.Append(LIBPATH=[out_libdir])
    if staticlib:
        if sys.platform == "win32":
            env.Append(LIBS=["raw_r"])
        else:
            if not env.StaticallyLink(env, "libraw_r", silent=True):
                env.Append(LIBS=["raw_r"])
    else:
        env.Append(LIBS=["raw_r"])


def LibRawName():
    if sys.platform == "win32":
        basename = "libraw_r.lib"
    else:
        basename = "libraw_r.a"

    return out_libdir + "/" + basename


Export("RequireLibRaw LibRawName")
