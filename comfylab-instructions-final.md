# ComfyLab Setup Instructions (Final)

This document outlines the steps needed to set up a complete ComfyUI environment with WAN2.1 integration using Poetry.

## Prerequisites

1. **NVIDIA GPU with CUDA support**
   - Install NVIDIA Driver 550 or later
   - CUDA 12.4 toolkit should be installed
   - Recommended: 16GB+ VRAM for WAN2.1 video generation

2. **Python Environment**
   - Python 3.12.3 is recommended and tested
   - Earlier versions (>=3.10) may work but are untested

3. **Poetry**
   - Install Poetry: https://install.python-poetry.org
   - Recommended version: 1.7.0+ (tested with 1.8.2)
   - Install with: `curl -sSL https://install.python-poetry.org | python3 -`
   - Add to PATH: `export PATH="$HOME/.local/bin:$PATH"`

4. **System Packages**
   - Required packages: `sudo apt install python3-dev python3-pip python3-venv build-essential git`
   - For Python 3.12: `sudo apt install python3.12-venv`
   - For Rust extensions: `sudo apt install build-essential libssl-dev pkg-config`

## Quick Start (After Setup)

Run the launcher script to start ComfyUI:

```bash
bash run-comfylab.sh
```

## Step 1: Create project directory & set up Poetry

Create a new directory and navigate to it:

```bash
mkdir -p comfylab
```

Move into the directory:

```bash
cd comfylab
```

Note: Copy the pyproject.toml file into this location (~/comfylab/pyproject.toml)

Use Python 3.12 for Poetry environment:

```bash
poetry env use python3.12
```

Set Poetry to create the virtual environment in the project directory:

```bash
poetry config virtualenvs.in-project true
```

Configure Poetry to use the old installer (modern one has issues with some packages):

```bash
poetry config installer.modern-installation false
```

Set worker limit for compilation (adjust based on CPU cores):

```bash
poetry config --local installer.max-workers 4
```

Install base dependencies without conflicting package groups:

```bash
poetry install --without comfyui_deps,wan_deps,manager_deps,dev -v
```

## Step 2: Clone and Set Up Core Repositories

Activate the Poetry environment:

```bash
poetry shell
```

Clone ComfyUI repository:

```bash
git clone https://github.com/comfyanonymous/ComfyUI.git
```

Install ComfyUI requirements (without dependencies):

```bash
cd ComfyUI
```

```bash
poetry run pip install --no-deps -r requirements.txt
```

```bash
cd ..
```

Clone WAN2.1 repository:

```bash
git clone https://github.com/Wan-Video/Wan2.1.git
```

Install WAN2.1 requirements (without dependencies):

```bash
cd Wan2.1
```

```bash
poetry run pip install --no-deps -r requirements.txt
```

```bash
cd ..
```

## Step 3: Install Extensions

### Option 1: Individual Extension Installation

Clone the ComfyUI-Manager repository:

```bash
cd ComfyUI/custom_nodes
```

```bash
git clone https://github.com/ltdrdata/ComfyUI-Manager.git
```

Install ComfyUI-Manager requirements:

```bash
cd ComfyUI-Manager
```

```bash
poetry run pip install --no-deps -r requirements.txt
```

```bash
cd ..
```

Clone WAN2.1 support extension:

```bash
git clone https://github.com/kijai/ComfyUI-WanVideoWrapper.git
```

Install WanVideoWrapper requirements:

```bash
cd ComfyUI-WanVideoWrapper
```

```bash
poetry run pip install --no-deps -r requirements.txt
```

```bash
cd ..
```

Clone TeaCache for performance optimization:

```bash
git clone https://github.com/welltop-cn/ComfyUI-TeaCache.git
```

Install TeaCache requirements:

```bash
cd ComfyUI-TeaCache
```

```bash
poetry run pip install --no-deps -r requirements.txt
```

```bash
cd ..
```

Clone ComfyUI-Copilot:

```bash
git clone https://github.com/AIDC-AI/ComfyUI-Copilot.git
```

Clone rgthree-comfy for UI improvements:

```bash
git clone https://github.com/rgthree/rgthree-comfy.git
```

Install rgthree requirements:

```bash
cd rgthree-comfy
```

```bash
poetry run pip install --no-deps -r requirements.txt
```

```bash
cd ..
```

### Option 2: Bulk Extension Installation

Alternatively, you can install all extensions at once. This script handles both requirements.txt and setup.py files:

```bash
# Create an installation script
cat > install_extensions.sh << 'EOF'
#!/bin/bash

# Loop through all custom_nodes directories
for node_dir in ./ComfyUI/custom_nodes/*/; do
  echo "Processing extension: $node_dir"
  
  # Install from requirements.txt if available
  if [ -f "${node_dir}requirements.txt" ]; then
    echo "  Installing requirements.txt..."
    poetry run pip install --no-deps -r "${node_dir}requirements.txt"
  fi
  
  # Install from setup.py if available
  if [ -f "${node_dir}setup.py" ]; then
    echo "  Installing setup.py..."
    (cd "$node_dir" && poetry run pip install --no-deps -e .)
  fi
  
  # Install from pyproject.toml if available
  if [ -f "${node_dir}pyproject.toml" ]; then
    echo "  Installing pyproject.toml..."
    (cd "$node_dir" && poetry run pip install --no-deps -e .)
  fi
done
EOF
```

Make the script executable and run it:

```bash
chmod +x install_extensions.sh
```

```bash
./install_extensions.sh
```

Return to the project root:

```bash
cd ../../..
```

## Step 4: Configure Environment Paths

To ensure ComfyUI and ComfyUI-Manager know about your Poetry environment, add these variables:

```bash
# Get the Poetry virtual environment path
VENV_PATH=$(poetry env info -p)
```

```bash
# Create an env_paths.json file for ComfyUI-Manager
cat > ComfyUI/custom_nodes/ComfyUI-Manager/env_paths.json << EOF
{
  "python_executable": "${VENV_PATH}/bin/python",
  "pip_executable": "${VENV_PATH}/bin/pip"
}
EOF
```

```bash
# Add PYTHONPATH to ComfyUI's environment
echo "export PYTHONPATH=\$PYTHONPATH:${VENV_PATH}/lib/python3.12/site-packages" >> .env
```

## Step 5: Install Special Packages

Install flash-attention 2.7.4.post1 without build isolation:

```bash
poetry run pip install flash-attn==2.7.4.post1 --no-build-isolation
```

## Step 6: Set Up Environment Variables

Create the .env file:

```bash
cat > .env << 'EOF'
# GPU selection (for multi-GPU systems)
export CUDA_VISIBLE_DEVICES=${CUDA_VISIBLE_DEVICES:-0}

# Memory management
export PYTORCH_CUDA_ALLOC_CONF=${PYTORCH_CUDA_ALLOC_CONF:-max_split_size_mb:512}

# Flash Attention optimization
export FLASH_ATTENTION_SKIP_VERSION_CHECK=${FLASH_ATTENTION_SKIP_VERSION_CHECK:-1}

# Parallel loading
export SAFETENSORS_FAST_GPU=${SAFETENSORS_FAST_GPU:-1}

# WAN2.1 specific
export WAN_CACHE_DIR=${WAN_CACHE_DIR:-"./models/wan_cache"}

# Build from source settings (when needed)
export TORCH_CUDA_ARCH_LIST=${TORCH_CUDA_ARCH_LIST:-"7.5;8.0;8.6"}  # Adjust for your GPU architecture
export MAX_JOBS=${MAX_JOBS:-4}  # Limit parallel compilation jobs
EOF
```

Load the environment variables:

```bash
source .env
```

## Step 7: Create Launcher Script

Create the launcher script:

```bash
cat > run-comfylab.sh << 'EOF'
#!/bin/bash
# Run me with: bash run-comfylab.sh

# Move to script directory (comfylab root)
cd "$(dirname "$0")"
ROOT_DIR=$(pwd)

# Load environment variables
source .env

# Activate Poetry environment
poetry shell <<EOF2
# Use absolute path to ComfyUI
cd $ROOT_DIR/ComfyUI
python main.py --listen 0.0.0.0 --port 8188 --normalvram --front-end-version Comfy-Org/ComfyUI_frontend@latest
EOF2
EOF
```

Make the script executable:

```bash
chmod +x run-comfylab.sh
```

## Step 8: Run ComfyUI

Make sure you're in the comfylab directory:

```bash
cd ~/comfylab
```

Run the launcher script:

```bash
./run-comfylab.sh
```

## Step 9: Validate Environment (Important)

Create a validation script to check for missing dependencies:

```bash
cat > validate_env.py << 'EOF'
import importlib.util
import sys
import subprocess
import os

# List of critical modules that must be present
required_modules = [
    'torch', 'torchvision', 'torchaudio', 'torchsde',  # PyTorch ecosystem
    'numpy', 'einops', 'safetensors',                   # Foundation libraries
    'transformers', 'tokenizers', 'diffusers',          # AI model libraries
    'flash_attn',                                       # Performance optimizations
    'kornia'                                            # Image processing
]

# Optional modules that enhance functionality but aren't critical
optional_modules = [
    'kornia_rs',                                        # Rust extension for kornia
    'spandrel', 'av', 'ftfy',                           # Media handling
    'gradio'                                            # UI components
]

print("\n===== ENVIRONMENT VALIDATION =====\n")

# Check Poetry environment
try:
    venv_path = subprocess.check_output(["poetry", "env", "info", "-p"], text=True).strip()
    print(f"✓ Poetry environment found at: {venv_path}")
except Exception as e:
    print(f"✗ Unable to locate Poetry environment: {e}")
    print("  Run this script with: poetry run python validate_env.py")
    sys.exit(1)

print("\n----- Required Modules -----")
missing_required = []
for module in required_modules:
    try:
        m = importlib.import_module(module)
        version = getattr(m, '__version__', 'unknown')
        print(f"✓ {module} (version: {version})")
    except ImportError as e:
        print(f"✗ {module} - {e}")
        missing_required.append(module)

print("\n----- Optional Modules -----")
missing_optional = []
for module in optional_modules:
    try:
        m = importlib.import_module(module)
        version = getattr(m, '__version__', 'unknown')
        print(f"✓ {module} (version: {version})")
    except ImportError as e:
        print(f"✗ {module} - {e}")
        missing_optional.append(module)

print("\n----- PyTorch CUDA Support -----")
try:
    import torch
    if torch.cuda.is_available():
        print(f"✓ CUDA available (version: {torch.version.cuda})")
        print(f"✓ Found {torch.cuda.device_count()} CUDA device(s)")
        for i in range(torch.cuda.device_count()):
            print(f"  - {torch.cuda.get_device_name(i)}")
    else:
        print("✗ CUDA not available in PyTorch")
except Exception as e:
    print(f"✗ Error checking CUDA: {e}")

# Summary and fix suggestions
print("\n===== VALIDATION SUMMARY =====\n")

if missing_required:
    print(f"❌ Missing {len(missing_required)} required module(s): {', '.join(missing_required)}")
    print("\nFix suggestions:")
    
    if 'torch' in missing_required or 'torchvision' in missing_required or 'torchaudio' in missing_required:
        print("- Install PyTorch ecosystem:")
        print("  poetry run pip install torch==2.4.0+cu124 torchvision==0.19.0+cu124 torchaudio==2.4.0+cu124 --index-url https://download.pytorch.org/whl/cu124")
    
    if 'flash_attn' in missing_required:
        print("- Install flash-attention:")
        print("  poetry run pip install flash-attn==2.7.4.post1 --no-build-isolation")
    
    for module in missing_required:
        if module not in ['torch', 'torchvision', 'torchaudio', 'flash_attn']:
            print(f"- Install {module}:")
            print(f"  poetry run pip install {module}")
else:
    print("✅ All required modules are installed!")

if missing_optional:
    print(f"\n⚠️ Missing {len(missing_optional)} optional module(s): {', '.join(missing_optional)}")
    print("\nOptional fix suggestions:")
    
    if 'kornia_rs' in missing_optional:
        print("- Install Rust first (if not installed):")
        print("  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh")
        print("  source \"$HOME/.cargo/env\"")
        print("- Then install kornia with Rust extensions:")
        print("  poetry run pip install kornia[rs]")
    
    for module in missing_optional:
        if module != 'kornia_rs':
            print(f"- Install {module}:")
            print(f"  poetry run pip install {module}")
else:
    print("✅ All optional modules are installed!")

print("\nTo fix all issues at once, you can create and run a fix script:")
print("  poetry run python -c \"import sys; open('fix_env.sh', 'w').write('#!/bin/bash\\n' + '\\n'.join([line[2:] for line in sys.stdin.read().split('\\n') if line.startswith('  poetry') or line.startswith('  curl')]))\" < <(python validate_env.py)")
print("  chmod +x fix_env.sh")
print("  ./fix_env.sh")
EOF
```

Run the validation script to check your environment:

```bash
poetry run python validate_env.py
```

If the validation script identifies missing dependencies, you can fix them using the suggested commands.

## Hardware Requirements

- **Minimum**: NVIDIA GPU with 8GB VRAM, CUDA 12.4 compatible
- **Recommended**: NVIDIA RTX 4070 or better with 16GB+ VRAM
- **WAN2.1 Video Generation**: 16GB+ VRAM 
- **Storage**: 30GB+ for models and cache
- **RAM**: 16GB+

## Troubleshooting

### CUDA Version Mismatches

Check if PyTorch sees CUDA:

```bash
poetry run python -c "import torch; print(f'CUDA available: {torch.cuda.is_available()}, Version: {torch.version.cuda if torch.cuda.is_available() else None}')"
```

Uninstall current PyTorch if needed:

```bash
poetry run pip uninstall -y torch torchvision torchaudio
```

Reinstall with CUDA support:

```bash
poetry run pip install torch==2.4.0+cu124 torchvision==0.19.0+cu124 torchaudio==2.4.0+cu124 --index-url https://download.pytorch.org/whl/cu124
```

### Flash-Attention Installation Issues

If you encounter errors with flash-attention:

```bash
poetry run pip install flash-attn==2.7.4.post1 --no-build-isolation
```

### Native Extension Dependencies (Like kornia_rs)

If you see errors about missing Rust-based extensions:

```bash
# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"

# Install system dependencies
sudo apt install build-essential libssl-dev pkg-config

# Install kornia with Rust extensions
poetry run pip install kornia[rs]
```

### Path Issues

If ComfyUI can't find models or custom nodes:

```bash
export PYTHONPATH=$PYTHONPATH:$(pwd)/ComfyUI
```

### Emergency Repair

If you need to start from a clean state but want to keep your existing environment:

```bash
# Get environment info
poetry env info

# Remove current virtualenv
poetry env remove python3.12

# Recreate virtualenv with the pyproject.toml changes
poetry install --without comfyui_deps,wan_deps,manager_deps,dev -v

# Reinstall required packages
poetry run pip install torch==2.4.0+cu124 torchvision==0.19.0+cu124 torchaudio==2.4.0+cu124 --index-url https://download.pytorch.org/whl/cu124
poetry run pip install flash-attn==2.7.4.post1 --no-build-isolation
```
