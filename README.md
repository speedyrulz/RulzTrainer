> **This app has moved.** RulzTrainer is now part of [RulZ AI Tools Suite](https://github.com/speedyrulz/RulZ-AI-Tools-Suite): its source is in [apps/RulzTrainer](https://github.com/speedyrulz/RulZ-AI-Tools-Suite/tree/main/apps/RulzTrainer), its downloads are on the [Releases page](https://github.com/speedyrulz/RulZ-AI-Tools-Suite/releases) and its guide is in the [wiki](https://github.com/speedyrulz/RulZ-AI-Tools-Suite/wiki/RulzTrainer). This repository is archived and no longer updated.

<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/logo_dark.png">
    <img src="assets/logo.png" alt="Rulz" width="420">
  </picture>
</p>

# RulzTrainer

A simple diffusion-model LoRA trainer with a WebUI. Supported models:

- **MOSS-Image** (5B single-stream DiT by OpenMOSS), both the **Base** and **SFT** checkpoints. Output LoRAs load
  directly in ComfyUI with the standard *Load LoRA* node.
- **Bria FIBO 2.0** (8B): train on dev, use on Edit Turbo ([details](#fibo-20-bria-training-on-dev-using-the-lora-on-edit-turbo)).
- **Krea 2** (12B): train on Raw, use on Turbo ([details](#krea-2-training-on-raw-using-the-lora-on-turbo)).
- **Qwen-Image 2.1** (7B): text-to-image LoRAs on one 16 GB card, with Fizgig's training adapter
  ([details](#qwen-image-21)).

| Dark | Light |
| --- | --- |
| ![The Train tab in the dark look](docs/screenshots/01-train-dark.png) | ![The Train tab in the light look](docs/screenshots/02-train-light.png) |

| Wizard: hardware step | Monitor | First visit |
| --- | --- | --- |
| ![The wizard's hardware step with recommendations](docs/screenshots/03-wizard-hardware.png) | ![The Monitor tab](docs/screenshots/04-monitor.png) | ![Choose your look](docs/screenshots/05-choose-your-look.png) |

## Install

**Windows:** double-click `install.bat`. It creates `venv\`, installs CUDA PyTorch (cu130 by default; pass another
index URL as the first argument, e.g. `install.bat https://download.pytorch.org/whl/cu128`) and the requirements.
Then double-click `run_webui.bat` to start the UI and open it in your browser (optional first argument: port).

**Linux:** `./install.sh` does the same (`venv/`, cu128 PyTorch wheels by default, pass another index URL as the
first argument, or `system` to keep an already installed torch), then `./run_webui.sh [port] [host]`. On a headless
box (no display) the WebUI has no native Browse dialogs: type paths, or upload datasets on the Dataset tab.
Cloud GPUs: see [Running on vast.ai](#running-on-vastai).

**Manual:**

```bash
pip install -r requirements.txt
```

Requires a CUDA build of PyTorch 2.9+, diffusers 0.38+ and transformers 5.3+ (Qwen3.5 support).

## Run the WebUI

```bash
python webui.py --port 7870
```

Open http://127.0.0.1:7870.

New to LoRA training? Start on the **Wizard** tab. It walks through a run one step at a time (goal, model, GPUs,
dataset, training, samples, review) and explains every setting. Each one gets a recommended value based on your GPU
memory and on the image count and size of your dataset. *Use* takes one recommendation and *Apply recommended* takes
every one on the page. The wizard edits the same settings as the **Train** tab, which lists every option grouped by
topic, with rarely changed ones folded under *Advanced*. *Start training*, *Stop* and *Kill* sit in the bar at the
bottom of the Train and Monitor tabs.

**Race wizard.** For a character, the wizard's first page also asks *One run* or *Race several setups*. A race sets
up the base racer on the usual pages, then adds two:

- **Racers:** how many (2-8), which settings may differ (any setting a race configuration may change: learning rate,
  LoRA rank, adapter, self-training, seed...), then one card per racer with a field for each, filled in with the base
  value. Only what you change is written into that racer's row; the rest follows the base setup.
- **Race rules:** whether the lowest racer leaves after each round and after how many warm-up rounds, and whether a
  racer leaves when it stops improving and after how many probes, with the race's cost in training steps.

The review then offers *Start the race*, or *Adjust in all settings* to fine-tune it on the Train tab (its
Configuration race section holds the same rows).

The WebUI has a dark look (the default) and a light one. The first time you open it in a browser it asks which one
you want (*Choose your look*); the sun / moon button at the top right switches between them at any time. The choice
is kept per browser (its local storage).

1. **Dataset**: a folder of images. Each image may have a same-name `.txt` caption. Use *Scan dataset*
   to see the aspect-ratio buckets. Pick one or more *Training resolutions* (megapixels); see
   "Training resolutions and resolution step-up" below.
2. **Model**: choose `MOSS-Image-Base` or `MOSS-Image-SFT` (or a custom local directory / HF repo id).
   *Download / verify model* fetches about 30 GB into `models/`.
3. **Start training**. The *Monitor* tab shows progress, loss curve, log tail and sample images. Training runs
   as its own process: reloading or closing the page, opening it from another browser, or restarting the
   WebUI does not stop it, and the page reconnects to the running job (it opens on *Monitor* and loads the
   job's settings into the form). *Stop (save)* and *Kill* work on a reconnected job too.
4. Results: `output/<job>/<job>_<step>.safetensors` (checkpoints) and `output/<job>/<job>.safetensors` (final).

**Training queue.** *Add to queue* (next to *Start training*, and on the wizard's last page) saves the current
settings as a run that starts by itself when the one running now, and every run queued before it, is done: finished,
stopped or failed. With nothing running it starts right away. The *Queue* section on the Monitor lists the waiting
runs (move them up or down, remove them), pauses or resumes the queue, clears it and shows the last runs it started
(or why one could not start: that run stays first and the queue pauses). Each queued run is a full copy of the
settings at the time you added it, so you can change the form for the next one straight away. A job name that is
running or already queued is refused, since the later run would write over the earlier one's folder. *Kill* pauses
the queue; *Stop* goes on to the next run. The queue is kept in `output/_queue.json`, but the WebUI is what starts
the next run, so keep it running; a WebUI restarted with runs waiting and nothing running pauses the queue until you
*Resume* it.

*Save* / *Load* (in the *Training job* card at the top of the Train tab) store the whole form as YAML (Browse opens
in `configs/`). Loading a config saved by
an older version fills every option added since with its default, appends those options to the end of the file
(your values and comments are left as they are) and lists them under the buttons. Options a newer version no longer
has are ignored instead of refusing the file. `configs/example.yaml` ships with the repo and is never modified.

### Remote access with Tailscale

To use the WebUI from your other devices (laptop, phone), start it with Tailscale. On Windows, double-click
`run_webui_tailscale.bat` or run `run_webui.bat 7870 tailscale`. On Linux, run `./run_webui.sh 7870 127.0.0.1 tailscale`.
`python webui.py --tailscale` and `RULZ_TAILSCALE=1` do the same.

It runs `tailscale serve`, so the WebUI stays bound to `127.0.0.1`. Tailscale publishes it over HTTPS at
`https://<machine>.<tailnet>.ts.net:7870`, and only devices logged in to your tailnet can open it. The URL is printed
on start. It uses its own HTTPS port, so an existing `tailscale serve` on the default port (ComfyUI, for example) is
not touched. When the WebUI stops (Ctrl+C, closing its console window, logging off), the address switches to a short
"RulzTrainer is not running" notice instead of a 502 error, and the next start brings the WebUI back on it.
Only a hard kill (Task Manager, a crash) leaves the 502. `tailscale serve --https=7870 off` removes the address for good.

Requirements: Tailscale installed and logged in on this machine, and HTTPS certificates turned on for the tailnet
(admin console > DNS > HTTPS Certificates). Browsers that connect remotely get no native *Browse* buttons, because
those dialogs would open on this machine's screen. Type server paths instead, or upload datasets on the Dataset tab.

### CLI

```bash
python -m rulztrainer.train --config configs/example.yaml --set steps=1000 --set job_name=cat_lora
```

## Running on vast.ai

Everything needed is in `scripts/vast/` and the repo is public, so a template only has to point at it.

**Create the template** (once), either with the CLI:

```bash
pip install vastai && vastai set api-key <your key>
bash scripts/vast/create_template.sh
```

or in the web editor (Templates -> Create) with these settings:

| field | value |
|---|---|
| Image path:tag | `vastai/pytorch:cuda-12.8.1-auto` (CUDA 12.8 build: RTX 30 / 40 / 50 series hosts) |
| Launch mode | Jupyter + SSH, direct connections |
| On-start script | `/opt/instance-tools/bin/entrypoint.sh` |
| Ports | `1111`, `7870`, `8080`, `22` |
| Environment variables | `PROVISIONING_SCRIPT=https://raw.githubusercontent.com/speedyrulz/RulzTrainer/main/scripts/vast/provision.sh`<br>`PORTAL_CONFIG=localhost:1111:11111:/:Instance Portal\|localhost:7870:17870:/:RulzTrainer\|localhost:8080:18080:/:Jupyter\|localhost:8080:8080:/terminals/1:Jupyter Terminal`<br>`OPEN_BUTTON_PORT=1111` |
| Disk space | 120 GB (the base model download is about 30 GB; datasets, caches and outputs come on top) |

Put `HF_TOKEN` (needed for the gated MOSS-Image repos) into your **account** environment variables
(cloud.vast.ai -> Account -> Environment Variables), never into the template: account variables are injected into
every instance and land in `/etc/environment`, where the WebUI and the training subprocess read them.

**Rent an instance** from the template (one 16 GB+ GPU is enough; two GPUs enable the multi-GPU modes). On the first
boot `provision.sh` clones the repo into `/workspace/RulzTrainer`, installs the requirements into the image's
`/venv/main` (its torch is reused) and registers the WebUI as a supervisor service; this takes 3-5 minutes and
shows in the Instance Portal's log viewer. Then click *Open* on the instance: the Instance Portal lists
**RulzTrainer** (the WebUI, proxied by Caddy on port 7870 with the portal's token / basic auth, so it is never open
to the internet unauthenticated), Jupyter (file browser + terminal for uploads and downloads) and the tunnel links.

Datasets: on the WebUI's Dataset tab, *Create dataset*, open it and upload images and `.txt` captions from your own
computer (drag and drop), or put them into `/workspace/RulzTrainer/datasets/<name>` with Jupyter / `scp` / Syncthing.
Results are written to
`/workspace/RulzTrainer/output/<job>/`; download them from Jupyter before destroying the instance (the disk is gone
with it; stopping keeps it). `models/` on the instance disk caches the downloaded base model between runs.

Every path field's Browse button opens a browser of the instance's own disk inside the page (folders, and files of
the field's type with their sizes; shortcuts to RulzTrainer's `models` / `datasets` / `output`, `/workspace`, home and
a ComfyUI `models` folder when there is one), so single model files, the dataset and the config are picked instead of
typed. The same browser opens for any viewer that is not at the machine's own desktop; there the system dialog opens.

**Auto-download your own model files from a Hugging Face Storage Bucket.** Add
`RULZ_MODEL_BUCKET=speedyrulz/MOSS-Image-bucket` to the template's environment variables (web editor: Environment
Variables; CLI: `MODEL_BUCKET=speedyrulz/MOSS-Image-bucket bash scripts/vast/create_template.sh`). After the
install, `provision.sh` syncs the bucket into `/workspace/RulzTrainer/models/bucket/` in the background, keeping
its folder layout, so the WebUI is usable (dataset uploads) while about 18 GB come down. Progress is in
`models/bucket_download.log`; `models/bucket/.download_complete` appears when every file is there. Then point the
single-file fields at the files, for example:

| WebUI field | path on the instance |
|---|---|
| Diffusion model | `models/bucket/diffusion_models/moss_image_base_int8_convrot.safetensors` |
| Text encoder | `models/bucket/text_encoders/qwen3.5_9b_int8_convrot.safetensors` |
| VAE | `models/bucket/vae/flux2-vae.safetensors` |
| Turbo LoRA (samples) | `models/bucket/MOSS-Image-Base-8-steps-dmd.safetensors` |

Rerun `bash scripts/vast/download_models.sh` (with `RULZ_MODEL_BUCKET` set) to fetch files you add to the bucket
later; complete files are skipped. `RULZ_MODEL_INCLUDE="vae/* text_encoders/*"` limits the download to matching
files and `RULZ_MODEL_DIR` changes the destination. A public bucket needs no token; a private one uses `HF_TOKEN`
from your account environment variables. The script upgrades `huggingface_hub` on the instance if its version
has no bucket support.

Service control on the instance: `supervisorctl status|restart rulztrainer`; its log is
`/var/log/portal/rulztrainer.log`. `RULZ_REPO` / `RULZ_BRANCH` environment variables point the provisioning at a
fork or branch. `Dockerfile` builds an image with the install baked in (`FROM vastai/pytorch:cuda-12.8.1-auto`,
same supervisor wiring) for faster instance starts: build, push and use it as the template image; the provisioning
variable is then unnecessary.

## How it trains

- **Caching first.** On the first run the Flux2 VAE encodes every image into normalized 128-channel
  latents (2x2 patchified, batch-norm normalized exactly as the reference pipeline does), and the
  Qwen3.5-9B text encoder embeds every caption through the chat template. Both are cached under
  `<dataset>/.rulz_cache`, then the encoders are unloaded. Training only holds the DiT and the LoRA.
- **Timesteps.** `logit_normal` (mean/std), `sigmoid` (scale) or `uniform`, with an optional `timestep_shift`.
  `logsnr` is the style-friendly sampler ([arXiv 2411.14793](https://arxiv.org/abs/2411.14793), via musubi-tuner):
  the log signal-to-noise ratio is drawn from N(`logsnr_mean`, `logsnr_std`), -6 and 2 as the paper recommends, so
  most steps are very noisy (median sigma about 0.95), where a style's colours and strokes are learned. It is meant for
  style LoRAs; RulzTrainer's `style` mode tilts the same way more gently.
- **Clean-end share** (`clean_share`, after Fizgig's MOSS-Image "Slide"; off by default). It sets the share of
  training steps at the clean end, where sigma is below 0.5 and faces and fine detail are refined, and it can change
  by epoch.
  - **Where the curve starts.** RulzTrainer's default logit-normal puts 50% of steps there, MOSS's own pretraining
    curve (`timestep_sampling: reference`) about 11%.
  - **How it hits the target.** The sampled curve is moved along the logit axis until the share is hit after
    `timestep_shift`, `sigma_min` / `sigma_max` and the 1000-step grid; its shape stays. The run logs each change.
  - **Notation** (Fizgig's; percentages, and 1-based epochs of the run's image queue):

    | Value | Meaning |
    |---|---|
    | `40` | 40% all run |
    | `75:5:40` | 75% for 5 epochs, then 40% |
    | `60:7:30:15` | 60% until epoch 7, sliding in a straight line to 30% by epoch 15 |
    | `60:7:30:15:2` | the same, plus one epoch in 2 after that back at 60% (Fizgig's MOSS default, the WebUI's *Slide* button) |

  - **What Fizgig measured.** On MOSS-Image, against MOSS's own ~11% curve and on its hardest dataset, Slide was the
    best of eight schedules. It led from epoch 12 and still held 71% likeness at epoch 60. A sudden drop from 75% to
    40% dipped likeness two epochs later.
  - **Unmeasured here.** RulzTrainer's default already starts at 50%, so race `clean_share: "60:7:30:15:2"` against
    `""` on your own data. It works with every model and timestep mode.
- **Flow matching loss.** `x_t = (1 - sigma) x0 + sigma * noise`, target `noise - x0`, timestep = `sigma * 1000`,
  sigmas snapped to the model's 1000-step training grid. Default sampling is logit-normal; `timestep_shift`
  is available.
- **LoRA** on the DiT attention and FFN projections (`q, k, v, o, ffn_up_gate, ffn_down`) in all 42 blocks, or
  only the blocks you pick (`lora_blocks`, e.g. `21-41`, `10-31,40`, `even`; the UI has a per-block toggle grid
  with presets). Blocks without LoRA get no backward pass if they sit before the first trained block, so
  training only later blocks is also faster per step. The exported file simply omits the untrained blocks.
  Optionally also the two text-projection blocks inside the DiT. The Qwen text encoder stays frozen, as the
  authors recommend.
- **Samples** are generated with the reference Euler schedule (shift 5, CFG 3, negative-branch layer-22 skip).

## Using two GPUs

`multi_gpu: data_parallel` with `devices: cuda:0,cuda:1` loads a full model copy on each GPU; every step each
copy trains its own images (effective batch = batch_size x GPUs), the LoRA gradients are averaged onto the
primary, the optimizer steps once and the LoRA weights are broadcast back. This is the mode that speeds
training up (roughly 1.7-2x throughput with two cards; the faster card waits for the slower one each step).
It needs no torch.distributed and works on Windows; each GPU must fit the whole model, so pair it with
`transformer_quant: int8_convrot` or `nvfp4` on 16 GB cards, or bf16 at modest resolutions.

Measured on an RTX 5060 Ti + RTX 4060 Ti at 1 MP, seconds per step of 2 images (per image in brackets):

| model | eager | compiled |
|---|---|---|
| MOSS-Image, int8_convrot | 7.30 (3.65) | **4.79 (2.40)** |
| Krea 2, nvfp4 | 12.74 (6.37) | 11.81 (5.91) |

### Unequal GPUs (`dp_balance`)

Each step waits for the slowest GPU. `dp_balance: auto` (the default) times every GPU's batches during the first
steps, logs them, and when the GPUs differ splits a step's batches by speed: with `gradient_accumulation: 4` on two
cards that run 1 : 1.6, 5+3 instead of 4+4. The images per step and the random draws stay the same; only the waiting
goes. A split replaces lockstep only when it is at least 5% faster per image; it is re-checked every 25 steps (another
job may share a card), resets when a resolution step-up starts a new stage, and is kept in training states. With
one batch per GPU per step there is nothing to split; `dp_balance: grow` then lets the faster GPU run more batches in
the slower one's time (for example 3+2), so a step holds more images and `steps` should be lowered to match.
`off` keeps lockstep. Drawing all of a step's batches first also drops the sync points between micro-batches:
Krea 2 with `gradient_accumulation: 4` trains at 6.26 s per image against 6.37 at 1 per step.

On these two cards the balancer finds nothing to split. Alone, the 5060 Ti trains Krea 2 nvfp4 at 8.5 s per image
(FP4 tensor cores) and the 4060 Ti at about 12.5 (it dequantizes), but together they measured 14.2 and 12.4 s per
batch. Both GPUs are driven by threads of one Python process; the 5060 Ti's many short FP4 kernels most likely wait
on the other thread's turn at the interpreter, while the 4060 Ti's longer kernels hide it. Compiling cuts the work
the threads do in Python (see the table above).

`multi_gpu: model_split` places consecutive block ranges on different GPUs instead. It halves the memory per
GPU (full bf16 at high resolution becomes possible) but is not faster, since the GPUs take turns. The split is
auto-balanced by estimated cost (blocks before the first trained block are forward-only and cheap; every later
block also runs backward), or set `split_plan: "cuda:0=0-25;cuda:1=26-41"` to give the faster card more.

### Splitting side jobs across the GPUs (`parallel_jobs`, on by default)

With either multi-GPU mode, the work around training is split across the GPUs too. Each GPU has its own worker
that pulls the next job from a shared queue, so the faster card simply does more of them:

- **Samples and probes**: every sample prompt and every probe render (face, body, each seed) is a job. Two probes
  means one per GPU. In data parallel each GPU renders on its own training copy. In model split the training
  model spans both GPUs, so images split only when a separate sampling model is set (it gets a copy per GPU).
  Scoring stays on the main thread after the renders.
- **Latent caching**: a VAE copy per GPU, each encoding its share of the images (and their flips).
- **Text caching**: a second text encoder copy is loaded on the other GPU when the encoder sits wholly on one
  GPU (a quantized Qwen3.5-9B, about 10 GB) and timing shows the copy saves more than it costs to load. A bf16
  encoder (19 GB) is spread over both GPUs and stays a single copy.
- **SAM 3 region masks** (`region_weights` with `mediapipe+sam3`): a SAM 3 copy per GPU, each kept resident for
  the whole batch instead of moving to the GPU and back for every image.

Face detection (InsightFace) and MediaPipe run on the CPU and are not split. With `compile_model`, the first
sampling round of each image size runs one GPU after the other while the graphs compile; later rounds run in
parallel. Turn the switch off ("Split samples, probes and caching across GPUs") to keep all side jobs on the
primary GPU.

Measured with data parallel on an RTX 5060 Ti + RTX 4060 Ti (8 images, 512 px, 8 sampling steps):

| Side job | One GPU | Split across both |
|---|---|---|
| 2 samples | 28.6 s | 14.7 s |
| Face + body probe (render and score) | 35.4 s | 17.7 s |
| Latent cache, 8 images + flips | 5.6 s | 1.9 s |
| SAM 3 region masks, 8 images | 16 s (was 28 s before SAM 3 stayed resident) | 9 s |

Text caching is only split when it pays: the first few prompts are timed on the loaded encoder, and a second copy
is loaded only if it saves more time than loading it takes (about 13 s for the int8 Qwen3.5-9B). Images rendered on
different GPU models differ very slightly in their pixels (different kernels); the same seed gives the same picture.

## Model sources and quantization

**Single safetensors files (the default, `model_source: files`)**: the model you pick comes as ComfyUI files from
Comfy-Org's repackages on Hugging Face, downloaded on the first run (into Hugging Face's cache, reused after that; the
WebUI's *Download now* fetches them ahead of time). `model_file_quant` and `text_encoder_file_quant` pick the format,
`int8_convrot` by default (about half of bf16, trains as fast, under 1% error):

| Model | Repo | Diffusion model | Text encoder | VAE |
|---|---|---|---|---|
| MOSS-Image Base / SFT | [Comfy-Org/MOSS-Image](https://huggingface.co/Comfy-Org/MOSS-Image) | int8_convrot 5.4 GB, bf16 10.2 GB | Qwen3.5-9B int8_convrot 9.8 GB, bf16 19.3 GB | `flux2-vae_small` |
| Krea 2 Raw / Turbo | [Comfy-Org/Krea-2](https://huggingface.co/Comfy-Org/Krea-2) | int8_convrot 13.5 GB, fp8_scaled 13.1 GB, bf16 26.3 GB | Qwen3-VL-4B int8_convrot 4.9 GB, fp8_scaled 5.2 GB, bf16 8.9 GB | `qwen_image_vae` |
| FIBO 2.0 dev / Edit Turbo | [Comfy-Org/Fibo-2.0](https://huggingface.co/Comfy-Org/Fibo-2.0) | int8_convrot 8.4 GB, bf16 16.5 GB | Qwen3-VL-4B, the same files as Krea 2's | `flux2-vae` |
| Qwen-Image 2.1 / Turbo | [Comfy-Org/Qwen-Image-2.1](https://huggingface.co/Comfy-Org/Qwen-Image-2.1) | int8_convrot 7.3 GB, bf16 14.2 GB | Qwen3-VL-8B int8_convrot 9.4 GB, bf16 17.5 GB | `qwen_image_2.1_vae_bf16` |

To use files you already have, set `transformer_file`, `text_encoder_file` and `vae_file` (the WebUI's optional file
fields): only the ones left empty are downloaded. Fused-qkv ("repacked") files are split back automatically. The
tokenizer is downloaded once (a few MB) if not given. MOSS-Image only takes its own Qwen3.5 tokenizer: earlier
versions took the first tokenizer folder under `models/`, Krea 2's when one was there, which turned every MOSS-Image
prompt into the wrong tokens (noise in samples and in the cached training captions); those cached captions are
encoded again automatically. For decoding MOSS-Image samples use the VAE that matches
MOSS-Image (the FLUX.2 klein "small" VAE, `flux2-vae_small.safetensors`); the encoder is identical across Flux2
VAEs, so caching latents works with either.

**Hugging Face / diffusers folder** (`model_source: hf`): the original diffusers repos (pick Base or SFT, or any
local folder with `transformer/`, `text_encoder/`, `tokenizer/`, `vae/`), downloaded into `models/`.

**FIBO 2.0 from single files**: ComfyUI's FIBO file fuses `to_q/to_k/to_v` into `qkv`, the cross-attention and
Perceiver `to_k/to_v` into `to_kv` and the feed-forward `w1/w3` into `w13`, and zero-pads the feed-forward width from
9536 to 9728. On load the fused weights are split back (an int8 weight is cut by rows with its per-row scales) and the
padding is cut off, so the model, and every LoRA trained on it, has Bria's shapes: a LoRA trained from single files
loads in diffusers. The int8 `w2` loses its padding columns by being dequantized, cut and quantized again (its
Hadamard rotation groups do not survive the cut), as per-row int8. Checked against Bria's weights: every split part
and every cut, re-quantized weight matches at cosine 1.00, and every diffusers weight is present. The text encoder is
Krea 2's Qwen3-VL-4B file (byte-identical in both repos) run text-only, the VAE MOSS-Image's Flux2 loader.

**Krea 2 from single files** works the same way with ComfyUI's Krea 2 files: the diffusion model (ComfyUI's own
`blocks.N.attn.wq` layout is renamed to ours on load; `transformer_quant: auto` keeps a quantized file's format where
it fits, converts an int8 file to nvfp4 on a single 16 GB card, and quantizes a bf16 / fp8 file for the memory it runs
in; a file name containing "turbo" marks the distilled model, whose
sampler shift is fixed), the Qwen3-VL-4B text encoder (ComfyUI's or the HF layout; the vision tower is dropped and the
language model runs text-only, which gives the same features) and the Qwen-Image VAE (its original Wan layout is
converted with diffusers' converter; the latent statistics are built in). The Qwen3-VL tokenizer comes from a
downloaded Krea 2 repo if there is one, else `Qwen/Qwen3-VL-4B-Instruct` (not gated, a few MB). The family is read
from the diffusion-model file's header, so the file name does not matter. Checked against the Hugging Face model:
the VAE file matches exactly, the int8 and scaled-fp8 text encoder files match the bf16 encoder at cosine 0.999
(quantization noise), and ComfyUI's int8_convrot DiT file matches the HF weights quantized the same way.

Accepted weight formats: bf16 / fp16 / fp32, fp8 (upcast) and ComfyUI's scaled fp8 (`weight_scale` applied on load),
ComfyUI `int8_tensorwise` with or without ConvRot (Hadamard rotation), and ComfyUI `nvfp4`. Those two quantized
formats are decoded exactly as comfy_kitchen does (verified against the HF weights: re-quantizing them with
RulzTrainer reproduces the ComfyUI file bit for bit).

*Diffusion model quant* / *Text encoder quant*:

| value | effect |
|---|---|
| `auto` | keep the file's format (bf16 stays bf16, int8/nvfp4 files stay quantized) |
| `none` | dequantize everything to bf16 |
| `int8`, `int8_convrot` | quantize eligible Linear layers on load (about half the VRAM) |
| `nvfp4` | quantize on load to FP4 with FP8 block scales (about a quarter of the VRAM) |

Only the frozen base weights are quantized. The LoRA trains in fp32. Exported LoRAs are bf16 by default
(`lora_save_dtype`).

### int8 compute: W8A8 (`int8_matmul`, `int8_backward`)

int8 and int8_convrot layers do not dequantize their weight any more. With `int8_matmul: true` (the default)
each Linear runs **W8A8**: the activations are quantized per token to int8 (symmetric, fp32 scale per row), the
int8 activations are multiplied with the stored int8 weight by `torch._int_mm` (int32 accumulate), and the
result is rescaled by the activation scale and the per-output-row weight scale in the compute dtype. For
int8_convrot files the activations are first rotated with the same grouped Hadamard (group 256) the weight was
stored with: x W^T = (xH)(WH)^T, so no inverse rotation of the weight is needed, and the rotation spreads
activation outliers over the group, which is what makes per-token int8 accurate on these models.

The two elementwise passes around the GEMM (per-token quantization of the activations, int32 -> bf16 rescale by
both scales) are single fused triton kernels when `triton` imports (`triton-windows` on Windows, in
`requirements.txt`; triton ships with the Linux torch wheels). For int8_convrot the Hadamard rotation of the
activations is folded into the quantization kernel: H_256 = H_16 (x) H_16, so each 256-group of a token row is
rotated as H_16 X H_16 with two tiny tensor-core dots (fp32 accumulate, the intermediate carried as a bf16
hi + lo pair), the row amax is taken over the rotated values and they are quantized straight from registers;
the rotated bf16 tensor is never written. Without triton the eager PyTorch versions run (a bf16 matmul for the
rotation, then two and three memory passes); the choice is made once at import and a kernel failure at run time
logs one warning and switches to eager. `RULZ_W8A8_TRITON=0` forces the eager path, e.g. for the before/after
bench.

The previous behaviour (`int8_matmul: false`) dequantized the weight to bf16 in every forward *and* every
backward, three times per step with gradient checkpointing, which made int8 training about 3x slower than
bf16. Measured with `scripts/bench_int8.py` (4096 tokens, int8_convrot, 50 iterations; "eager" is the same
W8A8 path with `RULZ_W8A8_TRITON=0`), forward / forward+backward ms:

| layer (K -> N) | path | RTX 5060 Ti | RTX 4060 Ti |
|---|---|---|---|
| 3072 -> 8192 (ffn up) | bf16 weight | 4.1 / 8.3 | 4.8 / 8.9 |
| | W8A8 fused, bf16 backward (default) | **1.9** / 6.7 | **2.3** / 7.2 |
| | W8A8 fused, int8 backward | 1.9 / 4.7 | 2.3 / 5.6 |
| | W8A8 fused, Hadamard as a separate matmul (previous) | 2.0 / 6.7 | 2.4 / 7.3 |
| | W8A8 eager, bf16 backward | 2.9 / 7.5 | 3.5 / 8.3 |
| | W8A8 eager, int8 backward | 2.9 / 6.6 | 3.5 / 8.2 |
| | dequantize per call (old) | 6.5 / 12.9 | 7.3 / 14.6 |
| | `torch._int_mm` alone | 1.2 | 1.3 |
| 8192 -> 3072 (ffn down) | bf16 weight | 4.2 / 8.3 | 4.7 / 8.9 |
| | W8A8 fused, bf16 backward | **1.7** / 6.5 | **2.0** / 7.2 |
| | W8A8 fused, int8 backward | 1.7 / 4.6 | 2.0 / 5.5 |
| | W8A8 fused, Hadamard as a separate matmul (previous) | 2.1 / 7.1 | 2.5 / 7.7 |
| | W8A8 eager, bf16 backward | 3.2 / 8.3 | 4.0 / 9.2 |
| | dequantize per call (old) | 6.4 / 12.9 | 7.3 / 14.6 |
| 3072 -> 3072 (q/k/v/o) | bf16 weight | 1.5 / 3.1 | 1.8 / 3.3 |
| | W8A8 fused, bf16 backward | **0.8** / 2.6 | **1.0** / 3.0 |
| | W8A8 fused, int8 backward | 0.8 / 2.0 | 1.0 / 2.6 |
| | W8A8 fused, Hadamard as a separate matmul (previous) | 0.9 / 2.8 | 1.1 / 3.1 |
| | W8A8 eager, bf16 backward | 1.5 / 3.3 | 1.9 / 3.8 |
| | dequantize per call (old) | 2.4 / 4.8 | 2.7 / 5.5 |

With the fused kernels the forward is 2-2.4x faster than bf16 on every MOSS layer shape (the eager W8A8
path was 1.4x on the ffn layers and only bf16 speed on the 3072 -> 3072 ones). Folding the Hadamard into the
quantization kernel took the rotate + quantize step from 0.31 / 0.93 ms (K = 3072 / 8192, 4060 Ti) to
0.13 / 0.44 ms; the remaining gap to the bare int8 GEMM is the int32 -> bf16 rescale of the output plus that
one read of x. With checkpointing the forward runs twice per step, so a step is bf16 speed or better at half
the VRAM. Forward error versus the dequantized weight is under 1% (the activation quantization), the same
order as the weight quantization itself. The fused quantizer and rescale round exactly like the eager ones
(the rescale even avoids the eager path's second bf16 rounding); the fused rotation keeps the rotated values
in fp32 where the separate bf16 matmul rounds them, so its int8 values are within 1 LSB of the two-step path
and closer to the exact rotation (about 5% of the two-step values are 1 LSB off an fp64 reference, about
0.002% of the fused ones).

`int8_backward` chooses how grad_x is computed:

| value | effect |
|---|---|
| `bf16` (default) | exact: the int8 weight is widened to bf16 once per backward and grad_x = grad_out @ W (no activation or gradient quantization; convrot applies the inverse Hadamard to grad_x) |
| `int8` | the weight scale is folded into grad_out, grad_out is quantized per token to int8 and multiplied with the int8 weight; about 1% gradient error, saves the bf16 weight copy and is slightly faster |

Fallbacks: CPU tensors and shapes the int8 kernel rejects (in/out features not multiples of 8) use the
dequantize path silently; a kernel error at run time logs one warning and falls back for that call. Rows are
padded to 17 when a call has 16 tokens or fewer, because `torch._int_mm` refuses smaller M. In the WebUI these
are *int8 layers compute* and *int8 backward* in the model section.

### nvfp4 compute: native FP4 on Blackwell (`nvfp4_matmul`)

nvfp4 layers multiply on the **FP4 tensor cores** of Blackwell GPUs (RTX 50, compute capability 12.x; B200 10.x)
with `nvfp4_matmul: true` (the default): the activations are quantized to NVFP4 by a fused triton kernel (per
16-value block an e4m3 scale written straight into cuBLAS's swizzled layout, values rounded to e2m1 nearest-even,
two per byte, plus a per-tensor fp32 scale), and `torch._scaled_mm` multiplies them with the stored NVFP4 weight.
That is **W4A4**: faster than any other path, but the activations lose precision too. By default the backward
stays exact against the stored weight (dequantized to bf16 for grad_x), as the int8 layers' default does; an
opt-in FP4 backward is described below.

GPUs without FP4 tensor cores (RTX 40 and older), `nvfp4_matmul: false`, CPU tensors and K not a multiple of 32
dequantize instead, now with one fused triton kernel (codes and scales read once, bf16 written directly; bit-identical
to the previous eager lookup). On a mixed machine CUDA numbers the GPUs by speed, not by slot, so check which index
is the RTX 50 card: the training log lists the GPUs that run native FP4 and names a capable GPU that was not picked.

Measured on an RTX 5060 Ti, one Krea 2 feed-forward layer (6144 -> 16384) on 4113 tokens:

| path | forward | error vs exact (random weights, uneven activations) |
|---|---|---|
| eager dequantize (previous) | 29.8 ms | 9.5% |
| fused dequantize | 17.2 ms | 9.5% (identical) |
| **native FP4 (W4A4)** | **4.0 ms** | 13.4% |
| bf16 weight, for reference | 16.7 ms | 0 |

The exact backward of that layer takes 17.5 ms either way. The kernels are checked bit for bit against the
reference quantizer and dequantizer in `tests/test_quant.py` (the native-matmul test runs on a Blackwell GPU only).

Whole training steps, Krea 2 LoRA (rank 16) in nvfp4 at 1 MP on one RTX 5060 Ti, batch 1, gradient checkpointing,
20 steps on the same 8 images:

| path | s/step | loss (average over 20 steps) |
|---|---|---|
| eager dequantize (previous) | 16.7 | 0.0714 |
| fused dequantize | 12.5 | 0.0713 |
| **native FP4** | **9.2** | 0.0724 |

Attention, norms and the bf16 backward are the same in all three, so the step gains less than the matmul alone.
Samples rendered through the native path come out clean.

**FP4 backward (`nvfp4_backward: fp4`, opt-in).** With native FP4 on, grad_x = grad_out @ W can run on the FP4
tensor cores too. grad_out is quantized to NVFP4 in 16-value blocks along the output axis with stochastic rounding
(unbiased; its block scales round up, so no value clips or flushes to zero), and the weight is requantized along the
other axis on the fly (a stored copy would cost 6 GB for Krea 2). Only the gradient flowing back through the frozen
base weights changes; the LoRA's own gradients stay in full precision. The default stays `bf16` (exact).

Measured on the RTX 5060 Ti, Krea 2 layer shapes on 4113 tokens:

| layer | exact backward | FP4 backward | grad_x error vs exact |
|---|---|---|---|
| feed-forward up (6144 -> 16384) | 17.6 ms | 11.4 ms | 17.6% (cosine 0.985) |
| feed-forward down (16384 -> 6144) | 17.3 ms | 9.8 ms | 17.5% |
| attention q (6144 -> 6144) | 6.5 ms | 4.6 ms | 17.5% |

On the whole model (one fixed batch, all LoRA gradients at once) the FP4 gradients point the same way as the exact
ones with cosine 0.99 at sigma 0.2, 0.93-0.96 at 0.5 and 0.97-0.98 at 0.8, and are 1-5% shorter. A 20-step run
(same setup as above, back to back):

| backward | s/step (steps 5-20) | loss (average over 20 steps) | peak memory |
|---|---|---|---|
| bf16 (exact) | 8.73 | 0.0724 | 12.9 GB |
| **FP4** | **7.93** (-9%) | 0.0724 (every step within 0.0007) | 13.1 GB |

20 steps show that training behaves the same, not what thousands of steps do with noisier gradients. Compare
samples on your own data before using it for a final run.

### Attention kernel (`attention_backend`)

The DiT blocks run PyTorch's scaled-dot-product attention. `torch` lets PyTorch pick the kernel (flash where it
can); `cudnn`, `flash`, `efficient` and `math` put one first; `auto` (the default) puts cuDNN first on a Blackwell
training GPU (RTX 50, B200; ai-toolkit uses it there too) and lets PyTorch pick elsewhere. The choice is PyTorch's
process-wide priority order, so every other kernel stays enabled behind the chosen one and a call it does not take (a
mask, dtype or head size) runs on the next. Until v2.3.1 the choice switched the other kernels off during each DiT call,
which broke a VAE decode running in another thread at that moment (samples and probes split over two GPUs): Krea 2's
fp32 VAE attention found no kernel enabled ("No available kernel"). Measured on
the RTX 5060 Ti, 1 MP, batch 1, 20 steps (median of steps 6-20), two runs each:

| model | PyTorch's pick | cuDNN | loss (both) | peak memory (both) |
|---|---|---|---|---|
| Krea 2 nvfp4 | 9.40 / 8.56 s per step | 7.90 / 6.70 s per step (-16% / -22%) | 0.0725 | 14.3 GB |
| MOSS-Image int8 | 6.66 / 6.65 s per step | 5.27 / 5.28 s per step (-21%) | 0.801 | 12.9 GB |

The attention kernel is a fifth of the step at 1 MP; the RTX 40 series (Ada) is not measured, so `auto` leaves
PyTorch's pick there. Set `cudnn` to try it on any card: the log says which kernel runs.

## Training resolutions and resolution step-up

*Training resolutions* is a list of megapixel values (`resolutions: [0.25, 1.0, 4.0]` in YAML; the WebUI offers
0.25 to 4 MP in 0.25 steps, 1 MP = 1024x1024). Every image is bucketed to each selected area with its aspect
ratio preserved (sides multiples of 16), and latents are cached once per image and resolution, so a second run at
the same resolutions starts immediately. `min_bucket_side` / `max_bucket_side` now rescale the bucket uniformly
before clamping, and the minimum is capped at half the resolution, so a 0.25 MP bucket keeps ~512x512 of area
(ai-toolkit style) instead of getting a 512 short edge, and the maximum is raised to twice the resolution so 4 MP buckets keep their area too. The old single `resolution`
(a side length) still works when `resolutions` is empty.

With several resolutions selected and *Resolution step-up* off, all of them are mixed in every epoch (one item per
image and resolution, so an epoch is N times longer). Per-image loss watch, curriculum weights and exclusions are
keyed by image, so they apply to every resolution of that image.

**Resolution step-up** (`resolution_step_up: true`) trains the resolutions one after another, lowest first, and
lets the likeness probes decide when to move on:

* every probe (`probe_every` steps) yields the smoothed combined face/body score; the first probe of a stage is
  that stage's baseline and a later probe counts as an improvement when it beats the stage's best by more than
  the noise band;
* after `step_up_patience` probes in a row without improvement (default 1) the batcher switches to the next
  resolution; a new stage starts with a fresh baseline, so a dip right after the switch does not count against it;
* at the highest resolution, `step_up_stop_patience` probes without improvement (default 1) stop training; the
  final LoRA is saved as usual and `<job>_best.safetensors` still tracks the best score over the whole run.

`steps` stays the hard upper limit. In this mode the plain early stop (`likeness_patience`) is not used; the
learning-rate control, curriculum and best-checkpoint actions keep working across stages. The mode needs probes
(`probe_every > 0` with face and/or body references); the trainer refuses to start otherwise. The current
resolution and stage are logged, shown in the monitor line and written to `likeness.jsonl` (`stage`,
`stage_best`, `stage_since_improve`) and to training states, so a resumed run continues at the same stage.

## Training states

`save_state: true` writes `output/<job>/state_XXXXXX/` folders (fp32 LoRA, optimizer, LR scheduler and RNG
states) every `save_state_every` steps (0 = with each LoRA save), keeps a rolling `keep_states` of them
(0 = all), and always writes the final state when training ends or is stopped. Set `resume_state` to one of
those folders to continue exactly where it stopped. `resume_from` loads only LoRA weights from any exported file.

## Checkpoints to keep (`keep_checkpoints`)

`save_every` writes a LoRA or LoKr file every N steps (`<job>_000250.safetensors`, and `_r<rank>` files beside it for a
fine-tune's extra ranks). `keep_checkpoints: N` keeps only the newest N of those and deletes the oldest each time a new
one is saved; 0 (the default) keeps them all. The final file (`<job>.safetensors`), the best-likeness file
(`<job>_best`), a race's dropped and winner files and the training states are never deleted: states have their own
`keep_states`. In a race each configuration keeps its own newest N. The WebUI's *Checkpoints to keep* field is in
Saving & samples.

## Weight average (`lora_ema`)

At batch size 1 each step pulls the LoRA toward the image it just saw, so a checkpoint carries the tail of the
shuffle. With `lora_ema: 0.98` (the default; 0 = off) every checkpoint, sample, likeness probe and `_best` file is
made from a running average of the LoRA over roughly the last 1 / (1 - decay) images (50 at 0.98), while training
goes on with the raw weights. The decay is per image, so the window is the same at any batch size, gradient
accumulation or number of GPUs; it warms up over the first few hundred images and is capped at a quarter of a short
run. Training states keep the raw weights in `lora.safetensors` and the average beside them, so a resume continues
both; a state without one starts the average from the resumed weights. The average costs one fp32 copy of the LoRA
(about 0.2 GB at rank 16) and under a millisecond or two per step.

The default follows [Fizgig](https://github.com/shootthesound/Fizgig)'s measurement on MiniMax H3 (4-way A/B, 50
epochs, same dataset and seed): decay 0.98 scored 61.5% likeness over epochs 41-50 against 56.5% without an average,
with the swing between epochs down from 3.9 to 1.6 points; 0.99 smoothed without lifting it, and 0.995 averaged over
so many steps that it lagged the run and finished below no average. Fizgig uses 0.98 for Krea 2 and MOSS-Image too;
it has not been measured on the models here.

## Samples and monitoring

`sample_seed` is the seed of the first prompt; with `sample_walk_seed` each further prompt uses seed + 1.
`sample_before_training` renders a sample set at step 0 for comparison. The log reports each prompt and the
Monitor page shows the sampling step. The Monitor page also has per-GPU VRAM/utilization bars, an adjustable
loss-smoothing slider, an auto-scrolling log and, under *Log and saved files*, a download button for every LoRA the run
has saved (the checkpoints, the best and the final file), so a run on a rented or remote GPU hands its LoRAs straight to
the browser. Only `.safetensors` files inside a run's folder are served. `compile_model` (default `auto`, see
[torch.compile](#torchcompile-compile_model)) runs the DiT blocks through torch.compile
(skipped with a warning if triton is not installed; `triton-windows` from requirements.txt provides it on Windows).
No C++ compiler is needed: the blocks are compiled with Dynamo's `specialize_float` so the LoRA scale and
multiplier stay Python constants instead of becoming CPU tensors (which would make Inductor emit C++ and fail with
"InvalidCxxCompiler: cl is not found" on a machine without MSVC on PATH), and the fused W8A8 kernels run eagerly
behind a graph break, so int8 layers keep their fused triton path inside compiled blocks.

## Identity similarity for character LoRAs (face + body)

Pick reference images on the Dataset tab (*Face ref* / *Body ref* toggles) or list them as `face_refs` /
`body_refs`. After every sampling pass each sample is scored and the results go to `face_scores.jsonl`, the
Monitor chart and the sample captions:

- **Face**: the largest face in the sample is embedded with InsightFace (ArcFace, `buffalo_l`) and compared by
  cosine similarity to every reference face; the best match is reported as a percent (cosine x 100). For
  ArcFace, 50%+ generally means the same person; a typical reference's best match among the others is shown as
  the realistic ceiling.
- **Body** (`body_score: appearance`, the default): the body from the neck to mid-thigh, turned upright, cut out
  from the background and without the head, embedded with DINOv2 and compared with the reference photos that show
  the same part of the body; the best match is reported as a percent (details under *Body score* below). Body
  references should include three-quarter or full-length shots; samples that show no hips (portraits) or too
  little of the window get no body score.
- **Body** (`body_score: proportions`): MediaPipe pose landmarks give scale-free proportion ratios (shoulder/hip
  width, torso/leg length, upper/lower leg, upper arm/forearm, head/shoulder width, leg/height...). With
  `body_backend: mediapipe+sam3`, a SAM 3 person mask adds silhouette widths at the shoulder, waist, hip and thigh
  levels (normalised by body height) plus waist/hip and shoulder/waist, which captures build rather than only bone
  proportions. A median profile over the body references is the target; a sample scores by how close its ratios
  are (20% deviation on a ratio ~ 61%). Silhouette features are only used when the skeleton confirms a full-body
  view, and samples without a usable pose get no body score. Body references should be full-body, upright,
  fairly neutral poses, and the same goes for the sample prompts you want body-scored.
- **Body** (`body_score: skeleton3d` or `smplx`): proportions measured so that the pose does not change them, against
  the references' own spread (details under *Body score* below).
- **Combined**: `face_weight` x face + `body_weight` x body (defaults 0.7 / 0.3; whichever is available).

Model downloads (the InsightFace pack, the MediaPipe pose model and SAM 3) happen on first use with a progress
bar in the UI (reference checks) and in the status line during training.

Requirements: `insightface` + `onnxruntime-gpu` (face), `mediapipe` (body); `facebook/sam3` is a gated
Hugging Face model (accept the license, then `hf auth login`), or point `sam3_model` at a local
transformers-format folder. SAM 3 (3.4 GB) is kept in system RAM and moved to `body_device` only while scoring.

## Per-image loss watch and per-image learning rate

`loss_watch: true` records every image's loss normalised by the expected loss at its noise level (40 sigma
buckets, kernel-smoothed) and smooths the residual per image. At each epoch boundary every image gets a
verdict relative to the cohort median: *easy*, *healthy*, *suspect* (hard but improving), *stuck* (hard and not
improving for two epochs) or *exhausted* (learned, flat). `per_image_lr: true` acts on the verdicts the way
Fizgig's loss watch does: stuck images are throttled x0.5, then x0.25, then x0.125 (recovering one rung per
improving epoch), exhausted ones ease to x0.6, the healthy cohort gets `loss_watch_healthy_mult`. With batch
size 1 and no gradient accumulation this is a true per-image learning rate (the optimizer lr is scaled for that
step); otherwise it becomes a per-sample loss weight. `loss_watch_exclude_after: N` benches an image that stays
at the lowest throttle for N more epochs; exclusions are remembered in `<dataset>/.rulz_cache/excluded.json`
and cleared by editing the image's caption. Output: `per_image_loss.jsonl` (every observation),
`per_image_loss.json` (the table), a Monitor table and verdict badges on the Dataset tab. Not used with
`automagic3`, whose step normalisation would absorb the per-image scaling. Verdicts need a few epochs to mean
anything, so this suits typical character sets trained for thousands of steps.

## Identity training: what each step teaches (character LoRAs)

Four levers change the training objective and data for likeness, all off by default:

- **Region-weighted loss** (`region_weights`): at cache time the largest face (InsightFace) and the person
  silhouette (SAM 3 with `body_backend: mediapipe+sam3`, else a MediaPipe landmark hull) are mapped onto the
  16-px latent cells of each bucketed image and turned into a weight map (`face_loss_weight` 3 / `body_loss_weight`
  1.5 / `background_loss_weight` 0.3, `face_margin`, `region_feather_cells`), normalised to mean 1 and stored as
  `<stem>_regions.safetensors`. The per-cell squared error is multiplied by it. `region_prior_weight` adds an
  optional prior-preservation term on background cells toward the base model's prediction (extra forward).
- **Timestep emphasis**: `timestep_sampling: identity` (logit-normal mean -0.5, median sigma ~0.38),
  `content` (sigma = u^`content_power`, low-noise heavy) or `style` (mirror), plus `sigma_min` / `sigma_max`.
- **Automatic crops** (`crop_augment: face | face+bust`): a square face crop (`crop_face_scale` x face size) and a
  bust crop (`crop_bust_scale`) per image with a face, with captions from `crop_face_caption` / `crop_bust_caption`
  (`{caption}`, `{raw}`, `{trigger}`), skipped when the image is already a close-up or the crop would be tiny.
  Crops are ordinary items (own latents, captions, region maps, loss-watch verdicts) and count as face-heavy for
  the curriculum; `crop_augment_weight` scales how often they are drawn.
- **LoRA+** (`lora_plus_ratio`, 16 = paper): the `lora_up` matrices train at lr x ratio for faster convergence
  (AdamW-style optimizers; ignored by automagic3). Resuming a state requires the same ratio.
- **Automagic lr groups** (`automagic_groups: by_type`, automagic3 only): text-side layers (Krea's text fusion,
  MOSS's text projection, FIBO's perceiver), attention, feed-forward and the rest each adapt their own learning
  rate instead of one shared rate the largest group dominates. Fizgig trains Krea 2 this way (its text fusion is the
  slow minority), with `automagic_polarity_history: 16`. Resuming a state requires the same setting.

The face analysis cache is `<dataset>/.rulz_cache/faces.json`; the Train tab has preview buttons for region
maps and crops.

Measured on a 5-image synthetic character set (120 steps, probes every 20 steps, same seeds, face likeness EMA
at step 100; baseline 74.2): region-weighted loss 76.0, LoRA+ ratio 4 74.7, `identity` timesteps 73.9, auto crops
65.5 (the only crops available were 300 px and had to be upscaled), everything combined with LoRA+ 16: 63.1
(unstable, loss 40% higher). The **Identity training preset** (a switch at the top of the Train tab, `--preset identity` on the CLI, or
`preset: identity` recorded in the YAML) applies the whole recommendation in one go and restores your previous
settings when switched off. Suggested recipe: `region_weights: true`, `lora_plus_ratio: 4` with `adamw8bit`,
crops only from high-resolution photos (`crop_augment_min_px: 0` means crops are never upscaled), timesteps at
`logit_normal`; treat `identity` timesteps and the curriculum as options to A/B with the probes.
In the Wizard, *Character / person* then asks how much of the person the likeness covers, and sets the loss weights
(face region / person region), the score weights, the crops and the probes for it:

| Focus | Loss face / person | Score face / body | Crops | Probes |
|---|---|---|---|---|
| Face only | 3 / 1 | 100% / 0 | face | face only (no body references) |
| Face focus (the recipe) | 3 / 1.5 | 70% / 30% | face + bust | both |
| Face and body | 1.5 / 1.5 | 50% / 50% | face + bust | both |
| Body focus | 1.5 / 3 | 30% / 70% | face + bust | both |
| Body only | 1 / 3 | 0 / 100% | none | body only (no face references) |

The curriculum is on for the three in the middle. The wizard also recommends the adapter (a LoRA for every model),
block swap for a card the model does not fit, LoRA Isolation, the look warm-up and how many checkpoints to keep.

### Experimental: direct identity loss

`id_loss_weight > 0` (default 0 = off; the applied weight is capped at 0.5) adds a face-recognition term to the
diffusion loss. On every `id_loss_every`-th training sample with `sigma <= id_loss_sigma_max` (0.35) and a cached
face at least `id_loss_min_face_cells` (4) latent cells wide, the trainer forms the predicted clean latent
`x0 = noisy - sigma * pred`, crops the face cells plus `id_loss_margin_cells` (2) of context, decodes only that
crop with gradients (`decode_latents_grad`), aligns it to the 5-point face template with a differentiable affine
warp built from the cached landmarks, embeds it, and adds `weight * (1 - cos(embedding, reference))`. The
reference is one random face reference per step (`id_loss_random_ref`) or their mean embedding.

- Backends (`id_loss_backend`): `arcface` ports InsightFace's `buffalo_l/w600k_r50.onnx` to torch with
  `onnx2torch` (`pip install onnx2torch`; a self-check against the ONNX scorer on a reference face must reach
  cosine > 0.99, else the loss is disabled). This is the very network the face probe scores with, so **the probe
  metric becomes the training target** and can climb without a real likeness gain. `facenet` uses a vendored
  InceptionResnetV1 with VGGFace2 weights (downloaded once to `models/facenet/`) and keeps the probe an
  independent check.
- Needs `face_refs` (the reference identity) and builds the face cells / landmarks cache even without
  `region_weights` (person masks are skipped then). Runs on the primary GPU only (data parallel: the other
  replicas train on the diffusion loss alone).
- Guard: with probes on, if the smoothed face score falls more than 2 x `likeness_noise_band` below its best for
  two probes in a row, the controller vetoes the identity loss for the rest of the run (logged, `id_loss_vetoed`
  in the status).
- The Monitor loss chart shows the identity loss (smoothed, right axis); the step log reports its mean, sample
  count, seconds per sample and the peak VRAM.

Cost and effect, measured on an 8-image synthetic character set (512 px, base int8, rank 16, lr 2e-4, 80 steps,
one 16 GB card, arcface, weight 0.2; 22 of 80 steps had an eligible sample): the ArcFace port matched ONNX at
cosine 0.9998; the decoder runs as a bf16 copy with gradient checkpointing (an 18 x 18-cell face crop costs
~0.6 GB for forward + backward instead of 3.5 GB in fp32); peak VRAM 8.0 GB vs 6.05 GB without the loss (+2 GB
including the extra VAE and face network); 0.86 vs 0.80 s/it (+7%). The identity loss fell from 0.86 (steps
10-25) to 0.61 (27-43) to 0.47 (51-79). A control with the same draws and a negligible weight (1e-6, measured
only) fell 0.88 / 0.68 / 0.54, so most of the drop is ordinary learning and the loss adds roughly 0.07 on top in
this short run. Treat it as an experiment to A/B against the probes (with the facenet backend for an independent
score), not a default.

### Trigger attention and identity alignment (`attn_localize`, `id_align`)

Two training-only losses that read the model's insides and reuse what the trainer already has (the region maps'
person mask and face box, the face analysis); the saved LoRA is the same kind of file. Both are switches in the
Region-weighted loss section (*Trigger attention and identity alignment*) and in the Wizard's Training step for a
character, and race keys. Neither is measured for likeness yet: race them against the run's own settings.

- **Attention localization** (`attn_localize`, weight `attn_localize_weight` 0.05). In a few attention layers
  (`attn_localize_layers`; auto: a third and half of the depth, FIBO 2.0's injection blocks) every image token's
  attention over the caption is read (softmax over the text tokens, averaged over heads) and the share that goes to
  the trigger word's tokens forms a map over the picture. Normalized to sum 1, the loss is the part of that map off
  the subject: 1 - share on the person mask and face box. It binds the identity to the trigger and keeps it off the
  background and other people, the kind of positional supervision MultiCrafter (2025) put on FLUX's attention to stop
  attributes leaking between subjects. The trigger's tokens are found in each cached caption with the model's own
  tokenizer and prompt template (each dataset's trigger, Isolation's fused words, caption dropout's prompt); FIBO 2.0
  condenses the caption into Perceiver gist tokens, so it is read in the cross-attention of its injection blocks.
  Person masks are built for it even without the region-weighted loss. Needs a trigger word.
  In LoRA Isolation's side-by-side pairs (the subject next to a generated stranger) the subject's panel is the
  region, so the trigger's attention on the stranger counts against it: MultiCrafter's case of two subjects in one
  picture. The step log counts the pairs it saw.
- **Identity alignment** (`id_align`, weight `id_align_weight` 0.05, block `id_align_layer`; auto: a third of the
  depth). After that block, the image tokens inside the face box are averaged and a small MLP (about 7M weights on
  Krea 2, its own AdamW, never saved) maps them onto that photo's ArcFace embedding (stored with the face analysis);
  the loss is 1 - cosine. It is REPA's representation alignment (Yu et al. 2024) pointed at identity: feedback at
  every noise level, where the identity loss above needs a nearly clean prediction and a VAE decode.
- **Body alignment** (`body_align`, weight `body_align_weight` 0.05, the same block). The same for the build: the
  image tokens of the person outside the face box (the region maps' person mask) are averaged and a second small MLP
  maps them onto that photo's DINOv2 body-window embedding, the one the body probe scores with (the neck to
  mid-thigh, upright, cut out on grey; cached in `bodies.json`). Photos that do not show the window, or have no
  person mask, are left out; person masks are built for it even without the region-weighted loss.
- How it runs: the trainer arms taps on the model for a batch, the main forward records, nothing records on later
  forwards, and a checkpointed block that recomputes in the backward computes the same tensors again (the gradients
  match an unchecked run exactly on all three models). Tapped blocks run uncompiled. Data parallel: localization on
  every GPU, alignment on the first (like the identity loss). Not in the parallel race or a fine-tune.
- Checked on real models at 0.25 MP (16 GB card), both losses in one short run: Krea 2 (blocks 9 and 14; steps as
  long as without them; the trigger's attention off the subject 0.87 -> 0.84 over 10 steps), MOSS-Image (blocks 14
  and 21) and FIBO 2.0 (its five injection blocks). A Krea 2 race with a row for each trained, probed and dropped
  them like any other configuration. The trigger's tokens were found in every cached caption on all three.

## Likeness probes and controller (character LoRAs)

With face/body references set, `probe_every: N` renders a fixed-seed headshot (`probe_face_prompt`) and/or
full-body image (`probe_body_prompt`) every N steps at `probe_size` (the face square, the body a 2:3 portrait with the
same pixel count) with the turbo LoRA (cheap enough to do often), scores them, and records `likeness.jsonl` plus a Monitor chart. Each probe renders `probe_seeds` seeds and averages the scores; the step-0 probe, the zero
of the normalized scale, renders at least `probe_floor_seeds` (4). Scores are smoothed with `probe_ema`
and only changes beyond `likeness_noise_band` count. The last step is always probed too, whether or not
`probe_every` divides `steps`, so the final LoRA has its own score (it can be the best). The controller then, each individually switchable:

- keeps `<job>_best.safetensors` at the best smoothed combined score (`likeness_track_best`) and stops after
  `likeness_patience` probes without improvement;
- on their own switches (off by default), also keeps `<job>_best_face.safetensors` at the best smoothed face score
  (`likeness_track_best_face`) and `<job>_best_body.safetensors` at the best smoothed body score
  (`likeness_track_best_body`), whatever the other part and the combined score do then; the same noise-band rule
  decides a new best. In a race each configuration keeps its own (`<job>_<name>_best_face.safetensors`);
- by default replaces a best checkpoint only when the smoothed score beats it by more than the noise band, so a probe
  can log a smoothed score a little above `best` (35.1 against best 34.1 with the 1.0 band) without a new save. With
  *New best on any rise* (`likeness_best_any_rise`) any higher smoothed score replaces the overall, face and body
  best files. Either way early stop and the resolution step-up count only a rise beyond the band as progress;
- hill-climbs a bounded learning-rate multiplier (`likeness_lr_control`, `likeness_lr_min/max/step`): up while
  the smoothed score rises, down when it falls. It works with AdamW-style optimizers (through the scheduler)
  and with Automagic 3 (by scaling its adaptive lr tensors), which keeps doing its own per-step adaptation;
- applies a face/body curriculum (`likeness_curriculum`): when the face score lags the body score, close-up
  dataset images (largest face >= 3% of the image, classified once with the face detector) are sampled more
  often and the timestep distribution shifts toward lower noise where facial detail forms; the reverse when the
  body lags. Strength 0..1.

The scores never enter the loss; they steer data, lr and stopping. Use the same reference set and prompts for a
whole run so the series is comparable.

The face detector rescales every image to `face_det_size` and misses a face that fills the frame (a tight headshot
probe: its box spanned 87% of the image), so when it finds nothing the image is retried at the centre of a canvas
2x and 4x its size, where the same face is found with confidence 0.9.

**Body score** (`body_score`). `appearance` (the default): a DINOv2-base embedding of the *body window*, scored
like the face as the best cosine match against the references, in percent. The window is the body from the neck to
mid-thigh: MediaPipe's shoulders and hips set its position and scale (1.6 torso lengths wide, from 0.15 above the
shoulder line to 1.75 below it), the image is turned so the shoulder-to-hip line points straight down, and
everything outside MediaPipe's person mask turns grey. The same part of the body then sits the same way up and at
the same scale in every photo and probe, so lying, leaning and distant bodies compare with standing close ones, and
the background, the head (the face score's job) and the framing drop out. Only the reference photos whose window
lies at least 90% inside the frame are compared (close-ups have no hips, waist-up shots no thighs); with fewer than
two such photos every photo with shoulders and hips counts, with a warning. The window is letterboxed to a square,
not centre-cropped.
Measured on a real run (60 photos: 20 close-ups, 40 medium to full-body shots; Krea 2, probes every 60 steps to
step 960): the earlier input, the padded pose box through DINOv2's 224 px centre crop, kept half of a standing body
and was matched against the close-ups too, so the base model's stranger outscored the person's own photos against
each other (0.73 against 0.67) and the score had no relation to training (rank correlation -0.13 with the step).
The window against the 16 photos that show it rose with training (+0.94; most of the rise survives in greyscale,
so body shape and skin rather than lighting), from 0 to about half of the way to the references by step 960 while
the face reached them. Krea 2 draws a 2:3 "full body" probe from head to thigh, which the window covers. Build,
skin and hair count, and clothing too, so vary the outfits among the references. The model (350 MB,
`facebook/dinov2-base`) downloads on first use and runs on `body_device`. `proportions`: the older
match of skeleton ratios (shoulder/hip width, leg/height, ...) against the references' median, plus SAM 3
silhouette widths with `body_backend: mediapipe+sam3`; proportions in a photo shift with pose and camera angle, so
this score can fall while the image gets closer.

`skeleton3d` and `smplx` measure proportions so that a bent leg is still the same leg (`bodyshape.py`):

- `skeleton3d`: bone-length ratios from MediaPipe's 3D landmarks (its own depth estimate): shoulder / hip width,
  torso / leg, thigh / shin, upper arm / forearm, arm / torso, shoulder width / torso, head / shoulder width. The
  person mask's widths at the shoulder, waist, hip and thigh (and waist / hip) count only where the person stands
  upright, legs straight, facing the camera, head to feet in the frame: the one view where a width is a width.
- `smplx`: NLF (Neural Localizer Fields, Sarandi & Pons-Moll, NeurIPS 2024) fits the SMPL-X body model to the
  person; its 10 shape parameters rebuild the body in the model's T-pose (the model ships inside NLF's file), where
  it is measured: shoulder width, leg, torso and arm length, and the bust, waist, hip and thigh girths (the
  perimeter of the mesh's slice, as a tape measure takes it), all over the height, plus waist / hip. A picture
  needs the shoulders, hips and knees in the frame. NLF's shape prior is eased (`beta_regularizer` 1, NLF's default
  10 pulls every body toward the average). NLF's weights are for noncommercial research use and are not downloaded
  for you: put `nlf_l_multi_0.3.2.patch6.torchscript` (550 MB,
  [isarandi/nlf v0.3.2](https://github.com/isarandi/nlf/releases/tag/v0.3.2)) in `models/nlf/` or set `nlf_model`.
  About 0.35 s a picture on an RTX 4060 Ti; it waits on the CPU between probes and runs on `body_device`.

Both compare through a calibrated profile: per measurement the references' median and their own spread (a robust
standard deviation, at least 3% of the median), so a measurement that varies a lot between the photos of the one
person counts for little; the score is the mean of exp(-z^2 / 2) over the shared measurements, and the references'
own level is each reference against the others. Clothing, skin and nudity do not count, unlike `appearance`.

Measured on one woman's 19 photos (14 close-ups or selfies) and the 1,160 body probes of 32 MOSS-Image runs:

| Body score | Photos measured | References' own level | Stranger (step 0) | Rise with training | Noise (one seed) |
|---|---|---|---|---|---|
| `proportions` (2D) | 18 | 64.0 | **82.2** | +3.1 | 3.0 |
| `skeleton3d` | 18 | 55.5 | 58.7 | +6.1 | 3.8 |
| `smplx` | 5 (down to the knees) | 68.5 | **31.6** | **+15.8** | 5.1 |

The 2D ratios score a stranger above the person's own photos; the 3D bone ratios are steadier across her photos
(3-9% against 15-50% in 2D) but adults' bone proportions differ less than that, so a stranger still lands at her
level. SMPL-X separates them: her photos measure a fuller build than the base model's default woman (waist / height
0.52 against 0.38, bust 0.64 against 0.53, waist / hip 0.82 against 0.68), training moved the probes toward it
(the best run to 0.48, 0.61 and 0.76), and limb lengths matched everywhere (they carry no identity here). Five
photos are a thin profile, most of them sitting or kneeling: a few standing full-length photos make it sturdier.

**One scale for face and body** (`likeness_normalize`, on by default). Raw, the two scores are not comparable: the
face score is an ArcFace cosine (a stranger scores ~10-25, your references against each other ~60-80), the proportion
body score a match where every human already scores ~80 and the references ~90-96. A run's face score moves ~60
points, a proportion body score ~10, so a 0.7 / 0.3 average is really ~93% face and "the face lags the body" is true
all run.
Each score is therefore mapped first onto one scale, 0 = what a stranger scores and 100 = what a typical reference
scores against the others: the floor is the step-0 probe (the base model without the LoRA; `probe_at_start`,
rendered with at least `probe_floor_seeds` seeds, default 4, for a steadier zero) or a default (face 20, body 80),
the ceiling the median of each reference's best match among the others when two or more are given (the statistic
a probe is scored with; faces cut off by the frame left out), or a default (face 75, body 90). The ceiling is
raised to at least 10 face / 8 body points above the floor when the step-0 score is at or above it, and scores stop
at 100: matching the references better than a typical one of them is no more likeness (a probe that copies one
photo reads higher). The median replaced the mean because hard references (profile views, cut-off faces) pull the
mean below what a clean front-facing probe scores: on a real run a typical probe read 108% against the mean and
100% against the median.

**The face scale is stretched and curved** (`face_scale_span` 1.35, `face_scale_curve` 1.5; 1 and 1 give the plain
scale above). An ArcFace cosine saturates: a decent likeness already matches the references' best-match level, so the
face read ~90 and then sat at 100 while the pictures still improved, and read far above the body (on a real run face 90
against body 51 for pictures judged equally close). The face's 100 now sits `face_scale_span` times as far above the
stranger as the references do, and the face score is its position on that scale raised to `face_scale_curve`. A face
that read 90 now reads ~54, your references' own face level reads ~64, and the score keeps growing beyond it.

**The body scale is stretched and curved the same way** (`body_scale_span` 1.35, `body_scale_curve` 1.5, since v2.2.0).
With the face alone stretched the body read 100 while the face read 50-60: a DINOv2 body score saturates too, because a
probe that copies the photos' outfit and setting matches them better than the references, in their varied outfits and
poses, match each other, and where the references' own body match sits close to the base model's (a ceiling only
`MIN_SPAN` above the floor) a few points of gain read 100. The body's ceiling now also sits at least halfway from the
floor to a perfect match (a real run's 73.3 -> 81.3 becomes 73.3 -> 86.7 before the stretch). Now the references' own
level reads ~64 on both scales, a
reference-level body no longer outscores a reference-level face, and 100 is out of reach of a probe that only copies the
photos (raw 100 reads ~84 with an ordinary scale). Set both body numbers to 1 for the old body scale. The scale in force is kept in each probe record's `scale`,
with the curves; runs saved before these changes used the plain scales, so compare their face (before v2.0) and body
(before v2.2.0) numbers with care. The ComfyUI
nodes use the same scale.

The combined score, the noise band, the curriculum gap,
the lr control, the best checkpoint, early stop and the race drops all use the normalized values. The Monitor and the
Samples tab show only the normalized scores (the log and `likeness.jsonl` keep the raw face / body next to them as
`face`, `body`, `face_norm`, `body_norm`, `scale`); probe images carry their overall / face / body score underneath.
In a race, the Monitor and the Samples tab put the same generation (step, prompt or probe kind, seed) of every
configuration side by side in one row. Earlier runs' `likeness.jsonl` keep their raw combined scores.

**Export spreadsheet** (Monitor, *Likeness probes & controller*, during or after a run) downloads `<job>_likeness.xlsx`,
also written into the job folder: *Updated scores* has every probe's smoothed and per-probe face, body and combined
score and the best smoothed combined score with its step; *Raw probe scores* the raw similarities; *Scale* the run's
floors (the step-0 probe), its references' own levels, the span and curve of each scale, the weights and the smoothing.
The scores are Excel formulas of the raw scores and the Scale sheet, so editing a span, a curve or a weight rescores
the run (a run from before a scale change is shown on the span and curve its config has now). In a race each
configuration, and the winner after the race, gets its own pair of sheets and its own Scale column. Without the WebUI:
`python -c "from rulztrainer import likeness_export; likeness_export.export('output/<job>')"`. The Scale sheet also shows
where your references' own face and body level land on the chosen scales.

The same spreadsheet comes from **Previous Runs** too (*Export spreadsheet* under the table, one file per ticked run),
built from the raw probe scores the run history keeps, so it works after a later run has reused the job folder. Runs
saved before this version have those scores only if their job folder still holds the run: *Import old runs* (or
opening the tab) adds them then.

## Likeness-filtered self-training (`selftrain`)

Partway through the steps the LoRA renders the subject over varied prompts; the pictures that pass a quality filter
and look more like the person than the probes do train beside the photos, at a lower weight, for the rest of the run.
It is reward-ranked fine-tuning (RAFT, Dong et al. 2023: sample, rank by a reward, fine-tune on the best) with the
probes' likeness score as the reward, done inside the run: far cheaper than the reward stage below, but it learns only
from the good pictures, not away from the bad ones. At each step of `selftrain_at` (fractions of the steps, `0.5`;
`0.5,0.75` for two rounds):

1. it renders `selftrain_images` (32) pictures with the weight average, cycling through `selftrain_prompts` (empty: the
   reward stage's close-up, upper-body and full-body prompts from the trigger word), with the Previews' steps and
   guidance (`sample_steps`, `sample_cfg`) at the largest training resolution, on new seeds each round;
2. leaves out the pictures with several faces, no face (when the face counts), a face the detector is unsure of
   (below 0.7) or the frame cuts off, and the ones far blurrier than the round's other pictures (variance of the
   Laplacian below 0.4 x the round's median, at the training resolution; on the face when it is 64 px or larger there,
   since a render's soft background is not blur, else on the whole picture). Renders are compared with renders: real
   photos carry grain and texture a clean render never has, so against them every render looks soft;
3. scores the rest like the probes (normalized face and body, combined with `face_weight` / `body_weight`) and keeps
   those above both `selftrain_min_likeness` (60) and the probes' current smoothed score, best first, skipping
   near-copies of a kept one (32 px thumbnails correlating above 0.95), up to `selftrain_share` (0.2) of the real
   images (repeats counted);
4. trains on them with their prompt as the caption at `selftrain_weight` (0.5) in place of the last round's pictures,
   at every training resolution they are not upscaled to (region maps from their own face; no person masks).

The pictures, every candidate's scores and why it was left out are in `<job>/selftrain/` (`selftrain.json`); a resumed
run trains on with the last round's pictures. Not in a race. Training a model on its own output can amplify its own
artifacts: LoRAs retrained on their outputs over several rounds lost quality visibly by the third (*Model Collapse in
the Self-Consuming Chain of Diffusion Finetuning*, arXiv 2407.17493), which is why the rounds are few, replace each
other and stay a small share. The scorer that picks the pictures is the probes' own, so judge the samples by eye too.

## Seed soup (`soup_files`, `scripts/soup.py`)

Train the same subject two or three times with different seeds (or keep a race's finalists, their `_best` files),
then average what each changes in the model and save it as one LoRA: the "model soup" of Wortsman et al. (ICML 2022),
whose greedy soup of fine-tuned models beat the best single one. What the runs agree on, likely the likeness, stays;
what each learned on its own fades. Untested on this trainer: compare by eye too.

- **What is averaged** is each layer's weight change (up @ down x alpha / rank; LoKr: kron(w1, w2)), never the
  factors: two runs can reach the same change with entirely different factors. A layer one file does not change counts
  as zero for it. LoRA and LoKr files of the same model (ComfyUI, PEFT or LyCORIS keys); OFT and DoRA change the
  weights through the base weights, so they cannot be averaged from the files alone.
- **Rank** (`soup_rank`, 32): the average of three rank-16 LoRAs has rank up to 48. `0` saves it exactly (LoRA files
  only: the factors side by side); a rank keeps the directions the files share and drops the ones only one has (an
  SVD, the full fine-tune's extraction). A LoKr changes nearly every direction of a layer, so a low rank keeps little
  of it (16% at rank 32 for two LoKr factor-8 files on Krea 2): soup LoKr files at a high rank (128 or more), and
  watch the energy share the log reports; every file is also scored through the same rank, so the comparison stays
  fair.
- **Strength:** where the files disagree the average is smaller than each of them, so a soup can act like a lower LoRA
  strength. The strength is baked into the saved file.
- **The report** says how much the files agree (the mean cosine between their changes), the soup's size against
  theirs, and the share of the average kept at the rank.

Two ways to make one (the Train tab's **Seed soup** section has both):

- **Average now** (`python scripts/soup.py a.safetensors b.safetensors --out soup.safetensors --rank 32 [--weights
  2,1] [--strength 1.1]`): every file, at strength 1, no probes; saved as `<job>_soup_plain.safetensors` from the
  WebUI.
- **Picked by the probes** (`soup_files: [a, b, c]` with `steps: 0`; the WebUI's *Pick the soup with the probes*
  switch sets both): Start trains nothing. It probes the base model (the scale's zero), each file alone (the best one
  also at every `soup_strengths`, 1.0, 1.15, 1.3, so a soup is never credited for a strength change), then adds the
  others best-first, each at its best strength, keeping it only when the soup's probe score does not drop (the
  greedy soup), and also scores the plain average of all for comparison. Saves `<job>_soup.safetensors` and
  `soup/soup.json` (every trial's score, agreement and size). Uses the run's model, references and probe settings;
  more `probe_seeds` give a steadier choice.

Seed twins in one run: a race row may set its own `seed`, which gives it its own LoRA initialisation, image order and
noise (the other rows keep sharing theirs); with `race_drop_lowest: false` every twin trains to the end and keeps its
`_best` file.

## Likeness reward fine-tuning (`reward_steps`)

An optional second stage after the regular steps that trains on likeness itself, after Flow-GRPO (Liu et al., NeurIPS
2025), the online reinforcement-learning recipe for flow-matching models. Regular training teaches the LoRA to
rebuild the training photos; this stage asks, over prompts the photos never had, which of the LoRA's own pictures look
most like the person. Each reward step:

1. renders `reward_group` (4) pictures of one prompt from different seeds with a stochastic sampler: the usual sampler
   turned into an SDE with the same marginals (Flow-GRPO's step, noise level `reward_noise` 0.7), so every denoising
   move is a Gaussian with a known probability. Steps and guidance are the probes' (`reward_sample_steps` 0 =
   `probe_steps`, `probe_cfg`), with the turbo LoRA the probes use;
2. scores each with the probes' scorers on their scale, uncurved (0 = a stranger, 1 = the references; a missing
   face counts -1), each part kept within -0.5 .. 1.5, combined with `face_weight` / `body_weight`. Far below a
   stranger says nothing about likeness: it is a body in a coat against unclothed body references, or an upper-body
   shot against full-body ones. Uncapped, such a body score (its scale several times narrower than the face's)
   picked each group's winner, so the stage learned to look like the reference photos rather than the person. When
   half the first round's body scores are capped, the log says so: reward prompts like the body references (or
   `body_weight` 0) make the body count again;
3. turns the scores into advantages within the group ((score - mean) / std: only the differences between the
   pictures count, not the prompt's difficulty);
4. on `reward_train_steps` (4) of the denoising moves, one picture at a time, raises the log-probability of the moves
   behind the better pictures and lowers it for the worse (policy gradient), plus `reward_kl` (0.04) times the KL
   divergence to the LoRA as the stage started; AdamW at `reward_lr` (2e-5), gradients clipped at 1.

- **Starts from** the LoRA the steps saved (their weight average). `<job>.safetensors` stays the first stage's; the
  stage saves `<job>_reward_<step>` every `save_every` reward steps (pruned with `keep_checkpoints`) and
  `<job>_reward.safetensors`; its pictures are in `<job>/reward/`. Probes continue every `probe_every` reward steps:
  a probe that beats the best of both stages saves `<job>_reward_best.safetensors`, and early stop ends the stage.
  When the stage's last step was not probed (`probe_every` not dividing `reward_steps`, or larger than it), a closing
  probe scores the final LoRA, so the stage always ends with its likeness on the run's scale.
- **Measured cost:** Krea 2 (nvfp4, 16 GB RTX 4060 Ti), 4 pictures of 512 px at 4 denoising steps with guidance 5.5,
  2 trained moves per update: about 77 s per reward step (with a turbo LoRA at guidance 1 the guidance passes go).
- **Alone:** `steps: 0` with a LoRA in `resume_from` runs only this stage, for example on a race winner's file (it
  does not follow a race in the same run).
- **Prompts:** `reward_prompts`, or by default eight varied close-up, upper-body and full-body prompts with the
  trigger word (full-body prompts dropped when the body weighs nothing, close-ups when the face does). Full-body
  prompts render 2:3 like the body probe; LoRA Isolation's fused trigger applies.
- **Risk:** the scorers are not differentiable here, so the LoRA cannot chase their gradients pixel by pixel, but it
  can still learn what they like more than a person does (reward hacking). The KL penalty, the small learning rate and
  the probes on their own fixed seeds are the guards; look at the pictures.
- **Cost:** every step renders the group (`reward_group` x steps forward passes, x2 with guidance) and trains
  `reward_train_steps` x `reward_group` x 2 forward passes (the policy and its reference). Every block recomputes
  during the stage (kept activations are released). One GPU (data parallel: the first), no fine-tune.

## Previous runs (run history)

The **Previous Runs** tab keeps the likeness history of every run, like TensorBoard, so runs can be compared after the
job folder has been reused. A run with likeness scores writes one small file in `output/_history/` while it trains (a
crash keeps what was saved) and again when it ends. It keeps what the Monitor's two likeness charts plot, on the run's
normalized scale: the **mean face and mean body** score of the sample images at each sampling step (Identity
similarity), and the smoothed **face, body and combined** score at each probe (Likeness probes & controller). Runs
from before the history existed are imported from their job folders when the tab first loads, or with *Import old runs*.

Tick any number of runs in the table (name, start, state, steps, adapter, learning rate, last and best scores) to
overlay them: one colour per run, solid lines for the face, dashed for the body and thick for the combined score, with
a *Show lines (all runs)* bar of Face / Body / Combined chips above each chart: switch one off to hide that line of
every ticked run at once, and on again to bring it back. A race run draws each configuration's probe lines in its own colour, named
after it (`<run> lora16 - face`; the winner's after the race: `winner`); *Import old runs* adds that split to race runs
saved before it existed, while their job folder is still there. *Rename* gives a run a name of your own and *Remove from history* deletes
its history file only (never the job folder). *Export spreadsheet* downloads each ticked run's likeness spreadsheet (see
*Export spreadsheet* under the likeness probes), from the raw probe scores the history keeps for it.

**Compare on one scale** (above the *Likeness probes & controller* chart) rescores the ticked runs and the table on the
same face span / curve, body span / curve and face weight, instead of each run's own settings: runs trained with
different scale settings, or before a default changed, then compare fairly. Each run keeps its own floor (its step-0
probe) and its references' own level; the smoothing is its own too. *Recommended* sets face 1.1 / 3, body 1 / 1 and
80% face. Runs saved before v2.9.0 stay on their own scale unless their raw probe scores were added (*Import old
runs*, while the job folder still holds the run); a note names them. The probe images' captions keep each run's own
scores. A run that is still training updates live.

**Probe images** (below the charts) puts the ticked runs' likeness probes side by side: pick Face or Body and a probe
step (the arrows step through them), and each run, or each configuration of a race, shows its probe images of that
step in one row with their overall / face / body scores. A run probed on another interval shows its latest probe before
that step, marked. Click an image to see it full size; tick the box under any two (any run, any step) to open them
side by side, large, with their scores (*Compare* opens them again, *Clear picks* starts over). When the ticked runs
used different probe settings (prompts, seed, size, steps, guidance, turbo LoRA or resolutions) a note says which:
their probes are not the same test then. The images are kept with the run, as small JPEG copies in
`output/_history/<run id>/`, so a later run with the same job name does not replace them; *Remove from history* deletes
them too. Runs saved before v2.4.0 get theirs from *Import old runs* while their job folder still holds that same run.

## Configuration race (`race`)

Not sure which settings suit a dataset? `race: true` trains several configurations in one run and lets the likeness
probes pick. Each configuration trains until its next probe (`probe_every` steps), then the next one does; once all
of them have been probed (a round), the one with the lowest smoothed combined probe score is dropped.
`race_warmup_probes` rounds pass before the first drop. The last one keeps training until `steps`, counted in its
own steps, so the race costs extra steps: with N configurations, W warm-up rounds and a probe every P steps,
P x (W x N + N(N+1)/2 - 1) steps of racing before the winner continues alone (10 configurations, 2 warm-up rounds, a
probe every 100 steps and 3000 steps: about 9300 steps in all, 3.1x one run; the WebUI shows the total).

**Slow starters (`race_patience_drop`, `race_drop_lowest`).** Dropping the lowest each round cuts a setting that learns
slowly but would end higher. With `race_patience_drop: true` a configuration leaves as soon as its own probes have not
improved (by more than `likeness_noise_band`) for `race_patience` (3) probes in a row, whatever the others score, and
the rest race on. `race_drop_lowest: false` stops the round drops altogether: every configuration trains all the steps
(`steps` must be a multiple of `probe_every`) unless the patience drop ends it, at up to one full run per
configuration. Either way the winner is decided at the end: the configuration whose best smoothed score was the highest
at any point. Its best checkpoint becomes `<job>_best.safetensors` and its settings `<job>_winner.yaml`; the last one
racing trains on to the end with the run's own file names (`<job>.safetensors`), and the log says when that is not the
winner. A configuration that leaves on patience is saved as `<job>_<name>_dropped.safetensors`, one that finished its
steps as `<job>_<name>_final.safetensors`, and each keeps its `<job>_<name>_best.safetensors`. Both work with the
round drops on (`race_drop_lowest: true`), and in parallel races. `race_patience_rising: true` gives one a chance to
come back after a dip: out of patience, it stays as long as each probe's smoothed likeness is higher than its previous
probe's, and leaves at the first probe that is not (beating its best resets the count as usual). The log and Monitor
show who stayed.

**Relay: everyone continues from the round's winner (`race_follow: true`).** Nobody is dropped. After each round
(every configuration trained `probe_every` steps and was probed) the round's winner, the highest smoothed combined
score, hands everything it learned to the others: its LoRA weights, optimizer state, weight average and likeness
history (best, smoothing, patience). Each keeps its own settings, seed, image order, noise, learning-rate schedule and
likeness lr factor, and trains the next round from there. Rows that differ only in their seed make this the best of
several random paths at every probe, instead of whole runs on different seeds: the WebUI's **Add seed racers** button
(seeds `1-4`, or `1, 5, 9`) adds one `seed=` row per seed and turns the relay on. Every configuration trains all the
steps (`steps` must be a multiple of `probe_every`): N configurations cost N runs' steps. The last round's winner is the
run's LoRA (`<job>.safetensors`, its settings `<job>_winner.yaml`), the others are saved as
`<job>_<name>_final.safetensors`, and since they share one lineage, the best checkpoint any of them reached is
`<job>_best.safetensors`. Rows must share the LoRA's shape (no `lora_rank`, `lora_alpha`, targets, blocks, network type
or DoRA rows; `lora_dropout` is fine), the drop and patience rules are ignored, and it does not run with
`race_parallel`. A configuration on another kind of optimizer than the winner's starts its optimizer state over (the
log says so). With 4 seeds the round's best of 4 probes is somewhat lucky: the probe noise (a few points) is part of
what picks the winner, so more probe seeds (`probe_seeds`) make the choice steadier.

**One configuration per GPU (`race_parallel: true`).** With `multi_gpu: data_parallel` on two or more GPUs, the
configurations of a round train side by side instead of in turn: each GPU's model copy trains its own configuration
at the same moment, both run the same `probe_every` steps, and each is probed at the end of its turn as before. With
an odd number left, the last one of the round trains alone over every GPU when its step has batches enough
(`gradient_accumulation` 2 or more, the batches split by the GPUs' measured speeds) and that is faster than the first
GPU alone (by 5%), else on the first GPU only. The winner trains on the same way. Every configuration sees
`batch_size` x `gradient_accumulation` images per step, wherever it runs, so the comparison stays fair (plain data
parallel gives each step that many images on every GPU). Each configuration owns its parameters, optimizer and weight
average, parked on the CPU while it waits; the model copies carry the weights of whoever trains there. Not with the
identity loss; `torch.compile` is off; no training states. Manual drops, the gallery and the race table work as before
(the table marks every configuration that is training). A three-configuration test on an RTX 5060 Ti and an RTX 4060
Ti ran two at once at about 6.6 s per step each, against about 13 s one after the other.

**On the Monitor.** The *Likeness probes* chart draws every configuration's smoothed face, body and combined scores
as its own lines, named after it (`lora16 face`, `lokr8 combined`): one colour per configuration, the combined score
thick, the body dashed; after the race the winner's go on as `winner`.

**Dropping one by hand.** The Monitor's race table has a *Drop* button on every configuration still in the race (after
a confirmation). The trainer reads the request after the current step: the configuration stops, its LoRA is saved as
`<job>_<name>_dropped.safetensors`, and the race goes on without it. If it was the one training, the next one takes
over at once; a round it was the last of to be probed ends with this drop instead of dropping the lowest score as well;
with one left, that one wins and trains on to `steps` like any winner. Without the WebUI, put the configuration's name
in a file `RACE_DROP` in the job folder (one name per line); the trainer reads and removes it.

```yaml
race: true
race_warmup_probes: 1
race_configs:
  - name: base                     # the run's own settings
  - name: lr2e-4
    learning_rate: 2.0e-4
  - name: rank32_tf
    lora_rank: 32
    lora_alpha: 32
    lora_targets: [q, k, v, o, ffn_up_gate, ffn_down, text_fusion]
  - name: identity_ts
    timestep_sampling: identity
    lora_ema: 0
```

A configuration may change the learning rate and optimizer settings, the timestep sampling and loss settings
(including region weights, the identity loss and the curriculum), the LoRA rank, alpha, targets, blocks and dropout,
caption dropout, flips and crop augmentation, and LoRA Isolation's DOP weight and interval when the run has isolation
on (the full list is `race.RACE_KEYS`, also under *What a configuration may change* in the WebUI); the model,
quantization, GPUs, dataset, resolutions, probes, batch size and the training adapter are shared: they are set up once
for every configuration (images, batches, model copies), so they cannot differ per row. The comparison is kept fair: every configuration starts from the same
LoRA initialisation for its shape, sees the same images in the same order with the same noise, and is probed at the
same own step with the same seeds. The trainer keeps one set of LoRA wrappers over every configuration's layers;
switching parks the outgoing configuration's weights, optimizer state and weight average on the CPU, so only the
training one takes GPU memory. Configurations that differ in region weights get their own region maps in the cache.

While the race runs, a configuration's files carry its name (`<job>_<name>_<step>.safetensors`,
`<job>_<name>_best.safetensors`, and samples and probe images prefixed with it). A dropped configuration is saved as
`<job>_<name>_dropped.safetensors` (its weight average). The winner continues under the run's own file names, its
best file is copied to `<job>_best.safetensors`, and `<job>_winner.yaml` holds its full settings (resume or retrain
from it). `race.json` and the Monitor tab show every configuration's score per round. Training states are saved only
after the race, and a race cannot resume or step up resolutions.

## LoKr (`network_type: lokr`)

Instead of a rank-r LoRA pair, each layer learns a Kronecker pair: `w1` [a, b] with a, b the largest divisors of the
layer's output / input size that are at most `lokr_factor`, and `w2` [out / a, in / b] full-matrix; the update is
kron(w1, w2), applied without materializing it. w1 starts kaiming-uniform and w2 at zero, so the delta is zero at
step 0 and both factors get gradients from step 1 (the LyCORIS / Fizgig conventions). Factor 8 is Fizgig's measured
default ("the highest likeness we've ever measured, with noticeably more natural skin than standard LoRA on the same
data"; ~20% slower steps; files ~1/64 of the weights, about 400 MB on a 12B model; no quality gain above 8).

- `lora_rank`, `lora_alpha`, `lora_plus_ratio` and `lora_dropout` are LoRA-only; a LoKr layer's scale is 1.
- Saved keys are LyCORIS: `<stem>.lokr_w1`, `.lokr_w2`, `.alpha` (1.0), with the family's usual stems
  (`diffusion_model.<module>` for MOSS-Image, `transformer.<module>` for Krea 2 and FIBO). ComfyUI's LoRA loader
  takes both for MOSS-Image and Krea 2 (it maps the diffusers stems); diffusers has no LoKr loader, so a FIBO 2.0
  LoKr loads only back into RulzTrainer. The metadata carries `ss_network_module: lycoris.kohya` and
  `ss_network_args: {"algo": "lokr", "factor": N}`.
- A LoKr file (Fizgig's, ai-toolkit's, LyCORIS) also works as a sampling-only `turbo_lora` / `sample_lora`;
  low-rank-stored factors (`lokr_w2_a/b`) are rebuilt, Tucker (`lokr_t2`) is not.
- EMA, the configuration race (one row `network_type=lokr, lokr_factor=8` against a LoRA row trains both on the same
  layers and lets the probes pick), the sampling model and data parallel all handle it.

## OFT (`network_type: oft`) and DoRA (`dora`)

Two more adapters to race against LoRA and LoKr. Both save files ComfyUI's LoRA loader applies as they were trained:
its own weight-adapter code, run on the saved keys, reproduces the training forward to about 1e-7 (plain and int8
weights, with and without the OFT constraint). diffusers has no loader for either.

- **OFT** (orthogonal fine-tuning, LyCORIS diag-OFT). Instead of adding to a layer it rotates the layer's output
  channels in blocks of `oft_block_size` (32; the largest divisor of each layer's outputs up to it), with an
  orthogonal R from the Cayley transform of a skew-symmetric matrix, so the base model's structure (its weights'
  norms and angles) is kept. The OFT paper reported better subject identity than LoRA (DINO 0.632 vs 0.613 on
  DreamBooth objects with Stable Diffusion) and stable results over long runs where LoRA drifted. `oft_constraint`
  > 0 (COFT) caps how far each layer rotates. Blocks start at zero (R = I). The rotation is applied to the frozen
  layer's output, so it trains on int8 / nvfp4 weights as they are. Keys: `<stem>.oft_blocks` [blocks, size, size]
  and, with a constraint, `<stem>.alpha` (ComfyUI reads it as that cap). Rank, alpha, LoRA+ and DoRA do not apply.
- **DoRA** (weight-decomposed LoRA, on a LoRA or a LoKr). The merged weight is W' = (W + delta) * m / ||W + delta||
  per input column: the adapter steers the direction, and a magnitude m per input channel trains on its own (it
  starts at ||W||, so the layer is unchanged at step 0). The norm is computed every forward from the dequantized
  weight, without gradient (the DoRA paper's memory saving), so steps cost more. Saved as `<stem>.dora_scale` [1, in]
  next to the LoRA / LoKr keys. The input axis is deliberate: ComfyUI's `weight_decompose` normalizes the merged
  weight on that axis, but on the output axis it divides by the original weight's norm, which would not match what
  DoRA trains. A plain LoRA resumed with `dora: true` starts its magnitude where the loaded weights leave it.
- In a race: rows like `network_type=oft, oft_block_size=32` or `dora=true` next to a LoKr row train on the same
  layers from the same start, and the probes pick. EMA, the sampling model, data parallel, `resume_from` and the
  race's saved files all handle both.
- Measured cost, Krea 2 nvfp4 at 0.25 MP on the 16 GB RTX 4060 Ti (12-step runs, steady step time; likeness not yet
  compared on a full run):

  | Adapter | Trainable weights | Step | Blocks kept from the recompute |
  |---|---|---|---|
  | LoKr 8 | 195 M | 3.5 s | 2 |
  | OFT block 32 | 58 M | 2.6 s | 4 |
  | LoKr 8 + DoRA | 197 M | 4.9 s | 1 |

  DoRA's extra time is the norm from the full (dequantized) weight in every forward and its recompute; while
  samples and probes render it is computed once per layer per round (a probe: 14-15 s, as LoKr's). ComfyUI's own
  loader (a 2026-10-20 build) read every key of the three race files above and merged them without errors.

## LoRA Isolation (`isolation`)

A character LoRA spreads its subject onto other people: strangers in the background, a friend beside them, a crowd.
`isolation: true` keeps the subject to its trigger, after Fizgig's LoRA Isolation. Fizgig measured it on Krea 2, two
people in one LoRA, 50 split pictures of a carefully described stranger next to the subject, each face scored with
ArcFace:

| | Normal LoRA | LoRA Isolation |
|---|---|---|
| Strangers that read as the subject | 48 of 50 | 0 of 50 |
| How much the subject looks like themselves | 0.77 | 0.78 |

Unmeasured here. Its parts:

1. **DOP** (Differential Output Preservation, from Ostris's AI-Toolkit). Every `dop_every` steps (2), each picture whose
   caption names a subject also trains on a freshly noised copy, with the subject swapped for its class phrase,
   held to what the model predicts there with the LoRA at 0, at `dop_weight` (2). Without the trigger the LoRA
   learns to change nothing. The log shows the DOP loss.
2. **Several subjects**, one per dataset, each with its own trigger (the Dataset rows' *Trigger*, or `trigger:` on a
   `datasets` entry). Two faces learned side by side teach the LoRA that a face belongs to one trigger, not to the
   class, so a second person helps; Fizgig recommends one even for a LoRA of one person. `isolation_balance` repeats
   smaller subject datasets so each trains about as often as the largest.
3. **The fused trigger.** `gwelt, a man` trains as `gweltman` (the caption files are never changed). The plain " man"
   token is shared by every other man; joined to the trigger it becomes a different token, so the subject's look
   does not attach to it. **Prompt with the joined word** (`gweltman at a cafe`); the sample and probe prompts are
   rewritten for you, and the log names the word. The class comes from the dataset's *Is a* (`class:`), else from the
   captions' "<trigger>, a [young] man". When the tokenizer splits the joined word so the class word is no longer
   a whole token (`zk | re | ab | oy`), the log warns: a trigger ending in a consonant cluster joins better.
4. **Side-by-side pairs.** Each person gets `isolation_pairs` (16) 1024 x 512 pictures: one of their photos, cropped
   around the face, next to one of 32 generated strangers (8 ethnicities, men and women, 20s to 50s;
   ShootTheSound/fizgig-multi-person-strangers, CC BY 4.0, pinned, downloaded once into `models/`), side at random,
   captioned stranger first: `an Indian man in his 30s on the left, and gweltman on the right, two people side by
   side, two close-up portraits`. The text encoders are causal, so words before the trigger stay clean of it. They
   train at `isolation_pair_weight` (2) times the loss and are never flipped (left and right are in the caption).
5. **What a subject is** (*Kind*, `kind:`). A person gets the fused trigger and the pairs. An animal or thing
   (`rexodog`) gets the fused trigger only. A style keeps its trigger as written and drops it from the trigger-free
   caption, so the look appears only when called.

A single `dataset_dir` uses `trigger_word`, `isolation_class` and `isolation_kind`. A dataset without any trigger
trains as it is, not as a subject. In a short Krea 2 test on a 16 GB card a step with a DOP pass took about twice as
long as one without (an extra forward without gradients and one with); Fizgig reports about 1.5x over a run. Not with
a fine-tune or a `race_parallel` race.

## Full fine-tune, then extract a LoRA (`finetune`)

A LoRA confines every update to a few directions, so concepts compete for them and pull pose, framing and lighting
along with the likeness. `finetune: true` trains the target layers' own weights instead (after Fizgig's fine-tune).
Fizgig measured on Krea 2: a LoRA *extracted* from a fine-tune beat one *trained* at the same rank on the same
dataset, and rank 64 was indistinguishable from the full 26 GB checkpoint at about 0.5 GB. The full-rank phase does
the work; extraction is one SVD per layer. Unmeasured here.

**How it fits one card.** The model stays as it was loaded (its int8 / nvfp4 trunk, or bf16). The target layers (the
same `lora_targets` / `lora_blocks` / `train_text_embedder` as a LoRA's) are grouped into *components*, one Linear
name across every block (every block's attention query, say), and the components are packed into *windows* of what
the card holds. One window at a time is swapped up to fp32 weights (gradients and optimizer state only for that
window; the forward runs in bf16), trained for `finetune_rotate_epochs` epochs, and swapped back: its update is kept
as a bf16 *delta* in RAM, at the delta's own precision, so a small update is never rounded away, and the trunk's
layer is re-quantized from original + delta. A *cycle* of windows trains every component once. The original
quantized weight of every trained layer stays in RAM too, so each delta is always against the exact base the model
loaded with and the trunk never accumulates re-quantization error.

- **Held near the original** (`finetune_anchor`, 0.01). After every step the trained weights move
  lr x `finetune_anchor` of the way back toward the original model's: w -= lr x anchor x (w - original), decoupled like
  AdamW's weight decay. This is the bounded drift of "Preserve and Personalize" (Kim et al., ICLR 2026), whose full
  fine-tune of Stable Diffusion 1.5 held near its base this way scored above an unheld one in likeness (DINO 0.639 vs
  0.603) and prompt-following (CLIP-T 0.298 vs 0.279). The optimizer's own `weight_decay` is off for a fine-tune: it
  would shrink the model's weights themselves toward zero (it did before v1.9.0). The original weights of the window
  that trains stay on the GPU in their own format (1 byte per weight for int8, 2 for bf16; the window plan counts 2).
  0 turns it off.
- **Windows.** Planned at launch from the free VRAM after the model loads, less an activation reserve
  (`finetune_reserve_gb`, default 3 GB + 3 GB per megapixel), at 10 bytes per weight with adafactor (fp32 weight,
  gradient, bf16 copy; 18 with adamw). A component larger than a window is cut into block ranges. `finetune_window_gb`
  sets the window size by hand. The log lists the plan.
- **Measured here.** MOSS-Image int8 on a 16 GB RTX 4060 Ti at 0.25 MP, every target, adafactor: 4.76 B weights in 10
  windows (q, k, v, o whole; up-gate in 4 block ranges; down in 2), 13.1 GB peak, about 1.4 s per step. RAM: about
  2 bytes per trained weight for the deltas plus the original quantized copy (MOSS: 10 + 5 GB; Krea 2: 24 + 12 GB).
- **Saves** extract a LoRA per rank in `finetune_extract_ranks` (default `[64]`; a second rank gets `_r<rank>`), in
  the family's usual LoRA format with alpha = rank, at cycle boundaries (`save_every` waits for the cycle to end, so
  every saved file has each component trained alike; the best-likeness checkpoint extracts at once). Samples and
  probes render the fine-tuned model itself, with the turbo LoRA as usual; `sample_model_path` is ignored.
- **The deltas.** `finetune_save_delta: true` also writes `<job>_finetune.safetensors` (bf16, 2 bytes per weight: 9.5
  GB for all of MOSS): the fine-tune's state. `finetune_resume: <that file>` continues from it, and
  `python -m rulztrainer.extract_finetune <that file> --rank 32 64 128` extracts at other ranks without the model.
- **Learning rate.** The weights themselves move, not an adapter: Fizgig measured 1e-5 on Krea 2 (1e-4 trains but
  overcooks) and 3e-5 for larger-weight models. The WebUI's switch sets 1e-5 with adafactor and warns above 5e-5.
  Runs are long: one cycle is one true epoch of the whole model.
- **Not with** a race, data parallel (`model_split` is fine), torch.compile, the weight average, training states,
  `resume_from` or the region prior; rank, alpha, the adapter kind, LoRA+ and dropout do not apply.

## Sampling with a different model

`sample_model_path` (a `.safetensors` file, a diffusers folder or a repo id) plus `sample_model_quant` use another
checkpoint for sample images only, for example training on Base while previewing on SFT. The current LoRA weights
are copied onto it before each sampling pass, the turbo LoRA (if any) is applied there, and the training model is
moved to system RAM for the duration (so RAM must hold both models; quantize the sampling model if tight).

## Turbo LoRA for samples

Samples and probes render with a turbo (step-distilled) LoRA by default: `turbo_lora: auto` downloads the model
family's own and the sampling defaults are the ones it is made for. It is applied only while generating sample and
probe images, never during training steps.

| Family | Turbo LoRA | Steps | CFG | Shift |
|---|---|---|---|---|
| MOSS-Image | lodestones' `8-steps-dmd-lora` ([lodestones/Kromoss](https://huggingface.co/lodestones/Kromoss)), Fizgig's pick, distilled from Base | 12 | 1 | 5 |
| Krea 2 | `krea2_turbo_lora_rank_64_bf16` ([Comfy-Org/Krea-2](https://huggingface.co/Comfy-Org/Krea-2)) | 8 | 1 | from the image size |
| FIBO 2.0 | `fibo_2.0_turbo_lora_avg_rank_270_bf16` ([Comfy-Org/Fibo-2.0](https://huggingface.co/Comfy-Org/Fibo-2.0)): Edit Turbo as a LoRA on dev | 4 | 1 | from the image size |
| Qwen-Image 2.1 | `qwen_image_2.1_turbo_lora_avg_rank_178_bf16` ([Comfy-Org/Qwen-Image-2.1](https://huggingface.co/Comfy-Org/Qwen-Image-2.1)), extracted from Qwen-Image-2.1-Turbo | 8 | 1 | the Turbo checkpoint's own sigmas |

MOSS-Image renders 12 steps rather than the card's 8 because Fizgig found 12 much better; Fizgig also found that
its turbo LoRA can understate likeness, so judge a run's likeness on full-quality samples (`turbo_lora: ""`, 20 steps,
CFG 3). The WebUI's *Turbo LoRA* choice (the model's own, none, or a file) moves the steps and CFG between the turbo
and the full-quality defaults while they are still at a default. `turbo_lora` can also be a path to any
ComfyUI-format LoRA with `turbo_lora_strength`. `sample_lora` + `sample_lora_strength` stack a second sampling-only
LoRA (for example a style LoRA) alongside the turbo one.
LoRA layers (`lora_down/up`, `lora_A/B`), `diff_b` bias offsets and `diff` offsets of normalization weights (swapped
in only while sampling) are applied; Krea 2 LoRAs may use ComfyUI's module names (`blocks.N.attn.wq`) or ours, and a
FIBO 2.0 LoRA in ComfyUI's fused layout (Comfy-Org's Turbo LoRA) is split back into the model's own, and so is a
Qwen-Image 2.1 LoRA on ComfyUI's fused feed-forward (`img_mlp.gate_up`, Comfy-Org's Turbo LoRA). LoKr / LoHa /
full-weight diffs of linear layers cannot be applied while sampling and are named in a warning. With a turbo LoRA,
Krea 2 samples on its distilled schedule (fixed shift, as ComfyUI does), and the log warns about guidance other than
CFG 1.

**Sample size.** The WebUI picks the sample size as an aspect ratio (1:1, 4:5, 3:4, 2:3, 9:16 and the landscape ones)
and an area in megapixels (0.25 to 4, 1 MP = 1024 x 1024); it is stored as `sample_width` / `sample_height`,
multiples of 16 (2:3 at 1 MP: 832 x 1248).

Krea 2's previews and probes decode in 512 px VAE tiles: the Qwen-Image VAE normalizes per pixel, so the tiles give
the full decode to 0.06 of an 8-bit level (PSNR 68 dB), at +2.0 GB instead of +7.9 GB per megapixel. The Flux2 VAE of
MOSS-Image and FIBO is not tiled (its GroupNorm and attention see the whole image: tiles would shift tones by 4-7
levels).

## Dataset page

Caption dropout (`caption_dropout`, 5% of batches) trains on the trigger word alone rather than the empty prompt
(`caption_dropout_keep_trigger`, on by default, as ai-toolkit does): the trigger by itself then summons the subject,
which is how a character LoRA is usually prompted. Off, a dropped caption is the empty prompt (FIBO: its null
caption), the base model's unconditional input.

The WebUI's Dataset tab lists every dataset as a card with a few thumbnails: the folders in `datasets/` next to
the WebUI (where *Create dataset* makes new ones; git-ignored) and folders elsewhere added with *Add a folder*
(remembered in `datasets/.rulz_datasets.json`; *Remove from list* forgets one without touching its files). Click a
dataset to open it:

- *Upload* sends images and their `.txt` or `.json` captions from the computer the browser runs on to the machine
  running RulzTrainer, a vast.ai instance included; on the same machine that is a plain copy. Files keep their names
  and a file with the same name is replaced.
- *Copy from a folder* copies the images and captions of a folder on the machine running RulzTrainer into the dataset,
  sub-folders kept; files the dataset already has are left alone.
- The grid shows every picture with its caption, saved to the `.txt` file as you type, and the *Face ref* / *Body ref*
  ticks; *Train on this dataset* ticks the dataset for training.

**Look consistency filter** (after Fizgig's), in the opened dataset: finds the pictures whose face drifts from the
look you want.
- **Baselines.** Tick *Baseline* on a few pictures that nail the look; 3 is a good number, since one photo would bake
  its own angle and light into every score.
- **Scan.** Every picture's largest face is embedded with the face model the likeness probes use (InsightFace ArcFace)
  and scored by its mean cosine similarity to the baselines' faces, worst first.
  - 0.45 and up is a solid match.
  - 0.30 to 0.45 is the same person, drifting.
  - Under 0.30 is likely off-look or someone else.
  - Pictures without a face get no score; judge those by eye.
- **Auto-suggest** marks the pictures under the fence: the median minus 1.5 times the interquartile range of the scores,
  never under 0.25, and only with at least 4 faces. Baselines and faceless pictures are never marked.
- **Move marked out** moves the marked pictures and their captions into `excluded_by_look/` inside the dataset, keeping
  sub-folders. That folder never trains and nothing is deleted. *Move all back* restores them.
- The scan is saved with the dataset as `look_scores.json`. Each picture in the grid shows its look score.

**Body outliers.** The *Body outliers* switch above the buttons runs the same filter on the body. It has its own
baselines (tick pictures that show the body you want), scan, marks and file (`look_scores_body.json`). The score is the
cosine similarity of the picture's body window (the trainer's body score: neck to mid-thigh, turned upright, the person
cut out on grey, embedded with DINOv2) to the baselines' windows, so a different build, skin, hair or outfit shows, and
so does an unusual pose or a rear view when the baselines are all front views. DINOv2 cosines sit high and close
together, so there are no ArcFace-style 0.45 / 0.30 rules: the colours read off the scan's median and fence, and the fence
has no floor. A picture without shoulders and hips in the frame, or with under 60% of its window inside it (a
portrait), gets no score. It runs on the *Body score device* (Identity similarity) and downloads the pose model and DINOv2
once, as the trainer does. On the 60 AZav50 photos, 22 had a usable window; with three front views as baselines the
fence flagged a pose lying on sand and two rear views. *Warm up look outliers* eases in the outliers of both scans.

On a real 60-photo set with three front-facing baselines, the scan took about a minute on the CPU. The median match was
0.60 and the fence 0.36. It flagged an extreme close-up with the eyes cropped off and a small, distant face turned away:
real shots, just unusual.

**Warm up look outliers** (`look_warmup`, Train tab, Per-image loss watch section; off by default) is for such shots when
you keep them. The pictures under the scan's fence start at a loss weight of `look_warmup_start` (0.4) and reach full
weight after `look_warmup_epochs` (4) epochs of their dataset, while the identity forms from the typical pictures. It
needs a scan of the dataset (Fizgig's `fizgig_look_scores.json` is read too), works without the loss watch, and a race
configuration may switch it.

**Several datasets per run.** On the Train tab (Dataset section) tick the datasets a run trains on. Each can train at
its own resolutions (none picked: the run's *Training resolutions*) and has *Repeats*, how many times each of its
images comes up per epoch (default 1): a dataset at 2 counts twice as much as one at 1, which weights a small or
important set up. In YAML:

```yaml
datasets:
  - {path: datasets/anna_portraits}
  - {path: datasets/anna_full_body, resolutions: [0.5, 1.0], repeats: 2}
```

The run trains at every resolution any dataset uses (resolution step-up steps through all of them; a dataset joins
the stages at its own resolutions). Each dataset caches its latents and region maps in its own `.rulz_cache`, so it
trains again without re-encoding whatever it is combined with; text embeddings, the face analysis and loss-watch
exclusions live in the first dataset's cache (or `cache_dir`, which then holds everything). A config with a single
`dataset_dir` and no `datasets` trains that folder as before; listing a dataset twice, or one inside another (whose
scan includes its sub-folders), is refused.

## FIBO 2.0 (Bria): training on dev, using the LoRA on Edit Turbo

RulzTrainer also trains LoRAs for [briaai/Fibo-2.0-dev](https://huggingface.co/briaai/Fibo-2.0-dev), Bria's 8B
single-stream DiT (44 blocks, Qwen3-VL-4B text encoder, the same Flux2 VAE as MOSS-Image). Pick **FIBO-2.0-dev** as
the model, or any diffusers folder / repo id whose transformer is a `BriaFibo2Transformer2DModel` (the architecture is
detected from the model; `model_arch: auto`). The repo is gated: accept its terms on Hugging Face and `hf auth login`.

- **Train on dev, use on Edit Turbo.** [Fibo-2.0-edit-turbo](https://huggingface.co/briaai/Fibo-2.0-edit-turbo) is
  distilled from dev with a byte-identical transformer config, so a LoRA trained on dev loads onto it unchanged. Bria
  says Turbo itself is not meant for fine-tuning.
- **Captions** are plain `.txt` sentences or structured VGL `.json` captions next to each image (the `.json` wins). A
  JSON caption is cleaned the way the model was trained (empty values and score keys dropped, compact form), and the
  trigger word goes into its `short_description`. Caption dropout trains on FIBO's null caption with the trigger in
  its `short_description` (`caption_dropout_keep_trigger`), or on the null caption itself.
- **Memory.** The bf16 transformer is 16 GB, so `transformer_quant: auto` quantizes it to int8 (W8A8) on GPUs under
  22 GB; set `none` for bf16 on a larger card. The text encoder (8.9 GB bf16) is loaded for caching only and unloaded
  afterwards (`text_encoder_quant: int8` halves it). Measured on a 16 GB RTX 4060 Ti with int8 + W8A8, rank 16 on
  the six block groups, gradient checkpointing, batch 1:

  | Resolution | Peak training memory | Step |
  |---|---|---|
  | 512 px | 8.9 GB | 2.6 s |
  | 768 px | 9.7 GB | 5.7 s |
  | 1024 px (1 MP) | 11.0 GB | 10.9 s |

  Loading and quantizing the transformer takes about 30 s; a 512 px preview of 12 steps with CFG takes about 10 s.
- **LoRA targets.** The six usual groups map onto FIBO's `to_q/k/v`, `to_out`, `w1/w3` and `w2`; FIBO adds
  `cross_attn` (the five text-injection blocks), `adaln`, `refiner`, `perceiver` and `embed`. Bria's own trainer adapts
  every Linear layer (all of these); the default here is the six block groups.
- **Timesteps.** `timestep_sampling: reference` uses Bria's recipe (sigma ~ sigmoid(N(shift, 1)) with the shift set by
  the image size, 10% uniform); the other modes work as for MOSS. Previews use the pipeline's schedule (size-shifted
  sigmas, CFG as a second pass with the null caption); Bria samples dev at 50 steps / CFG 5, 20-30 steps is fine for
  previews. The MOSS-only sample shift and skip layer are ignored.
- **Output format.** FIBO LoRAs are saved in the diffusers / PEFT format (`transformer.<module>.lora_A/B.weight` plus
  the adapter config in the file metadata), since ComfyUI has no FIBO 2.0 support yet:

  ```python
  import torch
  from diffusers import BriaFibo2Pipeline  # diffusers main (PR #14991)
  pipe = BriaFibo2Pipeline.from_pretrained("briaai/Fibo-2.0-edit-turbo", dtype=torch.bfloat16).to("cuda")
  pipe.transformer.load_lora_adapter("output/<job>", weight_name="<job>.safetensors")  # the pipeline has no load_lora_weights
  edited = pipe(prompt=vgl(scene), image=source, num_inference_steps=4, guidance_scale=1.0).images[0]
  ```

- **Not yet:** edit-pair training (source + target + instruction) and the `text_render` text images. Single files
  and the turbo LoRA for previews come from Comfy-Org (see [Model sources](#model-sources-and-quantization) and
  [Turbo LoRA for samples](#turbo-lora-for-samples)).

**Loading the LoRA in diffusers.** `BriaFibo2Pipeline` has no `load_lora_weights`; load through the transformer as
above. On a 16 GB card the bf16 transformer needs offloading (`pipe.enable_sequential_cpu_offload()`): load the LoRA
*before* enabling it. Loaded afterwards, the adapter weights miss the offload hooks and the output is noise or black
(diffusers' pipeline-level loader works around this; the transformer-level one does not).

**Verified** (12 photos of one person, 512 px, rank 16, lr 2e-4, `reference` timesteps, int8, one RTX 4060 Ti):

| | Face likeness (ArcFace vs 3 reference photos) |
|---|---|
| Probes during training, steps 0 / 50 / 100 / 150 / 200 / 250 | 13.5 / 36 / 46 / 58 / 58 / 61 % |
| diffusers `BriaFibo2Pipeline` on **dev**, text-to-image, base vs LoRA | 15 % vs **65 %** |
| diffusers on **Edit Turbo**, re-pose edit of another person's photo, base vs LoRA | 12 % vs 22 % |
| diffusers on **Edit Turbo**, "make the person smile" edit, base vs LoRA | 9 % vs 14 % |

The LoRA file loads into diffusers' transformer without warnings, and with the same LoRA the diffusers model and
RulzTrainer's produce bit-identical outputs. Note what an edit does, though: Edit Turbo conditions on the source
image, so a text-to-image LoRA shifts an edit toward the trained identity only partly; it does not replace the face
in a photo of someone else. Replacing identities in edits needs edit-pair training (not supported yet). Generation
with the LoRA (dev, or Turbo given no source image) carries the full identity.

## Krea 2: training on Raw, using the LoRA on Turbo

RulzTrainer trains LoRAs for [krea/Krea-2-Raw](https://huggingface.co/krea/Krea-2-Raw), Krea's 12B single-stream DiT
(28 blocks with grouped-query attention, a Qwen3-VL-4B text encoder whose twelve tapped layers are fused per token, and
the 16-channel Qwen-Image VAE). Pick **Krea-2-Raw** as the model, or any diffusers folder / repo id whose transformer
is a `Krea2Transformer2DModel` (detected from the model). The repo is gated: accept the Krea 2 Community License on
Hugging Face and `hf auth login`. The download skips the repo's 26 GB single-file copy and sample images (about 35 GB).

- **Train on Raw, use on Turbo.** Krea publishes Raw as the checkpoint to train on and uses LoRAs trained on it with
  [Krea-2-Turbo](https://huggingface.co/krea/Krea-2-Turbo), the 8-step distill (its own
  [Krea-2 LoRAs](https://huggingface.co/collections/krea/krea-2-loras) work that way). Set `sample_model_path:
  krea/Krea-2-Turbo` with `sample_steps: 8` and `sample_cfg: 1` to preview (and probe) the LoRA as it will be used;
  Turbo is gated separately, so accept its license too. Sampling Raw needs 28-52 steps with guidance.
- **Guidance.** `sample_cfg` uses the same convention for every model (1 = off). Krea's `guidance_scale` g is
  `sample_cfg` g + 1 here: Krea's 4.5 for Raw (its pipeline's default, 28 steps) is 5.5.
- **Memory.** The transformer is 25 GB in bf16, 13.5 GB in int8, about 8 GB in nvfp4. `transformer_quant: auto`
  counts the memory the model will live in, all GPUs of a model split together: bf16 from 40 GB, int8 (W8A8) from
  22 GB, else nvfp4. One 16 GB card therefore trains in nvfp4 (int8 plus the VAE already fill 14.2 GB, with no room
  left for training activations at 1 MP); two 16 GB cards with `multi_gpu: model_split` train in int8, the model
  loaded on the CPU and its blocks placed directly on both cards. On an RTX 50 card nvfp4 runs on the FP4 tensor
  cores (see [nvfp4 compute](#nvfp4-compute-native-fp4-on-blackwell-nvfp4_matmul)): 9.2 s per step at 1 MP on an
  RTX 5060 Ti, peaking at 14 GB. Split in int8 over an RTX 5060 Ti + RTX 4060 Ti: 10.0 s per step at 1 MP, peaking
  at 11.5 / 10.5 GB, with int8's smaller error (weights only, under 1%).
- **Single files.** `model_source: files` loads ComfyUI's Krea 2 files (diffusion model, Qwen3-VL-4B text encoder,
  Qwen-Image VAE), including finetunes and the gated Turbo, without the Hugging Face repo; see
  [Model sources](#model-sources-and-quantization).
- **Text features** follow `Krea2Pipeline`: the prompt in Krea's system/user template, twelve Qwen3-VL hidden layers
  per token. The pipeline pads every prompt to 512 tokens; padding is never attended to and text tokens have no
  position in the DiT, so the cache stores only the real tokens (a few MB per caption) with the same result.
- **Latents** come from the Qwen-Image VAE (16 channels, 8x), normalized with its per-channel mean/std and packed 2x2
  into 64 channels, so they are cached under their own file names (`_qwenvae`); region maps are shared with the
  other models (16-pixel cells either way).
- **LoRA targets**: the usual groups map onto `attn.to_q/k/v`, `attn.to_out.0`, `ff.gate/up`, `ff.down`; Krea 2
  adds `attn_gate` (the sigmoid output gate of every attention), `text_fusion` and `embed`. Krea's own LoRAs adapt all
  of them (rank 32). **By default Krea 2 trains the usual groups plus `attn_gate` and `text_fusion`**
  (`lora_targets: []` means the family's default; a config that lists its targets keeps them). The text-fusion stack
  (2 layerwise + 2 refiner blocks mixing the 12 text-encoder layers) is under-trained by the attention + feed-forward
  recipe: in [Fizgig](https://github.com/shootthesound/Fizgig)'s tests, scaling a finished LoRA's text-fusion blocks
  x3 lifted likeness 2-7 points on every LoRA but an overtrained one. At rank 16 this adds 8.9M LoRA weights (48.2M ->
  57.1M). The text-fusion blocks are checkpointed like the main ones, and since their gradient flows back through
  every block, `lora_blocks` no longer skips the backward of the earlier blocks while they are trained. ComfyUI loads
  these keys (`transformer.text_fusion.*` and `attn.to_gate` map to its `txtfusion.*` / `attn.gate`).
- **Export** is the diffusers / PEFT format with the same module names as Krea's LoRAs
  (`transformer.transformer_blocks.<n>.attn.to_q.lora_A.weight`), with the adapter config (alpha) in the metadata.
  Loaders that ignore the metadata assume alpha = rank, so keep alpha equal to rank (the default) for those.
- **Timesteps**: `timestep_sampling: reference` is a logit-normal shifted like the sampler (Krea has not published its
  training distribution); the default `logit_normal` works as for the other models.

```python
from diffusers import Krea2Pipeline  # needs peft for load_lora_weights
pipe = Krea2Pipeline.from_pretrained("krea/Krea-2-Turbo", torch_dtype=torch.bfloat16).to("cuda")
pipe.load_lora_weights("output/<job>/<job>.safetensors")
image = pipe("<trigger>, ...", num_inference_steps=8, guidance_scale=0.0).images[0]
```

Verified against diffusers: `tests/test_krea2.py` checks the port against `Krea2Transformer2DModel` (tiny random
weights; also the compact text layout, the latent packing, the schedule against `FlowMatchEulerDiscreteScheduler`, and
an exported LoRA merged into diffusers' weights). With the real Raw weights in bf16 the port matches diffusers' model to
bf16 rounding (cosine 0.99999 over a 512x512 prediction), and the compact prompt features are as close to the
pipeline's (cosine 0.99989) as the pipeline is to itself at another padding length.

## Qwen-Image 2.1

RulzTrainer trains text-to-image LoRAs for [Qwen/Qwen-Image-2.1](https://huggingface.co/Qwen/Qwen-Image-2.1), Qwen's
7B single-stream DiT (32 blocks, one modulation shared by every block, a Qwen3-VL-8B text encoder read at its last
layer before the final norm, and its own RGBA VAE: 16x, 64 channels). Pick **Qwen-Image-2.1** as the model. Edit LoRAs
(before / after pairs with reference images) are not supported yet.

- **Files.** By default the model comes as Comfy-Org's single files
  ([Comfy-Org/Qwen-Image-2.1](https://huggingface.co/Comfy-Org/Qwen-Image-2.1), int8 ConvRot): the diffusion model
  (7.3 GB), the Qwen3-VL-8B text encoder (9.4 GB, used to cache the captions and then unloaded) and the VAE, so one
  16 GB card trains it. The diffusers repo (`model_source: hf`) works too; `transformer_quant: auto` makes a bf16
  model int8 from 11 GB and keeps bf16 from 24 GB. A file name containing "turbo" marks Qwen-Image-2.1-Turbo.
- **Training adapter** (`training_adapter: auto`, the default). [Fizgig](https://github.com/shootthesound/Fizgig)
  measured that Qwen-Image 2.1 LoRAs trained alone collapse into texture or wobble part-way, with nothing in the loss
  to warn you; trained with a frozen *training adapter* (a small LoRA that is on for every training step, off for
  samples and probes, and never saved into your file) likeness reached 77 against 56 without. `auto` downloads
  Fizgig's ([ShootTheSound/Fizgig-Qwen-Image-2.1-Training-Adapter](https://huggingface.co/ShootTheSound/Fizgig-Qwen-Image-2.1-Training-Adapter),
  80 MB); `""` trains without one, a path uses another, `training_adapter_strength` scales it. The other families
  have none of their own, and a path works on them too. Not used by a full fine-tune.
- **Recipe** (Fizgig's, which the setup wizard suggests): 0.5 MP (it keeps more of the model's sharpness than 1 MP and
  is quicker), rank 8 at learning rate 2e-4 (rank 16 at half the rate for big or mixed datasets; 1.5e-4 for styles),
  EMA 0.98. Fizgig found identity sits in blocks 10-14 (`lora_blocks: 10-14` trains only those).
- **Buckets** are multiples of 32 px (the model takes its image tokens in 2x2 groups); latents are cached under their
  own names (`_qi21vae`), region maps are shared with the other models.
- **Attention** is block-causal as in the reference: caption tokens see only the caption before them, image tokens see
  everything, and the caption tokens are modulated at t = 0. RoPE positions are counted per sample, so a padded batch
  predicts exactly what each sample predicts alone.
- **Text features** follow `QwenImage21Pipeline`: the prompt in Qwen's system/user template, the last decoder layer's
  output before the final RMSNorm (hooked out, as the pipeline does on transformers 5), the system turn dropped. The
  int8 text encoder runs weight-only: W8A8 moved the features 5-8% against it at the same speed.
- **Samples** render with Comfy-Org's Turbo LoRA (8 steps, CFG 1, the Turbo checkpoint's own sigmas). Without it they
  follow the pipeline: the dynamic exponential shift (0.5 at 256 tokens to 0.9 at 8192) with the terminal stretch to
  0.02, 25 steps (ComfyUI's template) at CFG 1: Qwen-Image 2.1 is meant to be sampled without guidance. Sample sizes
  are rounded down to multiples of 32. Previews decode in 512 px VAE tiles: this VAE normalizes per pixel, so they give
  the full decode to 0.3 of an 8-bit level (PSNR 54 dB) at +1.7 GB instead of +6.6 GB at 1 MP.
- **LoRA targets**: the usual groups map onto `attn.to_q/k/v`, `attn.to_out.0`, `img_mlp.gate_layer/proj`
  (ffn_up_gate) and `img_mlp.out` (ffn_down); `text_in` adds the text projection and `embed` the image / timestep
  projections, the shared modulation and the final layer.
- **Export** is the diffusers / PEFT format (`transformer.transformer_blocks.<n>.attn.to_q.lora_A.weight`). ComfyUI's
  LoRA loader maps these keys, `img_mlp.gate_layer` / `img_mlp.proj` onto the two halves of its fused `gate_up`;
  keep alpha equal to rank (the default) for loaders that ignore the metadata.
- **Measured** on an RTX 5060 Ti (16 GB), int8, 0.5 MP, rank 8, identity preset: about 2.5 s per step with
  torch.compile, 10 of 32 blocks keeping their activations at a 13.1 GB peak.
- **Licence**: the Qwen Research License, non-commercial use unless Qwen grants more; a shared LoRA or fine-tune must
  say "Built with Qwen" or "Improved using Qwen".

```python
from diffusers import QwenImage21Pipeline  # needs peft for load_lora_weights
pipe = QwenImage21Pipeline.from_pretrained("Qwen/Qwen-Image-2.1-Turbo", torch_dtype=torch.bfloat16).to("cuda")
pipe.load_lora_weights("output/<job>/<job>.safetensors")
image = pipe("<trigger>, ...").images[0]
```

Verified against diffusers: `tests/test_qwen21.py` checks the port against `QwenImage21Transformer2DModel` (tiny random
weights, a padded batch against each sample alone), ComfyUI's fused single-file layout, the schedules against
`FlowMatchEulerDiscreteScheduler`, an exported LoRA merged into diffusers' weights, the Turbo LoRA's fused feed-forward
split onto both halves, and the training adapter (on while training, off while sampling, not saved).

## Using the LoRA in ComfyUI

Recent ComfyUI builds support MOSS-Image natively (`comfy/ldm/moss_image`, CLIP type `moss_image`), so no
plugin is required. Copy the trained `.safetensors` into `ComfyUI/models/loras/` and add a standard
**Load LoRA** node after *Load Diffusion Model*. Both the unfused checkpoints and ComfyUI's repacked
fused-qkv files work: ComfyUI maps the `proj.q/k/v` LoRA keys onto slices of the fused weight.

The file uses the generic ComfyUI key format (the same one ComfyUI's own MOSS-Image LoRAs use):

```
diffusion_model.layers.<n>.proj.<q|k|v|o|ffn_up_gate|ffn_down>.lora_down.weight
diffusion_model.layers.<n>.proj.<...>.lora_up.weight
diffusion_model.layers.<n>.proj.<...>.alpha
```

These names are identical in the diffusers checkpoint and the ComfyUI plugin model, so no key conversion is needed.

### Scoring likeness in ComfyUI

`comfyui/rulztrainer_likeness` is a ComfyUI node pack that scores any images against a person's references with the
trainer's own scoring code and defaults: face, body and overall on the normalized scale, averaged over one image, a
batch or folders of images, with every image's own scores and an optional .xlsx or CSV export. References can be
shared or separate for face and body, as in the trainer. An optional set of base-model images sets the 0 of the scale
like the trainer's step-0 probe, and a trainer job folder can supply that run's exact scale. Link the folder into
ComfyUI's `custom_nodes` and load `workflows/RulzTrainer Likeness Scoring.json`; its
[README](comfyui/rulztrainer_likeness/README.md) covers installation and every input.

## VRAM guidance (16 GB cards)

| Setting | Recommendation |
|---|---|
| Gradient checkpointing | on |
| Batch size | 1 (use gradient accumulation instead) |
| Optimizer | `adamw8bit`, or `automagic3` (self-tuning lr, ported from ai-toolkit) |
| Resolution | 1024 |
| Text encoder device | `auto` (spreads over all GPUs + CPU during caching only) |
| Max caption tokens | 512 to 1024 unless you train on very long prompts |

The DiT in bf16 is about 10 GB. The text encoder is about 18 GB in bf16, so it is loaded only for caching
with `device_map="auto"`; pick `8bit` / `4bit` quantization if you want it on a single GPU.

### Selective checkpointing (`checkpoint_keep_blocks`)

Gradient checkpointing stores only each block's input and runs the block's forward a second time during the
backward. Blocks that fit in spare GPU memory can keep their activations instead and skip that recompute: the
results are bit-identical, only speed and memory change. `checkpoint_keep_blocks: -1` (the default) measures one
forward and backward of the largest training bucket before the first step, with one and with three blocks kept, and
keeps as many of the last trained blocks as the free memory allows, minus a margin (1.5 GB or 8% of the card, 1.5 GB
more with the identity loss). Each GPU of a model split chooses for its own blocks. `0` checkpoints every block (the
previous behavior), `N` keeps N blocks. If a step still runs out of memory the kept blocks are halved and the step
reruns; if the allocator has to retry, the earliest kept block on that GPU goes.

Measured, 20 steps at 1 MP, seconds per step without and with auto:

| run | every block checkpointed | auto | change |
|---|---|---|---|
| MOSS-Image int8, one 16 GB card | 6.70 | 6.43 (6 of 42 blocks kept) | -4% |
| Krea 2 int8, split over two 16 GB cards | 9.87 | 9.68-9.88 (7 of 28) | 0-2%, within noise |
| Krea 2 nvfp4, one 16 GB card | 8.31 | 8.37 (1 of 28) | none |

The gain is small on 16 GB cards: there is little spare memory, and with int8/nvfp4 the forward a kept block skips
is cheap next to its bf16 backward. Cards with more free memory keep more blocks; that case is not measured here.

**Memory for samples and probes after training steps.** The activations a step frees stay reserved by PyTorch's
allocator, and with its default segments they are cut into hundreds of pieces by small tensors that live on, so a
large sample or probe could run out of memory after the first steps although the same image rendered before
training (measured, Krea 2 on Windows: 3.9 GB cached in 784 segments, the largest free piece 0.19 GB). On Linux the
trainer runs the allocator with `expandable_segments:True` (unless `PYTORCH_ALLOC_CONF` / `PYTORCH_CUDA_ALLOC_CONF`
is set), where freed memory stays one piece; before every sample and probe round it drops the finished step's
gradients and returns the cached memory. Measured on Linux: the probe after two training steps starts at 8.40 GB
allocated / 8.52 GB reserved, the one before training at 8.28 / 8.29 GB. The Windows CUDA driver has no expandable
segments; there only the cleanup applies.

### Block swap (`block_swap`)

For a card the model does not fit, after musubi-tuner's block swap. `block_swap: N` keeps the frozen weights of the
first N DiT blocks in system RAM (pinned) and streams each block's weights to the GPU just before it runs: the int8 or
nvfp4 data, scales, biases and norms. The next block is copied in on a side CUDA stream while the current one computes.
Frozen weights never change, so nothing is copied back. The LoRA adapters, the rest of the model and the activations
stay on the GPU.

- **Why it needs gradient checkpointing.** In training it relies on checkpointing: a checkpointed block keeps no
  weights for its backward, its recomputation streams them in again, and they go once that block's backward is done.
  The swapped blocks are therefore never among the blocks that keep their activations.
- **Two blocks at most.** At most two swapped blocks' weights are on the GPU at a time. The CPU waits rather than
  queue copies far ahead of the GPU, and a recomputation that stops early still releases its block.
- **Measured.** Krea 2 int8 on the 16 GB RTX 4060 Ti, 0.25 MP, 14 of 28 blocks swapped (3.4 GB in pinned RAM), every
  block checkpointed:

  | | Model on the GPU | Training-step peak | Step time |
  |---|---|---|---|
  | No swap | 9.17 GB | 10.81 GB | 2.2-2.8 s |
  | `block_swap: 14` | 5.74 GB | 7.66 GB | 2.6 s |

  The loss matches the unswapped run. The transfers mostly hide behind the blocks' compute on this PCIe link; a
  slower link or a smaller per-block compute shows more of them.
- **Limits.** Single GPU only (`model_split` spreads blocks over GPUs instead). Not with data parallel or a fine-tune.
  torch.compile is turned off. A sampling model swapped in and out keeps the split. The *Block swap* field is on the
  Train tab's memory section.

## torch.compile (`compile_model`)

`compile_model: auto` (the default) compiles the DiT blocks for runs with at least 100 steps to go, on a GPU with
triton installed; `true` always compiles, `false` never. Inductor fuses the elementwise work between the matrix
multiplies (norms, RoPE, modulation, gates, residuals) into fewer kernels. Measured on the RTX 5060 Ti at 1 MP, 20
steps, every block checkpointed:

| model | eager | compiled | warm-up (first steps) |
|---|---|---|---|
| MOSS-Image, int8_convrot | 6.61 s/step | **4.90 s/step (-26%)** | 25 s |
| Krea 2, nvfp4 (native FP4) | 8.77 s/step | **7.74 s/step (-12%)** | 36 s |
| Krea 2, nvfp4, cuDNN attention, 15-image set, 45 steps (median of steps 21-45) | 7.33 s/step | **6.09-6.22 s/step (-15 to -17%)** | 36 s (160 s with a cold inductor cache) |

Dynamo's settings are thread-local in torch 2.12+, so the raised recompile limits are applied in every thread that
runs a block (autograd's recompute thread, the data-parallel workers). Measured, that changed nothing here: the
per-block graphs use dynamic shapes, and the compile stats show 0 recompiles over 45 steps and 15 caption lengths, so
no limit was ever reached (`RULZ_COMPILE_THREAD_LIMITS=0` runs with torch's own limits in the other threads, for
measuring).

In data parallel over the 5060 Ti and a 4060 Ti (2 images per step) the gain was larger for MOSS-Image: 7.30 ->
4.79 s/step (-34%), and 12.74 -> 11.81 s/step (-7%) for Krea 2 (see [Using two GPUs](#using-two-gpus)). Losses match
to bf16 noise. The quantized Linears' triton kernels stay opaque to dynamo (a graph break at each),
which still leaves 19-29 graphs per model; the limits on recompiles are raised so no block silently drops back to
eager, and dynamo's warnings go to `train.log`. The log reports frames, graphs and graph breaks after 5 steps and at
the end. With `auto`, a block that fails to compile runs uncompiled instead of stopping the run. A profile of the
compiled steps puts attention at 42-50% and the matrix multiplies at 31-38% of the GPU time; the fused kernels are
about 10%, so removing the graph breaks (custom ops for the quantized Linears) could gain a few percent at most and
is not done. `RULZ_PROFILE_STEP=N` writes such a profile of step N (`profile_step_N.txt` in the job folder), and
`scripts/bench.py` runs the short timing runs used here (it waits for idle GPUs).

## Layout

```
rulztrainer/
  config.py        TrainConfig dataclass + YAML
  dataset.py       scanning, bucketing, latent/text caching, batching
  lora.py          LoRA injection, ComfyUI-format export/import, sampling-only extra LoRAs
  quant.py         int8 / int8_convrot / nvfp4 weight quantization matching comfy_kitchen, QuantLinear
  sampler.py       flow-matching Euler sampler (reference schedule)
  trainer.py       training loop, saving, sampling, status.json for the UI
  checkpointing.py selective gradient checkpointing (which blocks keep their activations)
  ema.py           weight average (EMA) of the LoRA for saves, samples and probes
  compiling.py     torch.compile settings, the compile_model auto rule, compile stats
  profiling.py     one-step GPU profile by kernel group (RULZ_PROFILE_STEP)
  race.py          configuration race: settings, round schedule, swapping configurations' LoRAs
  train.py         CLI entry point
  moss/model.py    MOSS-Image DiT (vendored, Apache-2.0, OpenMOSS)
  moss/loading.py  download + load (HF layout or single files), quant dispatch, encode helpers
  moss/single_file.py  ComfyUI single-file loaders (dequant, unfuse qkv, text-only Qwen3.5)
  archs.py         the model families (MOSS-Image, FIBO 2.0, Krea 2) behind one interface
  fibo/            Bria FIBO 2.0: DiT port, loading, VGL prompts, sampler
  krea2/           Krea 2: DiT port, loading (Qwen-Image VAE, Qwen3-VL layer stack), sampler
webui.py           NiceGUI web UI (runs training as a subprocess)
configs/           example configs
tests/             smoke tests (no weights needed): python -m pytest tests
install.bat / run_webui.bat   Windows install and start;  install.sh / run_webui.sh   the same for Linux
scripts/vast/      vast.ai: provision.sh (PROVISIONING_SCRIPT), supervisor wrapper + conf, create_template.sh
scripts/bench.py   short timing runs that wait for idle GPUs (s/step, warm-up, peak memory)
Dockerfile         optional vast.ai image with the install baked in
```

## Adding another model

Model families are `Arch` classes in `rulztrainer/archs.py` (MOSS-Image, FIBO 2.0 and Krea 2 today). An Arch provides
the loading functions (transformer, VAE, text encoder), `encode_prompt` / `pad_embeds`, `encode_images` /
`decode_latents` for its VAE, a sampler, the LoRA module names for the target groups, the export format, and its
reference timestep recipe. A family with a VAE of its own sets `latent_tag`, so its latents are cached under their own
file names. The transformer must take the trainer's convention, `model(latents, sigma * 1000, embeds, mask)` with the
normalized 2x2-patchified latents [B, C, H/16, W/16], and expose `.layers` (the blocks) for block selection, compile
and the model split, plus `split_tail_modules()` (what runs after the last block); `rulztrainer/krea2/` is the most
recent template, and `tests/test_krea2.py` checks a port against the diffusers model with tiny random weights.

## License

Open source under the [MIT License](LICENSE). Base models you download (MOSS-Image, FIBO, Krea, text encoders) keep their own licenses.

The RulZ name and logo are excluded from the MIT License; all rights reserved. If you fork or redistribute this
project, remove or replace the RulZ branding. See [LOGO-NOTICE.md](LOGO-NOTICE.md).
