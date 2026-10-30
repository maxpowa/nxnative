# Factorio (`01004200189F4000`)

Factorio 2.0.77 (build 84539), update v2097152, SDK 21.4.2. Wube's engine on SDL with the **NintendoSDK OpenGL driver**, which the host serves through its GL → Metal layer (`dm/gfx/gl/hle_gl.m`). The game is an unusual OpenGL port: it imports EGL and compiles GLSL at run time, so the shaders are translated by `dm/gfx/tools/gl_shaders.py` rather than the NVN shader chain. Modules `main` and `sdk`; `subsdk0` is the OpenGL driver and the host does not load it.

Brought up on an iPhone 15 (A16, 6 GB). The main menu, a new world, the intro cutscene and gameplay render.

## What it needs

| Setting | Where | Why |
|---|---|---|
| `modules main sdk` | `game.tsv` | `subsdk0` is NintendoSDK's OpenGL driver; the host's own GL layer replaces it |
| 4 GB layout: `mem_reserve 0x100000000`, `mem_alias 0x10000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | main's own pool (the log reports a ~3.1 GB guest heap) |
| `shader_groups /data/=data` | `game.tsv` | the GLSL sources under `rom:/data/core/graphics/shaders` |
| `zip_store .dat:.ogg:.png:.ttf:.zip:.metallib` | `game.tsv` | RomFS files kept stored in the IPA |
| `romfs_marker data/core/fonts/NotoSansCJKsc-Bold.ttf` | `game.tsv` | the RomFS survey marker |

No `patches.py`: the game's own builds are untouched. Everything found so far is a host-layer fix, and the profile-specific parts are gated on the build ID in the host.

## Rendering

- **GL coverage.** Factorio uses GL 4.5 desktop features: multiple render targets, uniform buffer objects, sampler objects, integer textures (`RGBA32UI`/`RG32UI`), `glCopyImageSubData`, compressed textures, texture readback and 3D textures. The GL layer implements these; 97 GLSL shaders translate and link.
- **MRT.** The 3D bitmaps render into two colour attachments; passes order attachments by `glDrawBuffers`, and per-attachment write masks are carried in the pipeline key. `glColorMaski` is per draw buffer.
- **UBOs/samplers.** Named uniform blocks get their own Metal buffer slots; `glGetUniformBlockIndex`/`glBindBufferBase`/`glUniformBlockBinding` and sampler objects are implemented.
- **Compression on A16.** A16 has no BC hardware. DXT1/3/5, RGTC1/2 and BPTC uploads are decoded to RGBA8/R8/RG8 with `bcdec` instead of asking Metal for a format it rejects. The game's compression self-test still reports PSNR failures (the decoded texture has no compressed bytes to read back), but rendering continues.
- **Factorio's copy-image dispatch slot.** The game fills `main+0x2e37260` from a weak `glCopyImageSubData` import and never asks `eglGetProcAddress` for it; the host writes its function there at the first 16³ 3D bitmap (gated on this build ID).
- **Cutscene.** `glCopyTexSubImage2D/3D` copy the read framebuffer into the bound texture; the level intro uses the 3D form.

## Services and input

- **Ramdisk.** `nn::fs::MountRamDisk` is replaced by a host mount of `<temporary storage>/ramdisk`; `GetRamDiskBufferSize` reports 64 MB. Without it, `ramdisk:/currently-playing/blueprint.dat` was ENOENT when starting a new game.
- **Software keyboard.** The save-name field uses the **UTF-8** `InlineKeyboard` callbacks (`SetChangedStringCallbackUtf8` / `SetDecidedEnterCallbackUtf8`), which the host records and answers with a `ChangedStringArgUtf8`/`DecidedEnterArgUtf8`. The host shows the iOS input box.

## Open

- `GL_TEXTURE_BUFFER` (`glBindBuffer(GL_TEXTURE_BUFFER)`, `glTexBuffer`) is not implemented. Factorio called it once during menu/level setup; the menu and gameplay continue without it.
- The character shadow appears as a checkerboard that stretches with orientation. Untriaged: the `[shadow, alpha-mask]` atlases or the missing texture-buffer path are candidates.
- The host reports the Switch's 256 MB save quota; Factorio warns that save space is low (`268435456/268435456`) even though the phone has free space. Saving has not been checked end to end.

## Building

```
scripts/build_ipa_from_nsp.py <folder with the base and update NSP> -k <keys> --allow-other-metal --env-file <env>
```

The GL shader library is generated for iOS by `dm/gfx/tools/gl_shaders.py --ios`; the profile's NVN shader tables are placeholders and unused.
