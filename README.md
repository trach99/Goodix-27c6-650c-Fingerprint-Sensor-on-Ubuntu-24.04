# Goodix 27c6:650c Fingerprint Sensor on Ubuntu 24.04

Had a lot of trouble getting the fingerprint sensor to work on my Lenovo Yoga 9 with Ubuntu 24.04.3 LTS. This guide shows how to get the **Goodix 27c6:650c fingerprint sensor** working on Ubuntu 24.04 (or derivatives) by compiling **libfprint v1.94.9** and **fprintd** from source.

> **Note:** This sensor is officially unsupported in Ubuntu’s repositories, so a compiled version of `libfprint` with Goodix support is required.

---

## Prerequisites

Install the necessary development packages:

```bash
sudo apt update
sudo apt install \
    git \
    build-essential \
    meson \
    ninja-build \
    libglib2.0-dev \
    libgusb-dev \
    libnss3-dev \
    libpam0g-dev \
    libpolkit-gobject-1-dev \
    libsystemd-dev \
    libusb-1.0-0-dev \
    libjson-glib-dev \
    libpixman-1-dev \
    pkg-config
```

> These are required to compile libfprint and fprintd, including Goodix driver support.

---

## Step 1: Remove conflicting system libraries

Remove old Ubuntu libfprint packages to avoid conflicts:

```bash
sudo apt remove libfprint-2-2 libfprint-2-tod1-goodix libfprint-2-tod1:amd64
sudo ldconfig
```

---

## Step 2: Build and install libfprint v1.94.9

1. Clone the libfprint repository:

```bash
cd ~
git clone https://gitlab.freedesktop.org/libfprint/libfprint.git
cd libfprint
git checkout v1.94.9
```

2. Clean any old build directory:

```bash
rm -rf builddir
```

3. Configure Meson with Goodix driver:

```bash
meson setup builddir --prefix=/usr/local -Ddrivers=goodixmoc
```

4. Compile and install:

```bash
ninja -C builddir
sudo ninja -C builddir install
sudo ldconfig
```

5. Verify installation:

```bash
pkg-config --modversion libfprint-2
# Should output 1.94.9
```

> The `goodixmoc` driver enables partial support for Goodix sensors like 27c6:650c.  
> Installing to `/usr/local` ensures the new library does not conflict with system libraries.

---

### Optional: Create a pkg-config file (if missing)

If `libfprint-2.pc` does not exist:

```bash
sudo mkdir -p /usr/local/lib/pkgconfig
sudo tee /usr/local/lib/pkgconfig/libfprint-2.pc <<EOF
prefix=/usr/local
exec_prefix=\${prefix}
libdir=\${exec_prefix}/lib
includedir=\${prefix}/include

Name: libfprint-2
Description: Fingerprint library
Version: 1.94.9
Libs: -L\${libdir} -lfprint-2
Cflags: -I\${includedir}/libfprint-2
EOF
```

Verify:

```bash
pkg-config --modversion libfprint-2
# Should now correctly return 1.94.9
```

---

## Step 3: Build and install fprintd

1. Clone fprintd:

```bash
cd ~
git clone https://gitlab.freedesktop.org/freedesktop/fprint/fprintd.git
cd fprintd
```

2. Clean previous build:

```bash
rm -rf builddir
```

3. Configure and compile:

```bash
meson setup builddir --prefix=/usr
ninja -C builddir
sudo ninja -C builddir install
```

4. Reload systemd and start the daemon:

```bash
sudo systemctl daemon-reload
sudo systemctl restart fprintd
systemctl status fprintd
```

---

## Step 4: Enroll and test fingerprints

```bash
fprintd-enroll $USER
```

- Follow the prompts to enroll one or more fingerprints.  
- Test verification:

```bash
fprintd-verify $USER
```

---

## Notes

- **System integration:** Once enrolled, fingerprints can be used for login, sudo, and screen unlock (if PAM is configured).  
- **Unsupported device:** Even with this build, some features may not work perfectly because 27c6:650c is officially unsupported.  
- **Future updates:** Recompiling libfprint/fprintd is required when upgrading or replacing these libraries.

---
