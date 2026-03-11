# 环境配置方案 — S3PO-GS

## 基本信息
- conda 环境名：`S3PO-GS`
- Python 版本：`3.11`
- CUDA 来源：系统 CUDA，路径 `/usr/local/cuda-11.8`
- CUDA 版本：`11.8.89`
- PyTorch 版本：`2.1.0`
- torchvision 版本：`0.16.0`
- torchaudio 版本：`2.1.0`

## 版本冲突分析
- `environment.yml` 把 conda、PyTorch CUDA 包和普通 pip 依赖混在一起，不适合拆分安装。
- 根目录当前没有 `requirements.txt`，执行前需要补一个根依赖文件。
- `environment.yml` 同时出现 `opencv=4.8.1` 和 `opencv`，属于重复定义；拆分后统一改为 pip 的 `opencv-python`。
- `environment.yml` 中的 `cuda-nvcc=11.8.89` 在本机已有系统 CUDA 11.8 时不必再装。
- `environment.yml` 中的 `matplotlib==3.5.3` 与 `Python 3.11` 存在兼容风险；执行时不沿用这个旧 pin，改用 Python 3.11 可用版本。
- 代码里还直接导入了 `rich`，但它不在当前 `environment.yml` 中；生成 `requirements.txt` 时需要补上。

## 安装步骤

### 1. 创建 conda 环境
```bash
source ~/.bashrc
conda create -n S3PO-GS python=3.11 pip -y
```

### 2. 进入环境并固定 CUDA 11.8
```bash
source ~/.bashrc
conda activate S3PO-GS
export CUDA_HOME=/usr/local/cuda-11.8
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:${LD_LIBRARY_PATH}
```

### 3. 安装少量运行时二进制依赖
说明：这一步只保留 `requirements.txt` 难以覆盖的运行时包，其余 Python 依赖全部走 pip。
```bash
source ~/.bashrc
conda activate S3PO-GS
conda install -c conda-forge ffmpeg glib tbb -y
```

### 4. 用 pip 安装 numpy 和 PyTorch 大包
```bash
source ~/.bashrc
conda activate S3PO-GS
pip install --upgrade pip
pip install numpy==1.24.4
pip install torch==2.1.0 torchvision==0.16.0 torchaudio==2.1.0 \
  --index-url https://download.pytorch.org/whl/cu118
```

### 5. 生成并安装根 requirements.txt
执行时会在仓库根目录新增 `requirements.txt`，内容来自：
- 根 `environment.yml` 里的 pip 依赖
- `dust3r/requirements.txt`
- `dust3r/requirements_optional.txt`
- 代码中实际导入但缺失的包

安装命令：
```bash
source ~/.bashrc
conda activate S3PO-GS
pip install -r requirements.txt
```

### 6. 编译/安装仓库内扩展
```bash
source ~/.bashrc
conda activate S3PO-GS
export CUDA_HOME=/usr/local/cuda-11.8
export PATH=$CUDA_HOME/bin:$PATH
export LD_LIBRARY_PATH=$CUDA_HOME/lib64:${LD_LIBRARY_PATH}

pip install submodules/simple-knn
pip install submodules/diff-gaussian-rasterization

cd croco/models/curope
python setup.py build_ext --inplace
cd ../..
```

### 7. 基本验证
```bash
source ~/.bashrc
conda activate S3PO-GS
python - <<'PY'
import torch
import cv2
import open3d
import gradio
import wandb
import pycolmap
import poselib
import rich
print("torch", torch.__version__)
print("cuda", torch.version.cuda)
print("cuda_available", torch.cuda.is_available())
print("cv2", cv2.__version__)
print("open3d", open3d.__version__)
PY
```

## 需要 sudo 的步骤
- 当前方案没有必须的 `sudo` 步骤。

## 数据与模型准备（供参考，不执行）
- 下载 `MASt3R_ViTLarge_BaseDecoder_512_catmlpdpt_metric.pth` 到 `checkpoints/`
- Waymo 处理后数据放到 `datasets/waymo`
- KITTI 处理后数据放到 `datasets/KITTI`
- DL3DV 处理后数据放到 `datasets/dl3dv`
- 如需复现实验，KITTI 全序列列表见仓库根目录 `KITTI_sequence_list.txt`

## 执行阶段的实际交付
- 新增根目录 `requirements.txt`
- 按上述顺序创建并安装 `S3PO-GS` 环境
- 编译两个高斯子模块和 RoPE CUDA 扩展
- 做一次最小导入验证
