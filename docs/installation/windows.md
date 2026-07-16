# Installation — Windows

## Prerequisites

- [Unity 6000.0+](https://unity3d.com/get-unity/download)
- [Conda](https://docs.conda.io/en/latest/) or [Mamba](https://github.com/mamba-org/mamba)

## Installation

1. Clone the repository:

   ```bash
   git clone https://github.com/AISoc-UNSW/FallGuys.git
   cd FallGuys
   ```

2. Create and activate the environment:

   ```bash
   conda env create -f environment.yml
   conda activate mlagents
   ```

3. *(For GPU support)* Install PyTorch with CUDA:

   ```bash
   pip install torch~=2.2.1 --index-url https://download.pytorch.org/whl/cu121
   ```

   If prompted, install the Microsoft Visual C++ Redistributable.

## Verify installation

```bash
mlagents-learn --help
```

If a list of parameters is printed, you're good to go.

## Still stuck?

Refer to the first half of [this video](https://www.youtube.com/watch?v=ZtbjlrmRbyc) if you can't figure out the setup.

!!! note
    We've already set up all the required packages for you in `environment.yml`, so you can just use that for installation instead of the manual conda method shown in the video.
