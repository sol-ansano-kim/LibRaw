import excons
import sys
import tarfile
import os
import glob
import re


env = excons.MakeBaseEnv()


staticlib = (excons.GetArgument("libraw-static", 1, int) != 0)
out_incdir = excons.OutputBaseDirectory() + "/include"
out_libdir = excons.OutputBaseDirectory() + "/lib"

excons.SetArgument("with-jpeg8", 1)

prjs = []
defs = ["USE_LCMS2", "USE_JPEG", "USE_JPEG8"]
deps = []
opts = {}
libs = []
cflags = ""
cppflags = ""

major = minor = patch = so_cur = so_rev = so_age = None
with open("libraw/libraw_version.h", "r") as f:
    for l in f.readlines():
        res = re.search("#define\sLIBRAW_MAJOR_VERSION\s+(\d+)+", l)
        if res:
            major = res.group(1)

        res = re.search("#define\sLIBRAW_MINOR_VERSION\s+(\d+)+", l)
        if res:
            minor = res.group(1)

        res = re.search("#define\sLIBRAW_PATCH_VERSION\s+(\d+)+", l)
        if res:
            patch = res.group(1)

        res = re.search("#define\sLIBRAW_SHLIB_CURRENT\s+(\d+)+", l)
        if res:
            so_cur = res.group(1)

        res = re.search("#define\sLIBRAW_SHLIB_REVISION\s+(\d+)+", l)
        if res:
            so_rev = res.group(1)

        res = re.search("#define\sLIBRAW_SHLIB_AGE\s+(\d+)+", l)
        if res:
            so_age = res.group(1)


if major is None or minor is None or patch is None or so_cur is None or so_rev is None or so_age is None:
    print ".. parsing libraw/libraw_version.h failed."
    sys.exit(1)

if sys.platform != "win32":
    cppflags += " -Wno-unused-parameter"
    cppflags += " -Wno-unused-variable"
    cppflags += " -Wno-missing-field-initializers"
    if sys.platform != "darwin":
        cppflags += " -Wno-unused-but-set-variable"
    else:
        cppflags += " -Wno-unused-private-field"
        cppflags += " -Wno-unused-function"
    libs.append("m")
    if not staticlib:
        defs.append("LIBRAW_NODLL")
        defs.append("LIBRAW_BUILDLIB")
        
else:
    cppflags += " /wd4101"
    cppflags += " /wd4996"
    defs.append("_CRT_SECURE_NO_WARNINGS")
    defs.append("_ATL_SECURE_NO_WARNINGS")
    defs.append("_AFX_SECURE_NO_WARNINGS")
    defs.append("DJGPP")
    

if sys.platform == "darwin":
    cppflags += " -Wno-deprecated-declarations"

cppflags += " -w -O4"

# LCMS setup
lcms_libname = excons.GetArgument("lcms-lib-name", "lcms2")
lcms_static = (excons.GetArgument("lcms-static", 1, int) != 0)
lcms_inc, lcms_lib = excons.GetDirs("lcms", silent=True)
if lcms_inc is None and lcms_lib is None:
    if not os.path.isdir("ext/lcms2-2.8"):
        print("=== Extracting lcms2-2.8.tar.gz...")
        f = tarfile.open("ext/lcms2-2.8.tar.gz")
        f.extractall("ext")
        f.close()
    lcms_static = True
    lcms_cppflags = cppflags
    if sys.platform != "win32":
        lcms_cppflags += " -Wno-strict-aliasing"

    prjs.append({"name": "lcms2",
                 "type": "staticlib",
                 "cflags": cflags,
                 "cppflags": lcms_cppflags,
                 "incdirs": ["ext/lcms2-2.8/include"],
                 "srcs": glob.glob("ext/lcms2-2.8/src/*.c"),
                 "deps": ["libjpeg"]})
    lcms_inc = "ext/lcms2-2.8/include"
    lcms_lib = excons.OutputBaseDirectory() + "/lib"

def LibLcms2Name(static=False):
    libname = "lcms"
    if sys.platform == "win32" and static:
       libname += "-static"
    return libname

def Lcms2Require(env, static=False):
    env.Append(CPPPATH=[out_incdir])
    env.Append(LIBPATH=[out_libdir])
    if sys.platform == "win32" and not static:
        # EXTERN macro should be redefined from "extern type" to "__declspec(dllimport) type"
        pass
    excons.Link(env, LibLcms2Name(static=static), static=static, force=True, silent=False)


# jpeg setup
def JpegLibname(static):
    name = "jpeg"
    if sys.platform == "win32" and static:
        name += "-static"
    return name

rv = excons.cmake.ExternalLibRequire({}, name="jpeg", libnameFunc=JpegLibname)
if rv is None:
    excons.PrintOnce("Build jpeg from sources ...")
    excons.Call("libjpeg-turbo", imp=["RequireLibjpeg", "LibjpegPath"])
    if sys.platform == "win32":
        deps.append(excons.cmake.OutputsCachePath("libjpeg"))
    else:
        deps.append(excons.automake.OutputsCachePath("libjpeg"))

    jpeg_static = True

    def JpegRequire(env):
        RequireLibjpeg(env, static=jpeg_static)
else:
    JpegRequire = rv


srcs =["internal/dcraw_common.cpp",
       "internal/dcraw_fileio.cpp",
       "internal/demosaic_packs.cpp",
       "src/libraw_cxx.cpp",
       "src/libraw_c_api.cpp",
       "src/libraw_datastream.cpp"]

prjs.append({"name": "raw",
             "type": "staticlib" if staticlib else "sharedlib",
             "cflags": cflags,
             "defs": defs,
             "cppflags": cppflags,
             "incdirs": [".", out_incdir, "ext/lcms2-2.8/include"],
             "srcs": srcs,
             "deps": ["libjpeg", "lcms2"],
             "custom": [JpegRequire, Lcms2Require]})


def LibrawName(static=False):
    libname = "raw"
    if sys.platform == "win32" and static:
       libname += "-static"
    return libname

def LibrawRequire(env):
    env.Append(CPPPATH=[out_incdir])
    env.Append(LIBPATH=[out_libdir])

    excons.Link(env, LibrawName(static=staticlib), static=staticlib, force=True, silent=True)

for sam in glob.glob("samples/*.cpp"):
    base = os.path.splitext(os.path.basename(sam))[0]
    prjs.append({"name": base,
                 "type": "testprograms",
                 "cflags": cflags,
                 "cppflags": cppflags,
                 "incdirs": [".", out_incdir],
                 "srcs": [sam],
                 "deps": ["raw"],
                 "custom": [LibrawRequire]})


targets = excons.DeclareTargets(env, prjs)
