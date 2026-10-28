# bloodborne_pc code map

- **Repo:** iannitram/bloodborne_pc (public fork). **Kind:** C loader and runtime plus a C++ GPU library (CMake and Ninja, driven by `build.sh`), Python offline tools and a GTK4 launcher: a native Linux port of one PS4 game (the project calls itself bbport).
- **Upstream:** deadinside28/bloodborne_pc (`upstream/master`). **Integration branch:** `master`.
- **Describes commit:** `0580713` (2026-10-27).

The game's original x86-64 executable is converted offline into a flat memory image with the PS4 libc linked in, and a small C loader maps it and jumps in. A purpose-written runtime replaces the PS4 system libraries, and a Vulkan renderer (its own command processor and shader translator, started from shadPS4's) draws the game, with temporal upscaling (FSR 3.1/4/4.1.1, DLSS). No game files are in the repo. This is a Parked reference mirror: upstream owns all code and docs, and the base the other PS4 ports (`ratchet-clank-pc`) build on.

## Layout

| Path | What it owns |
|---|---|
| `src/` | Loader (`probe.c`) and the HLE runtime, one file per area (`runtime_kernel.c`, `_memory`, `_thread`, `_file`, `_audio`, `_ajm`, `_ngs2`, `_pad`, `_savedata`, `_net`, `_content`, and others); `import_names*.inc` lists of imported symbols |
| `scripts/` | Offline preparation: `prepare.py`, `link_libc.py`, `link_modules.py`, patch compiler `patches.py`, `mods.py`, `game_check.py`, `content_profile.py`, `online.py`; `bbsc_gameserver/` (a local game-server helper with its own license and README) |
| `gpu/` | The renderer library `libbbgpu.so` (CMake project, `gpu/CMakeLists.txt`); see the next table |
| `launcher/` | GTK4 launcher (`bbport_launcher.py`, assets, i18n, Vulkan probing, desktop entry, `shell.nix`) |
| `packaging/` | `appimage.sh` and `bbport-entry.c` (AppImage for Steam Deck), `default.nix` (Nix package) |
| `patches/` | `Bloodborne.xml`: community game patches, compiled at start by `scripts/patches.py` |
| `tools/` | Developer tools: scripted runs and A/B toggles (`ab.sh`, `bench.sh`, `soak.sh`), FSR 4 shader rewrites (`fsr4_*`, `Fsr4SpirvCrossFixes.pm`), `fsr4cap/` (FSR 4.1.1 asset extraction), `bridge_helper/` (memory-bridge prototype), `re/` (Python analysis helpers) |
| `tests/` | About 37 files: C and C++ unit tests for the runtime, pad, online, CP decoder, upscaler pieces, shader corpus; Python `unittest` for the scripts and launcher |
| `docs/` | Design notes and measurements (`upscaler.md`, `parallel_gpu.md`, `motion_vectors.md`, `ROADMAP.md`), per-release `CHANGES_*.md`, `BBSC.md`, many notes in Russian (`*.ru.md`), `history/` |
| `third_party/LibAtrac9` | Submodule: ATRAC9 audio decoder |
| `build.sh`, `run.sh`, `shell.nix` | Build driver (also `--test`), run wrapper, Nix dev shell |
| `README.md`, `README.ru.md`, `LICENSE` | Project readme (English and Russian) and license |
| `AGENTS.fork.md`, `CHANGELOG.fork.md`, `PIPELINE.md` | Our rules for this mirror, our change history, our stage record |

### Inside `gpu/`

| Path | What it owns |
|---|---|
| `shim/` | Glue between runtime and renderer: `bbgpu.cpp`, game menu and overlay (ImGui), settings, guest-code and GNM hooks, frame-rate control, trace and diagnostics |
| `cp/` | PM4 command processor: decoder, graphics and compute paths, diagnostics |
| `gcn/`, `shader/` | GCN ISA definitions and decoder; shader translator GCN to SPIR-V (frontend, IR, backend) |
| `render/`, `vulkan/` | Pipelines, caches and state translation; Vulkan instance, command pools, descriptor heap, access tracking |
| `buffers/`, `textures/`, `memory/`, `layer/` | Buffer and texture caches, detiler and blitter; page watching and dirty tracking; the memory layer of the new memory model |
| `upscale/` | FSR 3.1/4/4.1.1, DLSS, frame generation, camera and object motion vectors, scene resolution, UI composition, shaders |
| `platform/`, `common/`, `games/` | Kernel event queue, GNM and video-out, AV player; shared utilities; per-game profile (`bloodborne.cpp`) |
| `bbnet20/`, `bbnet/` | Online module: the current one (`bbnet20`, shadNet port) and the former one, both kept; they carry shadPS4 network code |
| `dlss_bridge/`, `tools/`, `patches/` | DLSS bridge library; an FSR 4 benchmark; patches applied to the FSR-Vulkan submodule |
| `third_party/` | `sirit`, `gcn`, `half`, `httplib`, `imgui`, `fonts`, `amd-fsr1`, `fsr-vulkan` (submodule) |

## Where our changes live

Upstream owns everything. `git log upstream/master..HEAD` is three commits, five files, 55 added lines:

- `gpu/patches/fsr-vulkan/0002-...ffx_vk-never-pick-a-protected-memory-type.patch` and its entry in `gpu/patches/fsr-vulkan/README.md`: the FFX Vulkan backend never picks a protected memory type unless asked (fixes white or noisy scenes on one AMD APU, issue #85); `build.sh` applies it to the submodule.
- `AGENTS.fork.md`, `CHANGELOG.fork.md`, `PIPELINE.md`: our pipeline files.

## Build and test

```sh
bash build.sh                       # out/bb-probe and out/gpu/libbbgpu.so
bash build.sh --test                # C tests
python3 -m unittest discover -s tests
```

- Needs GCC, CMake, Ninja, Python 3, glslang, SDL3, Vulkan headers and the libraries in `shell.nix` (the script re-enters `nix-shell` when they are missing). `build.sh` also initialises submodules (`gpu/third_party/fsr-vulkan`, `gpu/third_party/imgui`, `third_party/LibAtrac9`). A PGO profile in `pgo/` is used when present (`BB_PGO`), `BB_LTO=OFF` disables LTO.
- The renderer tests are ninja targets (`ninja -C out/gpu motion-history-test ui-composition-test scene-resolution-test motion-shader-test settings-test`).
- The repo has no CI workflow.

## Mac / PC / Linux

- Linux x86-64 with a Vulkan 1.3 GPU is the only target (PIPELINE.md: platforms linux). The loader runs PS4 code natively and the build uses Linux libraries, so it does not build on macOS or Windows.
- Tested by upstream on one machine (AMD Radeon RX 7800 XT, Mesa RADV); NVIDIA and Intel are reported as tested by users.
- A Mac or Windows session can read and edit files but cannot build or run anything here (R04).
