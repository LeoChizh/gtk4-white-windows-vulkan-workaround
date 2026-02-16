Absolutely. Here is the translated and polished version of your post, ready for GitHub.

---

# Fixing GTK4 White Windows in VirtualBox (Ubuntu 24.04+) Without Falling Back to OpenGL

## The Problem

If you install Ubuntu 24.04 (or newer) in VirtualBox with **3D acceleration enabled**, you've likely encountered this issue: GTK4 applications—like **Settings**, **Files (Nautilus)**, **Text Editor**, and parts of GNOME Shell—render as blank, white windows .

The root cause is well-documented. Starting with GTK4 version 4.14, the default renderer was switched from the old OpenGL backend (`gl`) to a new one called `ngl` . This new backend does not play nicely with the virtualized graphics drivers (VMware SVGA II / VBoxSVGA) used by VirtualBox when 3D acceleration is active.

## The "Standard" Fix (And Why It's a Compromise)

The common advice found everywhere—from Ubuntu forums to official bug trackers—is to force GTK4 back to the old OpenGL renderer .

```bash
echo 'export GSK_RENDERER=gl' >> ~/.profile
# Or add it to /etc/environment for a system-wide effect
```

**This works.** The white windows go away. But at what cost? You're effectively disabling the modern, more efficient rendering pipeline of GTK4 and forcing it to use an older OpenGL path. It's a workaround, not a fix.

## A Different Approach: The Vulkan Path

I experimented with a different hypothesis. If the problem is the `ngl` (OpenGL) backend, why not try the **Vulkan renderer**?

The challenge: VirtualBox doesn't pass through physical Vulkan capabilities to the guest. However, it doesn't need to. Modern Mesa includes **`llvmpipe`**—a software rasterizer that can render Vulkan commands using your CPU.

The theory: If we can provide GTK4 with a fully functional Vulkan implementation (even a software one), it should use its Vulkan renderer successfully, bypassing the broken OpenGL path while keeping 3D acceleration enabled for other tasks.

The result? **It works.**

## The Solution: Build a Modern Vulkan Driver Stack

The version of Mesa in the Ubuntu repositories is either too old or compiled without the necessary patches for this to work seamlessly. The solution is to build the latest **Vulkan SDK from LunarG**, which bundles a modern `llvmpipe` that handles GTK4's Vulkan surface requests correctly.

**Warning:** This takes over an hour and requires significant disk space and compilation power. This is a deep-dive fix.

### Step-by-Step

**1. Install build dependencies and DRI3 components**

```bash
sudo apt install libxcb-dri3-dev libxcb-present0 libxcb-randr0-dev \
    libxcb-keysyms1-dev libxcb-ewmh-dev libxcb-glx0-dev \
    libxcb-xfixes0-dev libxcb-shape0-dev libxcb-sync-dev \
    git cmake ninja-build python-is-python3 mesa-vulkan-drivers
```

**2. Download and build the LunarG Vulkan SDK**

```bash
wget https://sdk.lunarg.com/sdk/download/latest/linux/vulkan-sdk.tar.xz
tar -xf vulkan-sdk.tar.xz
cd ./1.4.*  # Version number will vary
./vulkansdk  # This takes 60+ minutes. Go get coffee. Or a meal.
```

**3. Source the SDK environment automatically**

```bash
echo 'source ~/vulkansdk-linux-x86_64-1.4.341.1/1.4.341.1/setup-env.sh' >> ~/.bashrc
```
*(Adjust the path to match your downloaded version)*

**4. Force GTK4 to use the Vulkan renderer**

```bash
mkdir -p ~/.config/environment.d/
echo 'GSK_RENDERER=vulkan' >> ~/.config/environment.d/99-gtk-renderer.conf
```

**5. Reboot**

```bash
sudo reboot
```

## Why This Works

- **VirtualBox breaks OpenGL passthrough**, causing GTK4's `ngl` backend to fail .
- **VirtualBox does not break Vulkan**—because it doesn't attempt hardware passthrough for Vulkan at all.
- By providing a **modern software Vulkan implementation (`llvmpipe`)** via the LunarG SDK, GTK4's Vulkan renderer has a functional path.
- The result: GTK4 renders correctly, **3D acceleration in VirtualBox remains enabled** for other 3D applications, and you haven't sacrificed the modern rendering architecture.

## Caveats

- This is a **software rendering solution**. While it fixes the white windows, GTK4 applications are now rendering on the CPU via `llvmpipe`. Performance will be different (likely slower) compared to hardware-accelerated OpenGL.
- The LunarG SDK build is **heavy and time-consuming**.
- This is an **experimental workaround**. It's not an official fix, and future updates to GTK4, Mesa, or VirtualBox may change the behavior.

## Conclusion

The standard advice (`GSK_RENDERER=gl`) is a quick fix, but it rolls back to older technology. This approach demonstrates that it's possible to get GTK4 rendering correctly in a virtualized environment without disabling its modern rendering backend—by giving it a functional Vulkan target.

Has anyone else attempted a similar fix? I'd be interested to hear about other experiments or refinements to this method.

---

**Tags:** virtualbox, ubuntu, gtk4, vulkan, mesa, llvmpipe, linux