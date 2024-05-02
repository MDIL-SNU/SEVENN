

<img src="SevenNet_logo.png" alt="Alt text" height="180">


# SevenNet

SevenNet (Scalable EquiVariance Enabled Neural Network) is a graph neural network interatomic potential package that supports parallel molecular dynamics simulations with [`LAMMPS`](https://github.com/lammps/lammps). Its underlying GNN model is based on [`nequip`](https://github.com/mir-group/nequip).

The project provides parallel molecular dynamics simulations using graph neural network interatomic potentials, which was not possible despite their superior performance.

**PLEASE NOTE:** SevenNet is under active development and may not be fully stable.

The installation and usage of SevenNet are split into two parts: training (handled by PyTorch) and molecular dynamics (handled by [`LAMMPS`](https://github.com/lammps/lammps)). The model, once trained with PyTorch, is deployed using TorchScript and is later used to run molecular dynamics simulations via LAMMPS.

- [SevenNet](#sevennet)
  * [Installation](#installation)
  * [Usage](#usage)
    + [SevenNet-0](#sevennet-0)
    + [SevenNet Calculator for ASE](#sevennet-calculator-for-ase)
    + [Training sevenn](#training)
      - [Multi-GPU training](#multi-gpu-training)
    + [sevenn_inference](#sevenn_inference)
    + [sevenn_get_model](#sevenn_get_model)
  * [Installation for LAMMPS](#installation-for-lammps)
    + [Notes](#notes)
  * [Usage for LAMMPS](#usage-for-lammps)
    + [To check installation](#to-check-installation)
    + [For serial model](#for-serial-model)
    + [For parallel model](#for-parallel-model)
  * [Future Works](#future-works)
  * [Citation](#citation)

## Installation

* Python >= 3.8
* PyTorch >= 1.11
* [`TorchGeometric`](https://pytorch-geometric.readthedocs.io/en/latest/install/installation.html)
* [`pytorch_scatter`](https://github.com/rusty1s/pytorch_scatter)

You can find the installation guides for these packages from the [`PyTorch official`](https://pytorch.org/get-started/locally/), [`TorchGeometric docs`](https://pytorch-geometric.readthedocs.io/en/latest/install/installation.html) and [`pytorch_scatter`](https://github.com/rusty1s/pytorch_scatter). Remember that these packages have dependencies on your CUDA version.

```
git clone https://github.com/MDIL-SNU/SevenNet.git
cd SevenNet
pip install .
```

## Usage

### SevenNet-0
SevenNet-0 is a general-purpose interatomic potential trained on the [`Material Project dataset of M3GNet`](https://figshare.com/articles/dataset/MPF_2021_2_8/19470599). You can try SevenNet-0 to your application without any training. If the accuracy was unsatisfactory, SevenNet-0 can be [fine-tuned](#Training).

Checkpoint of SevenNet-0 (for use in ASE or fine-tuning) and deployed potentials (for LAMMPS) are located in `{path_to_SevenNet}/pretrained_potentials/SevenNet_0`.

For its detailed usage, please check [SevenNet Calculator for ASE](#sevennet-calculator-for-ase), [For serial model](#for-serial-model), and [For parallel model](#for-parallel-model)

### SevenNet Calculator for ASE

```python
from sevenn.sevennet_calculator import SevenNetCalculator
checkpoint_path = ### PATH TO CHECKPOINT ###
sevenet_cal = SevenNetCalculator(checkpoint_path, device='cpu')
```

If you want to use SevenNet-0, you can do something like below
```bash
echo "export SEVENNET_0_CP={PATH_TO_SEVENNET}/pretrained_potentials/SevenNet_0/checkpoint_sevennet_0.pth" >> ~/.bashrc
```
SevenNetCalculator tries to read the SEVENNET_0_CP environment variable.

```python
from sevenn.sevennet_calculator import SevenNetCalculator
sevenet_0_cal = SevenNetCalculator(device='cpu')
```

### Training

```
cd example_inputs/training
sevenn input_full.yaml -s
```

Examples of `input_full.yaml` can be found under `SevenNet/example_inputs`. The `structure_list` file is used to select VASP OUTCARs for training.
To reuse a preprocessed training set, you can specify `${dataset_name}.sevenn_data` to the `load_dataset_path:` in the `input.yaml`.

Once you initiate training, `log.sevenn` will contain all parsed inputs from `input.yaml`. Any parameters not specified in the input will be automatically assigned as their default values. You can refer to the log to check the default inputs.
Currently, detailed explanations of model hyperparameters can be found at [`nequip`](https://github.com/mir-group/nequip), or inside input_full.yaml.

#### Multi-GPU training

We support multi-GPU training features using PyTorch DDP (distributed data parallel). We use one process (CPU core) per GPU.
```
torchrun --standalone --nnodes={# of nodes} --nproc_per_node {# of GPUs} --no_python sevenn input.yaml -d
```
Please note that `batch_size` in input.yaml indicates `batch_size` per GPU.

### sevenn_inference

Assuming that you've done temporal training of 10 epochs by above "To start training using 'sevenn'", try below at the same directory
```
sevenn_inference checkpoint_best.pt ../data/label_1/*
```

This will create dir 'sevenn_infer_result'. It includes .csv files that enumerate prediction/reference results of energy and force on OUTCARs in `data/label_1` directory.
You can try `sevenn_inference --help` for more information on this command.

### sevenn_get_model

Assuming that you've done temporal training of 10 epochs by above "To start training using 'sevenn'", try below at the same directory
```
sevenn_get_model checkpoint_best.pt
```

This will create `deployed_serial.pt`, which can be used as lammps potential under `e3gnn` pair_style. Please take a look at the lammps installation process below.

The parallel model can be obtained in a similar way
```
sevenn_get_model checkpoint_best.pt -p
```

This will create multiple `deployed_parallel_*.pt` files. The number of deployed models equals the number of message-passing layers.
These models can be used as lammps potential to run parallel MD simulations with GNN potential using multiple GPU cards.

## Installation for LAMMPS 

* PyTorch (same version as used for training)
* LAMMPS version of 'stable_2Aug2023' [`LAMMPS`](https://github.com/lammps/lammps)
* [`CUDA-aware OpenMPI`](https://www.open-mpi.org/faq/?category=buildcuda) for parallel MD

**PLEASE NOTE:** CUDA-aware OpenMPI does not support NVIDIA Gaming GPUs. Given that the software is closely tied to hardware specifications, please consult with your server administrator if it is not available.

Ensure the LAMMPS version (stable_2Aug2023). You can easily switch the version using git from a shell.
```
$ git clone https://github.com/lammps/lammps.git lammps_dir
$ cd lammps_dir
$ git checkout stable_2Aug2023_update3
```

Note that the following commands will overwrite `comm_brick.cpp` and `comm_brick.h` in the original `LAMMPS`. While it does not affect the original functionality of `LAMMPS`, you may want to back up these files from the source if you're unsure.

```
cp {path_to_SevenNet}/pair_e3gnn/* path_to_lammps/src/
```

If you have correctly installed CUDA-aware OpenMPI, the remaining process is identical to [`pair-nequip`](https://github.com/mir-group/pair_nequip).

Please make the following modifications to lammps/cmake/CMakeLists.txt:
Change `set(CMAKE_CXX_STANDARD 11)` to `set(CMAKE_CXX_STANDARD 14)`.
Then append the following lines in the same file:

```
find_package(Torch REQUIRED)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${TORCH_CXX_FLAGS}")
target_link_libraries(lammps PUBLIC "${TORCH_LIBRARIES}")
```

To build lammps with cmake:

```
cd lammps
mkdir build
cd build
cmake ../cmake -DCMAKE_PREFIX_PATH=`python -c 'import torch;print(torch.utils.cmake_prefix_path)'`
```

### Notes
You can check whether your OpenMPI is CUDA-aware by using `ompi_info` command:

```
$ ompi_info --parsable --all | grep mpi_built_with_cuda_support:value
mca:mpi:base:param:mpi_built_with_cuda_support:value:true
```
We're currently developing other options (other than CUDA-aware OpenMPI) to leverage parallel MD. Please let us know other inter-GPU communication backends you want for the SevenNet.

Incompatibility with other LAMMPS versions usually arises from the modified comm_brick.h/cpp. We will seek a more proper multi-GPU communication implementation without changing the original LAMMPS code.
However, if you have to use other versions of LAMMPS, you can patch your LAMMPS version by simply putting
void forward_comm(class PairE3GNNParallel *);
void reverse_comm(class PairE3GNNParallel *);
these function declarations under 'public:' of comm_brick.h and copy-pasting bodies of the above functions from 'our' comm_brick.cpp to yours.
Also, don't forget to `include "pair_e3gnn_parallel.h"` in comm_brick.cpp

## Usage for LAMMPS

### To check installation

For serial MD,
```
$ cd ${path_to_SevenNetetet}/example_inputs/md_serial_example
$ {lammps_binary} -in in.lmp

###lammps outputs for 5 MD steps###

$ grep PairE3GNN log.lammps
PairE3GNN using device : CUDA
```

For parallel MD
```
$ cd ${path_to_SevenNet}/example_inputs/md_serial_example
$ mpirun -np {# of GPUs you want to use} {lammps_binary} -in in.lmp

###lammps outputs for 5 MD steps###

$ grep PairE3GNN log.lammps
PairE3GNNParallel using device : CUDA
PairE3GNNParallel cuda-aware mpi : True
```

Example MD input scripts for `LAMMPS` can be found under `SevenNet/example_inputs`. If you've correctly installed `LAMMPS`, there are two additional pair styles available: `e3gnn` and `e3gnn/parallel`.

In the `pair_coeff` of the lammps script, you need to provide the path of the trained models (either serial or parallel). For parallel models, you should specify how many segmented models will be used.

### For serial model

```
pair_style e3gnn
pair_coeff * * {path to serial model} {chemical species}
```

Note that SevenNet-0 serial model is located in `{PATH TO SEVENNET}/pretrained_potentials/SevenNet_0/serial_model/deployed_serial.pt`.

### For parallel model

```
pair_style e3gnn/parallel
pair_coeff * * {number of segmented parallel models} {space separated paths of segmented parallel models} {chemical species}
```

Note that SevenNet-0 parallel model is located in `{PATH TO SEVENNET}/pretrained_potentials/SevenNet_0/parallel_model/deployed_parallel_*.pt`.
I recommend using variables to handle file paths for parallel models.

```
pair_style e3gnn/parallel 
pair_coeff * * 5 ${pre}/deployed_parallel_0.pt ${pre}/deployed_parallel_1.pt ${pre}/deployed_parallel_2.pt ${pre}/deployed_parallel_3.pt ${pre}/deployed_parallel_4.pt {chemical species}
```

Now, you can execute this LAMMPS script with a prefix for parallel models

```
mpirun -np {# of GPUS to use} {LAMMPS_binary} -in {LAMMPS_script} -var pre {PATH TO SEVENNET}/pretrained_potentials/SevenNet_0/parallel_model/
```

If a CUDA-aware OpenMPI is not found (it detects automatically in the code), `e3gnn/parallel` will not utilize GPUs even if they are available. You can check whether `OpenMPI` is found or not from the standard output of the `LAMMPS` simulation. Ideally, one GPU per MPI process is expected. If the available GPUs are fewer than the MPI processes, the simulation may run inefficiently. You can select specific GPUs by setting the `CUDA_VISIBLE_DEVICES` environment variable.

## Future Works

* Implementation of pressure output in parallel MD simulations.
* Development of support for a tiled communication style (also known as recursive coordinate bisection, RCB) in LAMMPS.
* Easy use of parallel models

## Citation
If you use SevenNet, please cite (1) parallel GNN-IP MD simulation by SevenNet or its pre-trained model SevenNet-0, (2) underlying GNN-IP architecture NequIP

(1) Y. Park, J. Kim, S. Hwang, and S. Han, "Scalable Parallel Algorithm for Graph Neural Network Interatomic Potentials in Molecular Dynamics Simulations". arXiv, arXiv:2402.03789. (2024) (https://arxiv.org/abs/2402.03789)

(2) S. Batzner, A. Musaelian, L. Sun, M. Geiger, J. P. Mailoa, M. Kornbluth, N. Molinari, T. E. Smidt, and B. Kozinsky, "E (3)-equivariant graph neural networks for data-efficient and accurate interatomic potentials". Nat. Commun., 13, 2453. (2022) (https://www.nature.com/articles/s41467-022-29939-5)
