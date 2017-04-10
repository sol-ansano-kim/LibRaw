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

#excons.SetArgument("libjpeg-jpeg8", 1)

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



# LCMS2 setup
rv = excons.ExternalLibRequire("lcms2")
if not rv["require"]:
    excons.PrintOnce("Build lcms2 from sources ...")
    excons.Call("Little-CMS", imp=["RequireLCMS2"])
    def Lcms2Require(env):
        RequireLCMS2(env)
else:
    Lcms2Require = rv["require"]

# jpeg setup
def JpegLibname(static):
    return "jpeg"

rv = excons.ExternalLibRequire("libjpeg", libnameFunc=JpegLibname)
if not rv["require"]:
    excons.PrintOnce("Build jpeg from sources ...")
    excons.Call("libjpeg-turbo", imp=["RequireLibjpeg"])
    def JpegRequire(env):
        RequireLibjpeg(static=(excons.GetArgument("libjpeg-static", 1, 0) != 0))
else:
    JpegRequire = rv["require"]


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
    


def LibrawName():
    bn = "lib" if sys.platform == "win32" else ""
    libname = bn + "raw"
    if sys.platform == "win32" and staticlib:
       libname += "-static"
    return libname

def LibrawPath():
    name = LibrawName()
    if sys.platform == "win32":
        libname = name + ".lib"
    else:
        libname = "lib" + name + (".a" if staticlib else excons.SharedLibraryLinkExt())
    return out_libdir + "/" + libname

def RequireLibraw(env):
    if staticlib:
        env.Append(CPPDEFINES=["LIBRAW_NODLL"])
    env.Append(CPPPATH=[out_incdir])
    env.Append(LIBPATH=[out_libdir])
    excons.Link(env, LibrawName(), static=staticlib, force=True, silent=True)


prjs.append({"name": LibrawName(),
             "type": "staticlib" if staticlib else "sharedlib",
             "alias": "libraw",
             "cflags": cflags,
             "defs": defs,
             "cppflags": cppflags,
             "incdirs": [".", out_incdir],
             "srcs": srcs,
             "symvis": "default",
             "version": "%s.%s.%s" % (major, minor, patch),
             "soname": "libraw.so.%s" % (major),
             "deps": deps,
             "install": {"%s/libraw" % out_incdir: glob.glob("libraw/*.h")},
             "custom": customs})



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
                 "deps": ["libraw"] + deps,
                 "custom": [RequireLibraw] + customs})

excons.DeclareTargets(env, prjs)

Export("LibrawName LibrawPath RequireLibraw")

Default(["libraw"])

