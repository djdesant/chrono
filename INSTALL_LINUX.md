# Chrono Build Guide on Linux

This document describes how to build and run Project Chrono with the Irrlicht visualization and Synchrono modules on a modern Linux system.
It includes fixes for FlatBuffers, OpenMP linking, Python bindings, and remote GUI display through SSH.

---

## 1. System Requirements

Recommended environment:

* Ubuntu 18.04 or newer
* GCC 9 to 11 recommended
* Clang up to 15 supported, Clang 20+ requires FlatBuffers patch
* CMake 3.20 or newer
* Ninja build system
* SWIG 4.x for Python bindings
* OpenMPI
* HDF5
* Irrlicht 1.8.x

---

## 2. Install Dependencies

```bash
sudo apt update
sudo apt install -y \
  build-essential cmake ninja-build \
  gcc g++ \
  libopenmpi-dev openmpi-bin \
  libhdf5-serial-dev \
  libirrlicht-dev \
  libeigen3-dev \
  python3-dev python3-numpy \
  swig \
  xauth x11-apps
```

---

## 3. Recommended Compiler Configuration

The most stable configuration is GCC with system OpenMP.

Use in CMake preset:

```json
"CMAKE_C_COMPILER": "/usr/bin/gcc",
"CMAKE_CXX_COMPILER": "/usr/bin/g++",
"CMAKE_CXX_FLAGS": "-march=native -fopenmp",
"CMAKE_C_FLAGS": "-march=native -fopenmp"
```

### Using Clang 22 or newer

FlatBuffers 1.12 bundled with Chrono is not fully compatible with modern Clang.
Apply this patch to the submodule:

File: src/chrono_thirdparty/flatbuffers/include/flatbuffers/stl_emulation.h

```diff
-  const size_type count_;
+  size_type count_;
```

Rebuild flatc and regenerate all schemas afterwards.

cd src/chrono_thirdparty/flatbuffers
patch -p1 < ../../fix_flatbuffers_span.patch

Rebuild FlatBuffers Compiler (flatc)
```
cd src/chrono_thirdparty/flatbuffers
rm -rf build
mkdir build && cd build
```
```
cmake ..
make -j$(nproc)
```

Verify the rebuilt compiler:
```
./flatc --version
```

Regenerate Synchrono Messages

After rebuilding flatc you must regenerate all generated headers:
```
cd ../../chrono_synchrono/flatbuffer

../../chrono_thirdparty/flatbuffers/build/flatc \
  --cpp \
  --gen-mutable \
  --gen-object-api \
  --scoped-enums \
  -I fbs \
  -o message \
  fbs/*.fbs
```
Check that files were updated:
```
ls message/Syn*Message.h
```
---

## 4. Configure Chrono

Create a preset named linux-vm:

```json
{
  "name": "linux-vm",
  "binaryDir": "${sourceDir}/build-linux",
  "generator": "Ninja",

  "cacheVariables": {

    "CMAKE_BUILD_TYPE": "Release",

    "CMAKE_C_COMPILER": "/usr/bin/gcc",
    "CMAKE_CXX_COMPILER": "/usr/bin/g++",

    "CMAKE_CXX_STANDARD": "17",

    "CMAKE_CXX_FLAGS": "-march=native -fopenmp",
    "CMAKE_C_FLAGS": "-march=native -fopenmp",

    "SWIG_EXECUTABLE": "/usr/local/bin/swig",

    "CH_ENABLE_HDF5": "ON",
    "CH_ENABLE_MODULE_IRRLICHT": "ON",
    "CH_ENABLE_MODULE_SYNCHRONO": "ON",
    "CH_ENABLE_MODULE_VEHICLE": "ON",
    "CH_ENABLE_MODULE_PYTHON": "ON",

    "IRRLICHT_LIBRARY": "/usr/local/lib/libIrrlicht.so",
    "IRRLICHT_INCLUDE_DIR": "/usr/local/include/irrlicht",

    "EIGEN3_INCLUDE_DIR": "$env{HOME}/github/eigen",
    "THRUST_INCLUDE_DIR": "$env{HOME}/github/thrust",

    "MKL_DIR": "/opt/intel/oneapi/mkl/latest/lib/cmake/mkl"
  }
}
```

---

## 5. Build Chrono

```bash
cmake --preset linux-vm
cmake --build build-linux -j$(nproc)
```

---

## 6. Install to Local Directory

To avoid root permissions:

```bash
cmake -B build-linux \
  -DCMAKE_INSTALL_PREFIX=$HOME/github/chrono/chrono-install

cmake --build build-linux --target install
```

---

## 7. FlatBuffers Generation for Synchrono

After patching or rebuilding flatc:

```bash
cd src/chrono_synchrono/flatbuffer

../../chrono_thirdparty/flatbuffers/build/flatc \
  --cpp --gen-mutable --gen-object-api --scoped-enums \
  -I fbs -o message fbs/*.fbs
```

---

## 8. OpenMP Linking Fix

If linking fails with symbol __kmpc_dispatch_deinit:

Link against Intel OpenMP explicitly:

```bash
-L/opt/intel/oneapi/compiler/latest/lib -liomp5
```

Test:

```bash
nm -D lib/libChrono_core.so | grep kmpc_dispatch_deinit
nm -D /opt/intel/oneapi/compiler/latest/lib/libiomp5.so | grep kmpc_dispatch_deinit
```

---

## 9. Python Bindings

Ensure SWIG 4.x is used:

```bash
swig -version
```

If multiple versions exist:

```bash
sudo update-alternatives --install /usr/bin/swig swig /usr/local/bin/swig 100
```

Set Python path:

```bash
export PYTHONPATH=$HOME/github/chrono/build-linux/lib:$PYTHONPATH
```

---

## 10. Running GUI over SSH

### From Windows Client

Use X forwarding:

```cmd
ssh -Y user@linux-ip
```

On Linux server:

```bash
sudo apt install xauth x11-apps
echo $DISPLAY
xeyes
```

If DISPLAY is empty, add to sshd_config:

```
X11Forwarding yes
X11UseLocalhost yes
```

Restart:

```bash
sudo systemctl restart ssh
```

---

## 11. Running Demo

```bash
cd build-linux
./bin/demo_ROBOT_Curiosity_Rigid
```

---

## 12. Troubleshooting

### FlatBuffers Errors

* Use flatc 1.12 matching Chrono
* Do not mix system flatc with bundled headers
* Regenerate all generated headers after any change

### OpenMP Conflicts

* Prefer GCC with system libgomp
* For Clang, ensure same OpenMP library at build and link time

### Python Uses Wrong Version

CMake selects first Python found.
Install desired version and reconfigure with:

```bash
-DPython3_EXECUTABLE=/usr/bin/python3.9
```

---

## 13. Clean Rebuild

```bash
rm -rf build-linux
cmake --preset linux-vm
cmake --build build-linux -j$(nproc)
```

---

## 14. Support Notes

* GCC is the most reliable toolchain
* Clang 20+ requires FlatBuffers patch
* Do not use flatc 25 with Chrono 9
* X forwarding is required for Irrlicht GUI over SSH

---

End of guide.
