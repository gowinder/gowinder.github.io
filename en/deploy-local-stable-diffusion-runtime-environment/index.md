# Deploy a Local Stable Diffusion Runtime Environment


# Deploy a Local Stable Diffusion Runtime Environment

`midjourney` has been quite popular lately, but it has started charging—10 dollars a month, which is more than I can afford—so I switched to the open-source `Stable Diffusion`. You can deploy it on `Google Colab` with one click:
`https://github.com/camenduru/stable-diffusion-webui-colab`. However, the free `gpu` on `Google Colab` runs out quickly, and although the paid one is cheap, you still have to use it sparingly. If you have a local `gpu` with more than `8G` of VRAM, you can run it on your own computer to save costs; `6G` of VRAM should also work—try lowering the resolution and keywords.

This document is based on `https://github.com/camenduru/stable-diffusion-webui-colab`, and I have added some commonly used models myself.
One difference between `Stable Diffusion` and `midjourney` is that there are a great many models; choosing a model in the style you want will better generate the effect you want, and it is also more fun to play with.
You can pick models on a few commonly used websites:

- `https://huggingface.co/` has many open-source models; just search for `stable diffusion`. Users there share repositories similar to `github` repos, and large files can be downloaded directly or via `git lfs`
- `https://civitai.com/` also has models shared by users, and it is quite intuitive, with sample images, much like `instagram`—very convenient. Likewise, just search for `stable diffusion`

Some commonly used `stable diffusion` websites:

- `https://www.reddit.com/r/StableDiffusion/` — the `stable diffusion` section on `reddit`; its [tutorials](https://www.reddit.com/r/stablediffusion/wiki/tutorials/) are excellent
- `https://stable-diffusion-art.com` has many basic tutorials

Below I will record the deployment process.

## Hardware Environment

I have successfully deployed it on `proxvox ev 7.3` and `ubuntu 20.04`, using `telsa P4 8G` and `3070 8G` graphics cards.

## Set the Root Folder

`google colab` uses `/content/stable-diffusion-webui`, but I put it on `zfs` locally because the models are quite large and I don't want to keep downloading them over and over.

```shell
export PROJ_DIR=/tank/download/ai-relate/stable-diffusion-webui
```

After setting the `$PROJ_DIR` environment variable, all downloads will be based on this environment variable.

## Install Libraries

This step doesn't seem to be mandatory.

```shell
sudo apt -y update -qq
wget http://launchpadlibrarian.net/367274644/libgoogle-perftools-dev_2.5-2.2ubuntu3_amd64.deb
wget https://launchpad.net/ubuntu/+source/google-perftools/2.5-2.2ubuntu3/+build/14795286/+files/google-perftools_2.5-2.2ubuntu3_all.deb
wget https://launchpad.net/ubuntu/+source/google-perftools/2.5-2.2ubuntu3/+build/14795286/+files/libtcmalloc-minimal4_2.5-2.2ubuntu3_amd64.deb
wget https://launchpad.net/ubuntu/+source/google-perftools/2.5-2.2ubuntu3/+build/14795286/+files/libgoogle-perftools4_2.5-2.2ubuntu3_amd64.deb
sudo apt install -qq -y libunwind8-dev
sudo dpkg -i *.deb
env LD_PRELOAD=libtcmalloc.so
rm *.deb

sudo apt install libnvinfer8 python3-libnvinfer
```

## Install `conda`

For ordinary `python` projects I like to use `poetry` to manage packages, but for machine learning projects I generally use `miniconda`.

```shell
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh
# agreements 。。。
```

Create a `conda` environment: `conda create -n py310sd python=3.10`

## Basic Installation

Install `aria2`; it supports multi-threaded downloads and is faster than `wget`: `sudo apt -y install aria2`.
Then install the `python` libraries.

```shell

pip install torch==1.13.1+cu116 torchvision==0.14.1+cu116 torchaudio==0.13.1 torchtext==0.14.1 torchdata==0.5.1 --extra-index-url https://download.pytorch.org/whl/cu116 -U
pip install xformers==0.0.16 triton==2.0.0 -U
pip install tensorrt
```

Clone `stable-diffusion-webui` and common extensions.

```shell
#ln -s /opt/conda/lib/python3.10/site-packages/libnvinfer.so.8 /opt/conda/lib/python3.10/site-packages/libnvinfer.so.7
#ln -s /opt/conda/lib/python3.10/site-packages/libnvinfer_plugin.so.8 /opt/conda/lib/python3.10/site-packages/libnvinfer_plugin.so.7
#set -x LD_LIBRARY_PAH $LD_LIBRARY_PATH:/opt/conda/lib:/opt/conda/lib/python3.10/site-packages

git clone -b v2.1 https://github.com/camenduru/stable-diffusion-webui
git clone https://huggingface.co/embed/negative $PROJ_DIR/embeddings/negative
git clone https://huggingface.co/embed/lora $PROJ_DIR/models/Lora/positive
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/embed/upscale/resolve/main/4x-UltraSharp.pth -d $PROJ_DIR/models/ESRGAN -o 4x-UltraSharp.pth

wget https://raw.githubusercontent.com/camenduru/stable-diffusion-webui-scripts/main/run_n_times.py -O $PROJ_DIR/scripts/run_n_times.py
git clone https://github.com/deforum-art/deforum-for-automatic1111-webui $PROJ_DIR/extensions/deforum-for-automatic1111-webui
git clone https://github.com/camenduru/stable-diffusion-webui-images-browser $PROJ_DIR/extensions/stable-diffusion-webui-images-browser
git clone https://github.com/camenduru/stable-diffusion-webui-huggingface $PROJ_DIR/extensions/stable-diffusion-webui-huggingface
git clone https://github.com/camenduru/sd-civitai-browser $PROJ_DIR/extensions/sd-civitai-browser
git clone https://github.com/kohya-ss/sd-webui-additional-networks $PROJ_DIR/extensions/sd-webui-additional-networks
git clone https://github.com/Mikubill/sd-webui-controlnet $PROJ_DIR/extensions/sd-webui-controlnet
git clone https://github.com/camenduru/openpose-editor $PROJ_DIR/extensions/openpose-editor
git clone https://github.com/jexom/sd-webui-depth-lib $PROJ_DIR/extensions/sd-webui-depth-lib
git clone https://github.com/hnmr293/posex $PROJ_DIR/extensions/posex
git clone https://github.com/camenduru/sd-webui-tunnels $PROJ_DIR/extensions/sd-webui-tunnels
git clone https://github.com/etherealxx/batchlinks-webui $PROJ_DIR/extensions/batchlinks-webui
git clone https://github.com/camenduru/stable-diffusion-webui-catppuccin $PROJ_DIR/extensions/stable-diffusion-webui-catppuccin
git clone https://github.com/KohakuBlueleaf/a1111-sd-webui-locon $PROJ_DIR/extensions/a1111-sd-webui-locon
git clone https://github.com/AUTOMATIC1111/stable-diffusion-webui-rembg $PROJ_DIR/extensions/stable-diffusion-webui-rembg
git clone https://github.com/ashen-sensored/stable-diffusion-webui-two-shot $PROJ_DIR/extensions/stable-diffusion-webui-two-shot
git clone https://github.com/camenduru/sd_webui_stealth_pnginfo $PROJ_DIR/extensions/sd_webui_stealth_pnginfo

cd $PROJ_DIR
git reset --hard

aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_canny-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_canny-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_depth-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_depth-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_hed-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_hed-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_mlsd-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_mlsd-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_normal-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_normal-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_openpose-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_openpose-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_scribble-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_scribble-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/control_seg-fp16.safetensors -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o control_seg-fp16.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/hand_pose_model.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/annotator/openpose -o hand_pose_model.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/body_pose_model.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/annotator/openpose -o body_pose_model.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/dpt_hybrid-midas-501f0c75.pt -d $PROJ_DIR/extensions/sd-webui-controlnet/annotator/midas -o dpt_hybrid-midas-501f0c75.pt
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/mlsd_large_512_fp32.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/annotator/mlsd -o mlsd_large_512_fp32.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/mlsd_tiny_512_fp32.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/annotator/mlsd -o mlsd_tiny_512_fp32.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/network-bsds500.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/annotator/hed -o network-bsds500.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/upernet_global_small.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/annotator/uniformer -o upernet_global_small.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_style_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_style_sd14v1.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_sketch_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_sketch_sd14v1.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_seg_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_seg_sd14v1.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_openpose_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_openpose_sd14v1.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_keypose_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_keypose_sd14v1.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_depth_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_depth_sd14v1.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_color_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_color_sd14v1.pth
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/ControlNet/resolve/main/t2iadapter_canny_sd14v1.pth -d $PROJ_DIR/extensions/sd-webui-controlnet/models -o t2iadapter_canny_sd14v1.pth

sed -i -e '''/    prepare_environment()/a\    os.system\(f\"""sed -i -e ''\"s/dict()))/dict())).cuda()/g\"'' $PROJ_DIR/repositories/stable-diffusion-stability-ai/ldm/util.py""")''' $PROJ_DIR/launch.py
sed -i -e 's/fastapi==0.90.1/fastapi==0.89.1/g' $PROJ_DIR/requirements_versions.txt

mkdir $PROJ_DIR/extensions/deforum-for-automatic1111-webui/models

sed -i -e 's/\"sd_model_checkpoint\"\,/\"sd_model_checkpoint\,sd_vae\,CLIP_stop_at_last_layers\"\,/g' $PROJ_DIR/modules/shared.py
```

The basics are done; below are some popular `models`.

## stable-diffusion 1.5

```shell
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/runwayml/stable-diffusion-v1-5/resolve/main/v1-5-pruned.ckpt -d $PROJ_DIR/models/Stable-diffusion -o v1-5-pruned.ckpt
```

## download AbyssOrangeMix2 NSFW

```shell
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/OrangeMixs/resolve/main/AbyssOrangeMix2_nsfw.safetensors -d $PROJ_DIR/models/Stable-diffusion -o AbyssOrangeMix2_nsfw.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/OrangeMixs/resolve/main/AbyssOrangeMix2_hard.safetensors -d $PROJ_DIR/models/Stable-diffusion -o AbyssOrangeMix2_hard.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/OrangeCocoaMix/resolve/main/OrangeCocoaMix2_hard.safetensors -d $PROJ_DIR/models/Stable-diffusion -o OrangeCocoaMix2_hard.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/ckpt/sd-vae-ft-mse-original/resolve/main/vae-ft-mse-840000-ema-pruned.ckpt -d $PROJ_DIR/models/Stable-diffusion -o AbyssOrangeMix2_nsfw.vae.pt
```

## download inkpunk

`https://huggingface.co/Envvi/Inkpunk-Diffusion`

![Inkpunk](https://huggingface.co/Envvi/Inkpunk-Diffusion/resolve/main/inkpunk-v2-samples-2.png)

```shell
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/Envvi/Inkpunk-Diffusion/resolve/main/Inkpunk-Diffusion-v2.ckpt -d $PROJ_DIR/models/Stable-diffusion -o Inkpunk-Diffusion-v2.ckpt
```

## download Mo Di diffusion

`https://huggingface.co/nitrosocke/mo-di-diffusion`

![MoDi](https://huggingface.co/nitrosocke/mo-di-diffusion/resolve/main/modi-samples-03s.jpg)

```shell
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/nitrosocke/mo-di-diffusion/blob/main/moDi-v1-pruned.ckpt -d $PROJ_DIR/models/Stable-diffusion -o moDi-v1-pruned.ckpt
```

## AbyssOrangeMix3

`https://huggingface.co/WarriorMama777/OrangeMixs`

![AOM3A1B](https://github.com/WarriorMama777/imgup/raw/3e060893c0fb2c80c6f3aedf63bf8d576c9a37fc/img/AOM3/img_samples_AOM3A1B_01_comp001.webp)

```shell
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/WarriorMama777/OrangeMixs/resolve/main/Models/AbyssOrangeMix3/AOM3A1B_orangemixs.safetensors -d $PROJ_DIR/models/Stable-diffusion -o AOM3A1B_orangemixs.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/WarriorMama777/OrangeMixs/resolve/main/Models/AbyssOrangeMix3/AOM3A1_orangemixs.safetensors -d $PROJ_DIR/models/Stable-diffusion -o AOM3A1_orangemixs.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/WarriorMama777/OrangeMixs/resolve/main/Models/AbyssOrangeMix3/AOM3A3_orangemixs.safetensors -d $PROJ_DIR/models/Stable-diffusion -o AOM3A3_orangemixs.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/WarriorMama777/OrangeMixs/resolve/main/Models/AbyssOrangeMix3/AOM3A2_orangemixs.safetensors -d $PROJ_DIR/models/Stable-diffusion -o AOM3A2_orangemixs.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/WarriorMama777/OrangeMixs/resolve/main/Models/AbyssOrangeMix3/AOM3_orangemixs.safetensors -d $PROJ_DIR/models/Stable-diffusion -o AOM3_orangemixs.safetensors
```

## SCIFI

`https://huggingface.co/Corruptlake/Sci-Fi-Diffusion/tree/main`

![SCIFI](https://i.ibb.co/vw44xpj/sdvssf.png)

```shell
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/Corruptlake/Sci-Fi-Diffusion/resolve/main/Sci-Fi_Diffusion_1.0.safetensors -d $PROJ_DIR/models/Stable-diffusion -o Sci-Fi_Diffusion_1.0.safetensors
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/Corruptlake/Sci-Fi-Diffusion/resolve/main/Sci-Fi_Diffusion_1.0.ckpt -d $PROJ_DIR/models/Stable-diffusion -o Sci-Fi_Diffusion_1.0.ckpt
```

# hentai model

I won't include an image; the address is at

```shell
aria2c --console-log-level=error -c -x 16 -s 16 -k 1M https://huggingface.co/Deltaadams/HD-22/resolve/main/HD22%20S.zip -d $PROJ_DIR/models/Stable-diffusion -o hd22.zip
cd $PROJ_DIR/models/Stable-diffusion
unzip hd22.zip
```

## complexla style

```shell
aria2c -x 15 -s 10 -c -k 1M https://huggingface.co/Conflictx/Complex-Lineart/resolve/main/ComplexLA%20Style.ckpt -d $PROJ_DIR/models/Stable-diffusion -o ComplexLA-Style.ckpt
```

## f222

`https://huggingface.co/acheong08/f222`

![f222](https://i0.wp.com/stable-diffusion-art.com/wp-content/uploads/2023/01/image-47.png?w=786&ssl=1)

```shell
aria2c -x 15 -s 10 -c -k 1M https://huggingface.co/acheong08/f222/resolve/main/f222.ckpt -d $PROJ_DIR/models/Stable-diffusion -o f222.ckpt
```

## Chillout Mix

`https://civitai.com/models/6424/chilloutmix`

![ChilloutMix](https://i0.wp.com/stable-diffusion-art.com/wp-content/uploads/2023/03/Untitled.png?w=832&ssl=1)

```shell
aria2c -x 15 -s 10 -c -k 1M https://civitai.com/api/download/models/11745 -d $PROJ_DIR/models/Stable-diffusion -o ChilloutMix.safetensors
```

## URPM

```shell
aria2c -x 15 -s 10 -c -k 1M https://civitai.com/api/download/models/15640 -d $PROJ_DIR/models/Stable-diffusion -o URPM.safetensors
```

## OpenJourney

`https://huggingface.co/prompthero/openjourney`

![Open Journey](https://s3.amazonaws.com/moonup/production/uploads/1667904587642-63265d019f9d19bfd4f45031.png)

```shell
aria2c -x 15 -s 10 -c -k 1M https://huggingface.co/prompthero/openjourney/resolve/main/mdjrny-v4.ckpt -d $PROJ_DIR/models/Stable-diffusion -o mdjrny-v4.ckpt
```

## Anythings V5

`https://civitai.com/models/9409/or-anything-v5`

![Anythings V5](https://imagecache.civitai.com/xG1nkqKTMzGDvpLrqFT7WA/aeda34a6-4db8-4f7e-23fd-c7bac4bcae00/width=450/342218.jpeg)

```shell
aria2c -x 15 -s 10 -c -k 1M https://civitai.com/api/download/models/30163 -d $PROJ_DIR/models/Stable-diffusion -o AnyThingsV5.ckpt
```

## DreamShaper

`https://huggingface.co/Lykon/DreamShaper`

![DreamShaper](https://huggingface.co/Lykon/DreamShaper/resolve/main/1.png)

```shell
#  DreamShaper_5_beta2_BakedVae_fp16.ckpt
aria2c -x 15 -s 10 -c -k 1M https://huggingface.co/Lykon/DreamShaper/resolve/main/DreamShaper_5_beta2_BakedVae_fp16.ckpt -d $PROJ_DIR/models/Stable-diffusion -o  DreamShaper_5_beta2_BakedVae_fp16.ckpt
```

## Run `webui`

```shell
cd $PROJ_DIR

python launch.py --listen --xformers --enable-insecure-extension-access --theme dark --gradio-queue --multiple
```

If you need external network access, add the `--share` parameter.

