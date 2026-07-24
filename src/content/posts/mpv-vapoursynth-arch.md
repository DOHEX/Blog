---
title: 在 Arch Linux 上配置 mpv：迁移 PlayKit 与 mpv-config
published: 2026-07-23
description: 从 Windows 版 mpv PlayKit 和 dyphire/mpv-config 出发，在 Arch Linux 上配置 mpv、VapourSynth、GLSL 着色器与 TensorRT 补帧。
tags: [mpv, VapourSynth, Arch Linux, NVIDIA, TensorRT, Anime4K, 动画补帧]
category: Linux
licenseName: "CC BY 4.0"
draft: false
lang: zh-CN
---

这是一篇面向 Arch/NVIDIA 用户的迁移教程。假设你已经下载了面向 Windows 的 [mpv PlayKit](https://github.com/hooke007/mpv_PlayKit)（原 mpv-lazy）以及 [dyphire/mpv-config](https://github.com/dyphire/mpv-config)，希望在 Arch/CachyOS 上保留它们的 mpv 配置、uosc 菜单、GLSL 着色器和 VapourSynth 补帧/超分脚本。

本教程的目标是在 Linux 上还原这两套配置的绝大部分体验。

本文验证环境：mpv 0.41、VapourSynth R77、Python 3.14、CUDA 13.3、TensorRT 11.1 和 RTX 4060 Laptop GPU。

# 完成后目录应该是什么样

~~~text
系统包 / AUR
  mpv、VapourSynth core、CUDA、TensorRT、MVTools、MLRT TensorRT runtime

~/.config/mpv/
  mpv.conf、profiles.conf、input_uosc.conf
  shaders/                   GLSL 着色器
  scripts/、script-opts/     Lua 脚本和配置
  vs/                        RIFE、DRBA、MVTools 等 .vpy

~/.local/opt/mpv-vs/
  venv/                      k7sfunc、Akarin、ONNX 转换库
  xdg/vapoursynth/           只供包装器使用的 vapoursynth.toml

~/.local/bin/mpv-vs          使用私有 venv 启动 mpv 的包装器
~~~

带 ABI 的组件，例如 NVIDIA 驱动、CUDA、TensorRT、VapourSynth core 和原生插件，交给 pacman/AUR；着色器、Lua、.vpy 和普通配置放入 mpv 配置目录；只给 .vpy 使用的 Python 包放入 venv。

# 按顺序迁移

## 1. 先安装系统组件

使用 `nvidia-smi` 确认 NVIDIA 驱动已经正常工作。

安装以下软件包：

~~~fish
sudo pacman -S --needed mpv vapoursynth cuda vapoursynth-plugin-mvtools
paru -S --needed tensorrt vapoursynth-plugin-mlrt-trt-runtime-git
~~~

各包的作用：

| 包 | 用途 |
| --- | --- |
| mpv | 播放器和 VapourSynth 视频滤镜宿主 |
| vapoursynth | VS core、Python 绑定与 VSScript 原生库 |
| vapoursynth-plugin-mvtools | 传统运动估计与 MVTools 补帧 |
| cuda、tensorrt | CUDA 工具链和 TensorRT 推理运行时 |
| vapoursynth-plugin-mlrt-trt-runtime-git | 让 VapourSynth 将 ONNX 模型交给 TensorRT 的 core.trt 插件 |

确认发行版的 mpv 确实链接了 VapourSynth：

~~~fish
ldd /usr/bin/mpv | rg vapoursynth
~~~

预期有类似一行：

~~~text
libvapoursynth-script.so.0 => /usr/lib/libvapoursynth-script.so.0
~~~

## 2. 从两个项目复制配置

你可以将 mpv PlayKit 的 `portable_config` 目录中的内容，复制到 `~/.config/mpv/` 中，也可以自行组合你需要的配置。

:::note
Windows专有内容如.dll等文件无需复制。（我去不早说）
:::

打开 mpv.conf，修改 Windows 专属的输出后端；使用 PipeWire：

~~~ini
# 删除或注释 Windows 选项，例如：
# ao=wasapi
# gpu-api=d3d11
# gpu-context=d3d11

ao=pipewire
~~~

## 3. 清理 Windows/AMD 专用 .vpy

在 ~/.config/mpv/vs 中删除：

- *_DML.vpy：DirectML，只能在 Windows 上用。
- *_MIGX.vpy：MIGraphX，面向 AMD ROCm。

## 4. 创建 mpv 专用 venv

Arch 遵守 PEP 668，不建议用 `pip --break-system-packages` 将 k7sfunc 等包写进系统或用户全局 Python。

这里特意使用 `--system-site-packages` ：venv 只读取 pacman 提供的 VapourSynth Python 绑定和系统原生插件，不会向系统 Python 写入；k7sfunc、Akarin 和转换工具仍只安装在私有 venv。如果不加这个选项，venv 看不到系统安装的 vapoursynth 模块。

~~~fish
set ROOT ~/.local/opt/mpv-vs
set VENV $ROOT/venv

python3 -m venv --system-site-packages $VENV
$VENV/bin/python -m pip install --upgrade pip
$VENV/bin/python -m pip install --upgrade k7sfunc vapoursynth-akarin onnxconverter-common
~~~

确认系统插件能从该 venv 被导入：

~~~fish
$VENV/bin/python -c 'import vapoursynth as vs; print(hasattr(vs.core, "trt"), hasattr(vs.core, "mv"))'
~~~

预期输出为 True True。Akarin 需要额外插件路径，下一步配置包装器后再验证。

## 5. 配置 R77 VSScript，使 mpv 使用 venv Python

R77 使用 vapoursynth.toml 告诉 VSScript：某个入口库执行 .vpy 时，该嵌入哪个 Python 解释器和 libpython。

mpv 链接的是系统级 libvapoursynth-script.so.0，而 vapoursynth config 通常只登记 Python 包内的 libvsscript.so。它们不是两套 VS core，但 TOML 的映射 key 不同；两行都需要。

~~~fish
mkdir -p $ROOT/xdg/vapoursynth
~~~

创建 $ROOT/xdg/vapoursynth/vapoursynth.toml。下面的 Python 小版本和用户名替换为实际值：

~~~toml
"/usr/lib/python3.14/site-packages/vapoursynth/libvsscript.so" = ["/home/用户名/.local/opt/mpv-vs/venv/bin/python","/usr/lib/libpython3.14.so.1.0"]
"/usr/lib/libvapoursynth-script.so.0" = ["/home/用户名/.local/opt/mpv-vs/venv/bin/python","/usr/lib/libpython3.14.so.1.0"]
~~~

如果遗漏第二行，mpv 常见错误是：

~~~text
could not initialize vapoursynth scripting
~~~

## 6. 创建 mpv-vs 包装器

创建 ~/.local/bin/mpv-vs：

~~~sh
#!/bin/sh
set -eu

ROOT="$HOME/.local/opt/mpv-vs"

export XDG_CONFIG_HOME="$ROOT/xdg"
export VAPOURSYNTH_EXTRA_PLUGIN_PATH="$ROOT/venv/lib/python3.14/site-packages/vapoursynth/plugins"

exec /usr/bin/mpv --config-dir="$HOME/.config/mpv" "$@"
~~~

~~~fish
chmod +x ~/.local/bin/mpv-vs
~~~

| 设置 | 作用 |
| --- | --- |
| XDG_CONFIG_HOME | 让 VSScript 读取私有 TOML，并嵌入 venv 的 Python。 |
| --config-dir | 明确让 mpv 仍读取原有 mpv.conf、脚本、着色器和 .vpy。 |
| VAPOURSYNTH_EXTRA_PLUGIN_PATH | 额外加载 venv 内 pip 安装的原生插件；本例中是 Akarin。 |

验证 Akarin：

~~~fish
env VAPOURSYNTH_EXTRA_PLUGIN_PATH="$ROOT/venv/lib/python3.14/site-packages/vapoursynth/plugins" \
  $VENV/bin/python -c 'import vapoursynth as vs; print(hasattr(vs.core, "akarin"))'
~~~

应输出 True。

## 7. 迁移 PlayKit 的 ONNX 模型

将 mpv PlayKit 的模型文件：

`/path/to/mpv_PlayKit/vs-plugins/models`

复制进：

`/usr/lib/python3.14/site-packages/vapoursynth/plugins/models`


不要复制 .engine、.engine.onnx、.cache 或 .lock。它们由当前 GPU、TensorRT 与系统环境生成，无法跨 Windows/Linux 复用。第一次开启一个模型或新的分辨率时需要等待 TensorRT 建 engine 是正常现象。

## 8. 让桌面启动的 mpv 也经过包装器

为了让桌面启动器启动的 mpv 也使用 venv，复制系统 desktop 文件到用户目录：

~~~fish
cp /usr/share/applications/mpv.desktop ~/.local/share/applications/mpv.desktop
~~~

将其中两项改为：

~~~ini
TryExec=/home/用户名/.local/bin/mpv-vs
Exec=/home/用户名/.local/bin/mpv-vs --player-operation-mode=pseudo-gui -- %U
~~~

并刷新缓存：
~~~fish
update-desktop-database ~/.local/share/applications
~~~

# 常见问题

## TensorRT 11 与 k7sfunc 1.8.1

PlayKit 当前的 k7sfunc/VSMLRT 调用主要按 TensorRT 10 设计，可能传递 --fp16 及旧的 layer-precision 参数。TensorRT 11 默认使用 strongly typed network：ONNX 图必须显式携带精度，旧命令行开关不再适用。因此会看到：

~~~text
Unknown option: --fp16
~~~

NVIDIA 的 ModelOpt AutoCast 是将 FP32 ONNX 转为混合精度 ONNX 的官方工具之一，但不是唯一方案。我的 TensorRT 11 兼容补丁采用较小的改动：预先转换模型到 FP16、为 DRBA/RIFE 的坐标路径保留 FP32，并只在 TensorRT 10 及以下传递旧参数。该补丁只验证了 k7sfunc 1.8.1、DRBA v2 lite AP 和 RIFE v4.6；INT8/FP8 与其他模型不在保证范围内。

你可以使用 AI 编程工具自行修改。

## uosc 的着色器预设报“找不到文件”

PlayKit 的 saved-glsl-list.json 同时保存 list 与 str：list 是路径数组，str 是用分号拼接的文本。原脚本恢复预设时使用 str：

~~~lua
mp.commandv("change-list", "glsl-shaders", "set", presets[index].str)
~~~

Linux 上的 mpv 会将 a.glsl;b.glsl 视为一个文件路径。修改 ~/.config/mpv/scripts/uosc_addones/menu_shader.lua，改为直接设置数组：

~~~lua
mp.set_property_native("glsl-shaders", presets[index].list)
~~~
