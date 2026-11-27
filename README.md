# PyAV video timing build

This wheel is PyAV 19.0.2 with a small native source patch. It is for CPython 3.12 and newer,
64-bit x86 Windows. Python 3.8 / PySide2 keeps its existing PyAV 12.3.0 requirement;
other platforms keep the normal optional PyAV package. No application API or decoded format changed.

## Why it exists

A Python worker does not protect presentation from a native call that holds the GIL.
On the author's machine, a codec destructor blocked the GUI and other Python threads for about
22 ms. Inspecting an incoming format on the GUI separately caused a 32 ms preparation pause;
that application bug is fixed by asynchronous metadata inspection, not by this wheel.

PyAV upstream fixed codec teardown in commit
[482efead0d823a0295011e9bcc1d481a53c3cdc2](https://github.com/PyAV-Org/PyAV/commit/482efead0d823a0295011e9bcc1d481a53c3cdc2).
The installed public 19.0.2 wheel predates that commit. `video_gil.patch` backports that fix and
locally releases the GIL around `avcodec_open2` and the two `av_hwdevice_ctx_create` calls too.
Python argument preparation and error handling remain under the GIL; only the native operations
run without it. SimpleWorship gives each decoder to one worker and closes it after decoding ends.

Release `.2` also drops the hardware device reference of `HWDevice` and `HWAccel` (`av_buffer_unref` in
their `__dealloc__`) outside the GIL. The codec context gives its own reference back without the GIL, but the
`HWAccel` object holds the last one: freeing a D3D11VA decoder destroyed the device with the GIL held for
22-25 ms on the author's machine, and a frame was lost at the end of every fade from a hardware-decoded
video. With `.2` the teardown still takes about 22 ms, but other Python threads run meanwhile (about 1 ms
of starvation, measured with a ticker thread).

The wheel contains the same FFmpeg library versions as the original 19.0.2 installation:
avcodec/avformat/avdevice 63.1.102, avutil 61.1.102, avfilter 12.1.102,
swscale 10.1.102 and swresample 7.1.102. The local version identifies the patched build;
it does not claim to be an official PyAV release. PyAV's license is included in the wheel.

## Rebuild

Use a separate build environment with the Windows SDK and Visual C++ build tools installed.
The tested compiler was MSVC 14.44.35207. The official upstream build procedure is in
[PyAV's release workflow](https://github.com/PyAV-Org/PyAV/blob/v19.0.2/.github/workflows/tests.yml).

1. Download and extract [the v19.0.2 source archive](https://github.com/PyAV-Org/PyAV/archive/refs/tags/v19.0.2.zip).
1. In that extracted source directory, apply this folder's `video_gil.patch` with `git apply`.
1. Install Cython 3.3.0, setuptools 84.2.1, wheel 0.48.0 and delvewheel 1.13.2 in the build environment.
1. Fetch the matching official FFmpeg development bundle using the source's own script:

```text
python scripts/fetch-vendor.py --config-file scripts/ffmpeg-latest.json ../vendor
python setup.py --ffmpeg-dir=../vendor bdist_wheel
python -m delvewheel repair --add-path ../vendor/bin --wheel-dir ../wheelhouse dist/av-19.0.2+simpleworship.2-cp312-abi3-win_amd64.whl
```

The patch uses the standard LF unified-diff format (`git apply` refuses a CRLF copy of it against the LF source). Application Python, documentation and requirements retain CRLF.
Generated wheel timestamps need not be identical between rebuilds; the shipped artifacts are identified below.

| Artifact | SHA-256 |
| --- | --- |
| Original source ZIP | c5c46e5b8ec57b26990be7a42a7f41ed1c0c6263b672b2edbe73d916153a0e16 |
| Official FFmpeg bundle `9.0.2-1/ffmpeg-windows-x86_64.tar.gz` | add88a2e31f63be16f4ba671862770f7f9313de2f8d2967b1eade4c18bf4b093 |
| Shipped patched wheel `.2` | 96ea327dc6a5960df885e43338edd34f81d9a557ccce5c6976d5b8190d900f21 |
| Earlier wheel `.1` (release `v19.0.2-simpleworship.1`) | 640c4dddef3b001e54b27b8aa9d4d2a5eca74dc482f055544a56a710a6cb1971 |

## Install and retire the patch

Run `python -m pip install -r requirements-pyside6.txt` from the project root, or use
`tools/setup_env.py` as usual. A process already running must be restarted to load the new native library.
The wheel is self-contained; end users do not need Visual Studio, Cython or the CUDA toolkit.

When an official release includes these timing fixes, verify it with the same CPU/GPU transition
and continuous-playback measurements, then remove the local wheel and simplify the modern requirement.
Do not silently replace the legacy Python 3.8 dependency with this Python 3.12+ build.
