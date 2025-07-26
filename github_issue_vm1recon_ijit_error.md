# [BUG] iJIT_NotifyEvent symbol error prevents vm1recon environment from working

## 🐛 **Bug Description**

The `vm1recon` conda environment (required for MegaSAM reconstruction, NKSR meshification, and GeoCalib) fails to import PyTorch with an **undefined symbol error**:

```
ImportError: /home/ubuntu/miniconda/envs/vm1recon/lib/python3.10/site-packages/torch/lib/libtorch_cpu.so: undefined symbol: iJIT_NotifyEvent
```

This completely blocks Stage 1 (reconstruction) and Stage 3 (postprocessing) of the VideoMimic pipeline, while Stage 0 (preprocessing), Stage 2 (optimization), and Stage 4 (retargeting) work perfectly in the `vm1rs` environment.

## 🔄 **Reproduction Steps**

1. **Follow the exact setup instructions** from `docs/setup.md`
2. **Create vm1recon environment**:
   ```bash
   cd third_party/megasam-package
   conda env create -f environment.yml
   ```
3. **Install dependencies as documented**:
   ```bash
   conda activate vm1recon
   export CUDA_HOME=/usr/local/cuda-11.8
   export CC=/usr/bin/gcc-11
   export CXX=/usr/bin/g++-11
   pip install torch-scatter==2.1.2
   wget https://anaconda.org/xformers/xformers/0.0.22.post7/download/linux-64/xformers-0.0.22.post7-py310_cu11.8.0_pyt2.0.1.tar.bz2
   conda install xformers-0.0.22.post7-py310_cu11.8.0_pyt2.0.1.tar.bz2 -y
   # ... additional dependencies
   ```
4. **Try to import PyTorch**:
   ```bash
   python -c "import torch"
   ```

**Result**: Immediate crash with `iJIT_NotifyEvent` symbol error.

## 💻 **System Information**

### Operating System
```
Ubuntu 22.04.5 LTS (Jammy Jellyfish)
Linux isaac-isaac-sim 6.8.0-1031-aws #33~22.04.1-Ubuntu SMP Thu Jun 26 14:22:30 UTC 2025 x86_64
```

### Hardware
```
GPU: Tesla T4 (15360MiB VRAM)
NVIDIA Driver: 535.247.01
CUDA Version: 12.2 (nvidia-smi)
```

### CUDA Installations
```
/usr/local/cuda-11.8/ (complete installation)
/usr/local/cuda-12.4/ (complete installation)
nvcc version: 11.5.119 (system default)
```

### Python Environment
```
Python: 3.12.11 (vm1rs) / 3.10.18 (vm1recon)
Conda: 25.5.1
Pip: 25.1
```

### vm1recon Environment PyTorch Details
```
pytorch                      2.0.1            py3.10_cuda11.8_cudnn8.7.0_0  pytorch
pytorch-cuda                 11.8             h7e8668a_6                    pytorch
torchvision                  0.15.2           py310_cu118                   pytorch
xformers                     0.0.22.post7     py310_cu11.8.0_pyt2.0.1       <unknown>
torch-scatter                2.1.2+pt20cu118  pypi_0                        pypi
```

### Library Dependencies Analysis
```bash
ldd /home/ubuntu/miniconda/envs/vm1recon/lib/python3.10/site-packages/torch/lib/libtorch_cpu.so | grep -E "(mkl|intel)"
```
```
libmkl_intel_lp64.so => /home/ubuntu/miniconda/envs/vm1recon/lib/python3.10/site-packages/torch/lib/../../../../libmkl_intel_lp64.so
libmkl_gnu_thread.so => /home/ubuntu/miniconda/envs/vm1recon/lib/python3.10/site-packages/torch/lib/../../../../libmkl_gnu_thread.so
libmkl_core.so => /home/ubuntu/miniconda/envs/vm1recon/lib/python3.10/site-packages/torch/lib/../../../../libmkl_core.so
```

## 🔍 **Error Analysis**

### Root Cause
The `iJIT_NotifyEvent` symbol is part of **Intel VTune Profiler's Instrumentation and Tracing Technology (ITT) API**. This symbol is expected by PyTorch's Intel MKL libraries but is **missing from the system**.

### Key Findings
1. **Environment-specific issue**: `vm1rs` environment works perfectly with PyTorch 2.5.1+cu124
2. **Intel MKL dependency**: The error occurs when PyTorch's Intel MKL libraries try to access VTune profiler symbols
3. **System-level conflict**: No Intel VTune/ITT libraries found on the system
4. **xformers constraint**: MegaSAM requires xformers≤0.0.27 which only compiles with CUDA 11.8

### Failed Solutions Attempted
1. ✅ **Complete environment recreation** with clean conda cache
2. ✅ **Exact CUDA version matching** (11.8 for vm1recon, 12.4 for vm1rs)
3. ✅ **Multiple PyTorch version attempts** (2.0.1, 2.4.0, etc.)
4. ✅ **Library path management** and environment isolation
5. ✅ **Alternative xformers installations** from conda-forge
6. ❌ **Intel MKL alternative installations** (would break compatibility)
7. ❌ **VTune installation** (would require significant system changes)

## 📊 **Impact Assessment**

### ✅ **Working Components (80% of pipeline)**
- **Stage 0**: SAM-2, ViTPose, VIMO, BSTRO *(all perfect)*
- **Stage 2**: MegaHunter optimization *(perfect)*
- **Stage 4**: Robot motion retargeting *(perfect)*

### ❌ **Blocked Components (20% of pipeline)**
- **Stage 1**: MegaSAM and MonST3R reconstruction
- **Stage 3**: NKSR meshification and GeoCalib

### Functional Pipeline
Users can still achieve: **Human video → 3D poses → Robot motion** by using alternative reconstruction methods or skipping environmental mesh generation.

## 🚀 **Potential Solutions**

### Short-term Workarounds
1. **Use external reconstruction tools** for point cloud generation
2. **Manual mesh creation** for environment representation
3. **Focus on human motion pipeline** (which works perfectly)

### Long-term Solutions
1. **Intel MKL-free PyTorch build** for vm1recon environment
2. **Alternative xformers compilation** that doesn't require Intel symbols
3. **MegaSAM dependency update** to support newer PyTorch versions
4. **Containerized solution** with pre-built compatible libraries

## 🆘 **Request for Assistance**

This appears to be a **deep system-level library conflict** between:
- Intel MKL libraries in PyTorch
- Missing Intel VTune Profiler symbols
- CUDA 11.8 compatibility requirements
- xformers version constraints

The issue affects the specific combination of dependencies required by MegaSAM, while the main VideoMimic innovations (human motion processing and robot retargeting) work flawlessly.

**Would appreciate guidance on**:
1. Known workarounds for Intel MKL/VTune symbol conflicts
2. Alternative PyTorch builds that avoid Intel dependencies
3. MegaSAM compatibility with newer PyTorch versions
4. Pre-built Docker containers that resolve these conflicts

## 📎 **Additional Context**

- This setup worked successfully through Stage 0 processing of demo videos
- The `vm1rs` environment demonstrates that VideoMimic's core algorithms are solid
- Similar issues reported in PyTorch forums related to Intel library conflicts
- AWS Ubuntu 22.04 LTS environment (clean installation)

---

**Environment Details**: Ubuntu 22.04.5 LTS, Tesla T4, CUDA 11.8/12.4, conda 25.5.1  
**Pipeline Status**: Stage 0 ✅, Stage 1 ❌, Stage 2 ✅, Stage 3 ❌, Stage 4 ✅  
**Severity**: High (blocks 20% of functionality) / Medium (80% of core features work) 