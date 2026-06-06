# Final1: 3D Reconstruction, Text-to-3D, Image-to-3D and Scene Fusion

This repository contains the source code and reproducibility instructions for the course project:

- **Object A:** real-object multi-view reconstruction with COLMAP + 2D Gaussian Splatting (2DGS).
- **Object B:** text-to-3D generation of a smartphone with `threestudio` + Stable Diffusion SDS loss.
- **Object C:** single-image-to-3D generation of a thermos with a Magic123-style pipeline.
- **Background and fusion:** counter-style background reconstruction/proxy and multi-view scene rendering with Objects A/B/C inserted into the same environment.

Project links:

- GitHub repository: <https://github.com/wangsichen021-beep/final1>
- Model weights and large assets: <https://drive.google.com/drive/folders/1P-9ezcaRHK4eCabDh-lK62IOW6r8VMb1?usp=sharing>

The GitHub repository is intentionally source-only. Large files such as `.ply`, `.ckpt`, `.mp4`, training outputs, and packaged results should be downloaded from the Google Drive link above.

## Repository Structure

```text
.
+-- environment.yml
+-- objectA/
|   +-- code/                  # frame extraction, COLMAP, mesh export, visualization
|   +-- env/                   # recorded pip-freeze files from the completed run
+-- objectB/
|   +-- code/                  # SDS export scripts and low-VRAM threestudio patches
|   +-- run_configs/           # final text-to-3D config and command log
+-- objectC/
|   +-- code/                  # image preparation, Magic123 export, final mesh script
|   +-- run_configs/           # final single-image-to-3D config and command log
+-- scene_fusion/
    +-- code/                  # background proxy and fusion rendering scripts
    +-- run_configs/           # full Mip-NeRF 360 / counter command templates
    +-- run_counter_proxy_2dgs.ps1
    +-- run_scene_fusion.ps1
```

## Requirements

The project was completed on Windows with PowerShell, CUDA PyTorch, and an NVIDIA RTX 3050 Laptop GPU. The same code can be adapted to Linux by replacing PowerShell commands with equivalent shell commands.

Required system tools:

- Windows 10/11 or Linux
- NVIDIA GPU with CUDA support
- Conda or Mamba
- Git
- COLMAP or `pycolmap`
- FFmpeg, optional but recommended for video export

External repositories expected in the project root:

```text
final1/
+-- 2d-gaussian-splatting/     # official 2DGS implementation
+-- threestudio/               # official threestudio implementation
+-- ...
```

Recommended setup:

```powershell
conda env create -f environment.yml
conda activate final1-3d
python -c "import torch; print(torch.__version__, torch.cuda.is_available())"
```

If the original 2DGS CUDA extensions or the original `threestudio` CUDA extensions fail to compile on a low-VRAM Windows machine, use the included pure-PyTorch fallback files under:

```text
objectB/code/modified_threestudio_files/
objectC/code/modified_threestudio_files/
```

## External Code Setup

Clone external frameworks into the repository root:

```powershell
git clone https://github.com/hbb1/2d-gaussian-splatting.git 2d-gaussian-splatting
git clone https://github.com/threestudio-project/threestudio.git threestudio
```

Copy the final configs into `threestudio/configs/`:

```powershell
Copy-Item objectB\run_configs\objectB-phone-sds-final.yaml threestudio\configs\objectB-phone-sds-final.yaml -Force
Copy-Item objectC\run_configs\objectC-thermos-magic123-lowvram.yaml threestudio\configs\objectC-thermos-magic123-lowvram.yaml -Force
```

For the low-VRAM Windows run used in this project, also apply the patched files in `objectB/code/modified_threestudio_files/` and `objectC/code/modified_threestudio_files/` to the corresponding `threestudio` modules before training. These patches replace the unavailable CUDA-heavy renderer path with a lightweight PyTorch renderer while keeping the SDS/Magic123 training logic.

```powershell
# Common low-VRAM patches used by the Object B SDS run
Copy-Item objectB\code\modified_threestudio_files\exporters_init.py  threestudio\threestudio\models\exporters\__init__.py -Force
Copy-Item objectB\code\modified_threestudio_files\guidance_init.py   threestudio\threestudio\models\guidance\__init__.py -Force
Copy-Item objectB\code\modified_threestudio_files\implicit_volume.py threestudio\threestudio\models\geometry\implicit_volume.py -Force
Copy-Item objectB\code\modified_threestudio_files\materials_init.py  threestudio\threestudio\models\materials\__init__.py -Force
Copy-Item objectB\code\modified_threestudio_files\networks.py        threestudio\threestudio\models\networks.py -Force
Copy-Item objectB\code\modified_threestudio_files\ops.py             threestudio\threestudio\utils\ops.py -Force
Copy-Item objectB\code\modified_threestudio_files\misc.py            threestudio\threestudio\utils\misc.py -Force
Copy-Item objectB\code\modified_threestudio_files\renderers_init.py  threestudio\threestudio\models\renderers\__init__.py -Force
Copy-Item objectB\code\modified_threestudio_files\uncond_eff.py      threestudio\threestudio\data\uncond_eff.py -Force

# Extra patches used by the Object C Magic123-style run
Copy-Item objectC\code\modified_threestudio_files\simple_volume_renderer.py             threestudio\threestudio\models\renderers\simple_volume_renderer.py -Force
Copy-Item objectC\code\modified_threestudio_files\stable_diffusion_prompt_processor.py  threestudio\threestudio\models\prompt_processors\stable_diffusion_prompt_processor.py -Force
```

## Data Preparation

Create the data and output folders:

```powershell
New-Item -ItemType Directory -Force data\raw, data\objectA, data\objectC, output | Out-Null
```

### Object A: Real Multi-View Input

Place the phone-captured orbit video at:

```text
data/raw/objectA.mp4
```

Extract sharp frames:

```powershell
python objectA\code\prepare_objectA_frames.py `
  --video data\raw\objectA.mp4 `
  --dataset data\objectA `
  --frames 72 `
  --max-side 960
```

Run COLMAP / pycolmap pose recovery:

```powershell
python objectA\code\run_pycolmap_objectA.py `
  --dataset data\objectA `
  --camera-model SIMPLE_RADIAL `
  --use-gpu `
  --reset
```

After this step, `data/objectA/` should contain COLMAP-compatible images and `sparse/0`.

### Object B: Text Prompt

Object B uses only a text prompt. The final prompt is encoded in:

```text
objectB/run_configs/objectB-phone-sds-final.yaml
```

Prompt summary:

```text
a realistic 3D model of a slim modern smartphone, glossy black glass front screen,
silver aluminum frame, rounded corners, back side has two large camera lenses
and a small flash, high detail
```

### Object C: Single Clean Foreground Image

Place the clean thermos image at:

```text
data/raw/thermos.png
```

Prepare RGBA foreground and mask:

```powershell
python objectC\code\prepare_objectC_image.py `
  --input data\raw\thermos.png `
  --output data\objectC\thermos_rgba.png `
  --preview data\objectC\thermos_rgba_preview.png `
  --mask data\objectC\thermos_mask.png `
  --rect 0.285,0.085,0.43,0.84 `
  --iterations 6
```

### Pretrained Weights and Final Assets

Download large assets from Google Drive:

```text
https://drive.google.com/drive/folders/1P-9ezcaRHK4eCabDh-lK62IOW6r8VMb1?usp=sharing
```

Recommended local layout after download:

```text
model_weights_submission/
├── objectA_2dgs/
├── objectB_sds/
├── objectC_magic123/
└── background_2dgs/

scene_fusion/sources/objects/
├── objectA_largest_100k_best.ply
├── objectB_final_phone_best.ply
└── objectC_final_thermos_best.ply
```

The `scene_fusion/sources/objects/` files are needed for the final fused scene rendering command.

## Train Commands

Run commands from the repository root unless stated otherwise.

### Train Object A: COLMAP + 2DGS

```powershell
python objectA\code\run_pycolmap_objectA.py `
  --dataset data\objectA `
  --camera-model SIMPLE_RADIAL `
  --use-gpu `
  --reset

Push-Location 2d-gaussian-splatting
python train.py `
  -s ..\data\objectA `
  -m ..\output\objectA_2dgs `
  --iterations 10000 `
  --resolution 1 `
  --test_iterations 1000 5000 10000 `
  --save_iterations 10000
Pop-Location
```

Optional mesh export and cleanup:

```powershell
python objectA\code\export_objectA_tsdf_inputs.py `
  -s data\objectA `
  -m output\objectA_2dgs `
  --iteration 10000 `
  --output output\objectA_tsdf_inputs_grabcut

python objectA\code\fuse_objectA_mesh_open3d.py `
  --input output\objectA_tsdf_inputs_grabcut\transforms_tsdf.json `
  --output output\objectA_mesh_grabcut `
  --depth_trunc 10 `
  --voxel_size 0.02 `
  --sdf_trunc 0.10

python objectA\code\postprocess_objectA_mesh.py `
  --input output\objectA_mesh_grabcut\objectA_mesh_clean.ply `
  --output output\objectA_mesh_grabcut_final
```

### Train Object B: Text-to-3D with SDS

```powershell
Push-Location threestudio
python launch.py `
  --config configs/objectB-phone-sds-final.yaml `
  --train `
  --gpu 0
Pop-Location
```

Export the latest SDS result:

```powershell
$run = Get-ChildItem output\objectB_threestudio\objectB_text_to_3d `
  -Directory `
  -Filter 'phone_sds_final@*' |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

python objectB\code\export_objectB_mesh_simple.py `
  --run-dir $run.FullName `
  --output output\objectB\sds_phone_mesh.ply `
  --resolution 128 `
  --threshold 1.0

python objectB\code\create_objectB_enhanced_phone.py
```

### Train Object C: Single-Image-to-3D with Magic123-Style Guidance

```powershell
Push-Location threestudio
python launch.py `
  --config configs/objectC-thermos-magic123-lowvram.yaml `
  --train `
  --gpu 0 `
  tag=thermos_magic123_final
Pop-Location
```

Export the latest Magic123 result:

```powershell
$run = Get-ChildItem output\objectC_magic123\objectC_single_image_to_3d `
  -Directory `
  -Filter 'thermos_magic123_final@*' |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

python objectB\code\export_objectB_mesh_simple.py `
  --run-dir $run.FullName `
  --output output\objectC\magic123_mesh.ply `
  --resolution 128 `
  --threshold 0.005

python objectC\code\create_objectC_enhanced_thermos.py
```

### Train Background Counter Proxy 2DGS

```powershell
.\scene_fusion\run_counter_proxy_2dgs.ps1
```

For a full Mip-NeRF 360 `counter` run on a larger GPU, use the template:

```powershell
.\scene_fusion\run_configs\run_full_mipnerf360_counter_2dgs.ps1
```

## Test / Rendering Commands

### Test Object A: Novel-View Orbit Rendering

```powershell
python objectA\code\render_objectA_orbit.py `
  -s data\objectA `
  -m output\objectA_2dgs `
  --iteration 10000 `
  --output output\objectA_orbit `
  --frames 120
```

### Test Object A: Train-View Visual Comparison

```powershell
python objectA\code\visualize_objectA.py `
  -s data\objectA `
  -m output\objectA_2dgs `
  --iteration 10000 `
  --output output\objectA_visualization
```

### Test Object B: Export Mesh from Checkpoint

```powershell
$run = Get-ChildItem output\objectB_threestudio\objectB_text_to_3d `
  -Directory `
  -Filter 'phone_sds_final@*' |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

python objectB\code\export_objectB_mesh_simple.py `
  --run-dir $run.FullName `
  --output output\objectB\sds_phone_mesh.ply `
  --resolution 128 `
  --threshold 1.0
```

### Test Object C: Export Mesh from Checkpoint

```powershell
$run = Get-ChildItem output\objectC_magic123\objectC_single_image_to_3d `
  -Directory `
  -Filter 'thermos_magic123_final@*' |
  Sort-Object LastWriteTime -Descending |
  Select-Object -First 1

python objectB\code\export_objectB_mesh_simple.py `
  --run-dir $run.FullName `
  --output output\objectC\magic123_mesh.ply `
  --resolution 128 `
  --threshold 0.005
```

### Test Final Scene Fusion

Make sure the three final object meshes exist:

```text
scene_fusion/sources/objects/objectA_largest_100k_best.ply
scene_fusion/sources/objects/objectB_final_phone_best.ply
scene_fusion/sources/objects/objectC_final_thermos_best.ply
```

Then run:

```powershell
.\scene_fusion\run_scene_fusion.ps1
```

Expected outputs:

```text
scene_fusion/results/fused_counter_scene_orbit.mp4
scene_fusion/results/fused_counter_scene_contact_sheet.jpg
scene_fusion/results/fusion_manifest.json
```

## Expected Results

The final project report gives detailed quantitative and qualitative analysis. Key values from the completed local run:

| Task | Output | Key Metric / Observation |
| --- | --- | --- |
| Object A | 2DGS model + cleaned mesh | final loss 0.02985, PSNR 28.72 dB, SSIM 0.957 |
| Object B | smartphone mesh | final SDS loss 2640.06, best visual result kept |
| Object C | thermos mesh | final SD loss 26.82, Magic123-style single-image result kept |
| Background | counter proxy 2DGS | 300-iteration proxy model kept as reconstruction evidence |
| Fusion | multi-view video | A/B/C inserted into counter-style scene with unified scale and camera trajectory |

## Notes on Representation Fusion

Object A and the background are represented by explicit 2DGS/point-cloud-like data. Objects B and C are exported from `threestudio` / Magic123-style implicit representations into colored meshes. For final fusion, the mesh surfaces are sampled into colored points and normalized into the same coordinate system as the background. A shared renderer then projects all objects along the same camera orbit.

The original background 2DGS point cloud can look sparse if opened directly as a PLY file. Proper 2DGS visualization requires a Gaussian splatting rasterizer. For the final showcase video, a smooth counter-style geometry background is used for visual clarity while preserving the trained background 2DGS model as reconstruction evidence.

## Troubleshooting

- If `torch.cuda.is_available()` is `False`, reinstall CUDA-enabled PyTorch in the Conda environment.
- If 2DGS CUDA extensions fail on Windows, use the already trained weights from the Drive link or run on a Linux machine with a matching CUDA toolkit.
- If `threestudio` fails because of `nerfacc` or `tinycudann`, apply the low-VRAM fallback files under `objectB/code/modified_threestudio_files/` and `objectC/code/modified_threestudio_files/`.
- If Object A COLMAP reconstruction fails, reduce motion blur, increase image overlap, avoid reflective surfaces, and rerun frame extraction with more frames.
- If final scene fusion cannot find meshes, download the large assets from Drive and place them under `scene_fusion/sources/objects/`.
