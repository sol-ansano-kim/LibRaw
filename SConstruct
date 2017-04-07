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
with_jpg = (excons.GetArgument("libraw-with-jpeg", 0, int) != 0)
with_lcms2 = (excons.GetArgument("libraw-with-lcms2", 0, int) != 0)

excons.SetArgument("with-jpeg8", 1)

prjs = []
defs = []
deps = []
opts = {}
libs = []
customs = []
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
    cppflags += " -Wno-unused-function"
    cppflags += " -Wno-sign-compare"

    if sys.platform != "darwin":
        cppflags += " -Wno-unused-but-set-variable"
        cppflags += " -Wno-parentheses"
        cppflags += " -Wno-type-limits"
        cppflags += " -Wno-narrowing"
        cppflags += " -Wno-maybe-uninitialized"
        cppflags += " -Wno-aggressive-loop-optimizations"
    else:
        cppflags += " -Wno-unused-private-field"
        cppflags += " -Wno-constant-conversion"
        cppflags += " -Wno-sometimes-uninitialized"

    libs.append("m")
        
else:
    cppflags += " /LD"
    cppflags += " /wd4101"
    cppflags += " /wd4996"
    defs.append("_CRT_SECURE_NO_WARNINGS")
    defs.append("_ATL_SECURE_NO_WARNINGS")
    defs.append("_AFX_SECURE_NO_WARNINGS")
    defs.append("DJGPP")
    if staticlib:
        defs.append("LIBRAW_NODLL")
    else:
        defs.append("LIBRAW_BUILDLIB")

if sys.platform == "darwin":
    cppflags += " -Wno-deprecated-declarations"



# LCMS setup
def Lcms2Name():
    return ("lib" if sys.platform == "win32" else "") + "lcms2"

if not os.path.isdir("ext/lcms2-2.8"):
    print("=== Extracting lcms2-2.8.tar.gz...")
    f = tarfile.open("ext/lcms2-2.8.tar.gz")
    f.extractall("ext")
    f.close()

lcms_cppflags = cppflags
if sys.platform != "win32":
    lcms_cppflags += " -Wno-strict-aliasing"

lcms_inc = "ext/lcms2-2.8/include"

prjs.append({"name": Lcms2Name(),
             "alias": "lcms2",
             "type": "staticlib",
             "cflags": cflags,
             "cppflags": lcms_cppflags,
             "incdirs": [lcms_inc, out_incdir],
             "install": {out_incdir: glob.glob("%s/*.h" % lcms_inc)},
             "srcs": glob.glob("ext/lcms2-2.8/src/*.c"),
             "deps": ["libjpeg"] if with_jpg else []})

def Lcms2Require(env):
    env.Append(CPPPATH=[lcms_inc, out_incdir])
    env.Append(LIBPATH=[out_libdir])
    excons.Link(env, Lcms2Name(), static=True, force=True, silent=True)


# jpeg setup
def JpegLibname(static):
    name = "jpeg"
    if sys.platform == "win32" and static:
        name = "lib" + name + "-static"
    return name

rv = excons.cmake.ExternalLibRequire({}, name="jpeg", libnameFunc=JpegLibname)
if rv is None:
    excons.PrintOnce("Build jpeg from sources ...")
    excons.Call("libjpeg-turbo", imp=["LibjpegName"], overrides={"ENABLE_SHARED": "OFF"})
    if sys.platform == "win32":
        excons.cmake.OutputsCachePath("libjpeg")
    else:
        excons.automake.OutputsCachePath("libjpeg")

    jpeg_static = True


def JpegRequire(env, static=False):
    env.Append(CPPPATH=[excons.OutputBaseDirectory() + "/include"])
    env.Append(LIBPATH=[excons.OutputBaseDirectory() + "/lib"])
    excons.Link(env, JpegLibname(static=True), static=True, silent=True)


srcs =["internal/dcraw_common.cpp",
       "internal/dcraw_fileio.cpp",
       "internal/demosaic_packs.cpp",
       "src/libraw_cxx.cpp",
       "src/libraw_c_api.cpp",
       "src/libraw_datastream.cpp"]


if with_jpg:
    defs += ["USE_JPEG8", "USE_JPEG"]
    deps += ["libjpeg"]
    customs += [JpegRequire]

if with_lcms2:
    defs += ["USE_LCMS2"]
    deps += ["lcms2"]
    customs += [Lcms2Require]
    

def LibrawName(static=False):
    bn = "lib" if sys.platform == "win32" else ""
    libname = bn + "raw"
    if sys.platform == "win32" and static:
       libname += "-static"
    return libname

prjs.append({"name": LibrawName(staticlib),
             "type": "staticlib" if staticlib else "sharedlib",
             "alias": "libraw",
             "cflags": cflags,
             "defs": defs,
             "cppflags": cppflags,
             "incdirs": [".", out_incdir],
             "srcs": srcs,
             "version": "%s.%s.%s" % (major, minor, patch),
             "soname": "libraw.so.%s" % (major),
             "deps": deps,
             "install": {"%s/libraw" % out_incdir: glob.glob("libraw/*.h")},
             "custom": customs})


def LibrawRequire(env):
    env.Append(CPPPATH=[out_incdir])
    env.Append(LIBPATH=[out_libdir])

    excons.Link(env, LibrawName(static=staticlib), static=staticlib, force=True, silent=False)

for sam in glob.glob("samples/*.cpp"):
    if "unprocessed_raw.cpp" in sam:
        if sys.platform == "win32":
            continue

    base = os.path.splitext(os.path.basename(sam))[0]
    prjs.append({"name": base,
                 "alias": "tests",
                 "type": "testprograms",
                 "cflags": cflags,
                 "cppflags": cppflags,
                 "incdirs": [out_incdir],
                 "defs": defs,
                 "srcs": [sam],
                 "deps": [LibrawName(staticlib)] + deps,
                 "custom": [LibrawRequire] + customs})

targets = excons.DeclareTargets(env, prjs)

Default("tests")
