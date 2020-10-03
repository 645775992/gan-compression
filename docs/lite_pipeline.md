# GAN Compression Lite Training Tutorial
## Prerequisites

* Linux
* Python 3
* CPU or NVIDIA GPU + CUDA CuDNN

## Getting Started

### Preparations

Please refer to our [README](../README.md) for the installation, dataset preparations, and the evaluation (FID and mIoU).

### Pipeline

Below we show a lite pipeline for compressing pix2pix and cycleGAN models. **We provide pre-trained models after each step. You could use the pretrained models to skip some steps.** For more training details, please refer to our codes.

## Pix2pix Model Compression

We will show the whole pipeline on `edges2shoes-r` dataset. You could change the dataset name to other datasets (such as `map2sat`).

##### Train an Original Full Teacher Model (if you already have the full model, you could skip it)

Train an original full teacher model from scratch.
```shell
bash scripts/pix2pix/edges2shoes-r/train_full.sh
```
We provide a pre-trained teacher for each dataset. You could download the pre-trained model by
```shell
python scripts/download_model.py --model pix2pix --task edges2shoes-r --stage full
```

and test the model by

```shell
bash scripts/pix2pix/edges2shoes-r/test_full.sh
```

##### "Once-for-all" Network Training

Train a "once-for-all" network from scratch to search for the efficient architectures.

```shell
bash scripts/pix2pix/edges2shoes-r_lite/train_supernet.sh
```

We provide a trained once-for-all network for each dataset. You could download the model by

```shell
python scripts/download_model.py --model pix2pix --task edges2shoes-r_lite --stage supernet
```

##### Select the Best Model

* **[New!!!]** Evolution Searching.

  The evolution searching uses the evolution algorithm to search for the best-performed subnet. It is much much faster than the brute force searching. You could run:

  ```bash
  bash scripts/pix2pix/edges2shoes-r_lite/evolution_search.sh
  ```

  It will directly tells you the information of the best-performed subnet which satisfies the computation budget in the following format:

  ```
  {config_str: $config_str, macs: $macs, fid/mIoU: $fid_or_mIoU}
  ```

* **[Previous]** Brute Force Searching.

  The brute force searching would evolution all the candidate sub-networks meeting the computation budget given a specific searching space. It is much more slower that the evolution searching. You could run:

  ```shell
  bash scripts/pix2pix/edges2shoes-r_lite/search.sh
  ```

  The search result will be stored in the python `pickle` form. The pickle file is a python `list` object that stores all the candidate sub-networks information, whose element is a python `dict ` object in the form of

  ```
  {'config_str': $config_str, 'macs': $macs, 'fid'/'mIoU': $fid_or_mIoU}
  ```

  such as

  ```python
  {'config_str': '32_32_48_40_64_40_16_32', 'macs': 5761662976, 'fid': 30.594936138634836}
  ```

  `'config_str'` is a channel configuration description to identify a specific subnet within the "once-for-all" network.

  To accelerate the search process, you may need to want to search the sub-networks on multiple GPUs. You could manually split the search space with [search.py](../search.py). All you need to do is add additional arguments `--num_splits` and `--split`. For example, if you need to search the sub-networks  with 2 GPUs, you could use the following commands:

  * On the first GPU:

    ```bash
    python search.py --dataroot database/edges2shoes-r \
      --restore_G_path logs/pix2pix/edges2shoes-r_lite/supernet-stage2/checkpoints/latest_net_G.pth \
      --output_path logs/pix2pix/edges2shoes-r_lite/supernet-stage2/pkls/result0.pkl \
      --ngf 64 --batch_size 32 \
      --config_set channels-64-pix2pix \
      --real_stat_path real_stat/edges2shoes-r_B.npz --load_in_memory --budget 6.5e9 \
      --num_splits 2 --split 0
    ```

  * On the second GPU:

    ```bash
    python search.py --dataroot database/edges2shoes-r \
      --restore_G_path logs/pix2pix/edges2shoes-r_lite/supernet-stage2/checkpoints/latest_net_G.pth \
      --output_path logs/pix2pix/edges2shoes-r_lite/supernet-stage2/pkls/result1.pkl \
      --ngf 64 --batch_size 32 \
      --config_set channels-64-pix2pix \
      --real_stat_path real_stat/edges2shoes-r_B.npz --load_in_memory --budget 6.5e9 \
      --num_splits 2 --split 1 --gpu_ids 1
    ```

  Then you could merge the search results with [merge.py](../merge.py)

  ```bash
  python merge.py --input_dir logs/pix2pix/edges2shoes-r_lite/supernet-stage2/pkls \
    --output_path logs/cycle_gan/horse2zebra/supernet
  ```

  Once you get the search results, you could use our auxiliary script [select_arch.py](../select_arch.py) to select the architecture you want.

  ```shell
  python select_arch.py --macs 6.5e9 --fid 32 \ 
    --pkl_path logs/pix2pix/edges2shoes-r/supernet/result.pkl
  ```

  ##### Fine-tuning the Best Model

(Optional) Fine-tune a specific subnet within the pre-trained "once-for-all" network. To further improve the performance of your chosen subnet, you may need to fine-tune the subnet. For example, if you want to fine-tune a subnet within the "once-for-all" network with `'config_str': 32_32_40_40_40_64_16_16`, use the following command:

```shell
bash scripts/pix2pix/edges2shoes-r_lite/finetune.sh 32_32_48_40_64_40_16_32
```

##### Export the Model

Extract a subnet from the "once-for-all" network. We provide a code [export.py](../export.py) to extract a specific subnet according to a configuration description. For example, if the `config_str` of your chosen subnet is `32_32_40_40_40_64_16_16`, then you can export the model by this command:

```shell
bash scripts/pix2pix/edges2shoes-r_lite/export.sh 32_32_40_40_40_64_16_16
```

## CycleGAN Model Compression

The pipeline is almost identical to pix2pix. We will show the pipeline on `horse2zebra` dataset.

##### Train an Original Full Teacher Model (if you already have the full model, you could skip it)

Train an original full teacher model from scratch.

```shell
bash scripts/cycle_gan/horse2zebra/train_full.sh
```

We provide a pre-trained teacher model for each dataset. You could download the model using

```shell
python scripts/download_model.py --model cycle_gan --task horse2zebra --stage full
```

and test the model by

```shell
bash scripts/cycle_gan/horse2zebra/test_full.sh
```

##### "Once-for-all" Network Training

Train a "once-for-all" network from scratch to search for the efficient architectures.

```shell
bash scripts/cycle_gan/horse2zebra_lite/train_supernet.sh
```

We provide a pre-trained once-for-all network for each dataset. You could download the model by

```shell
python scripts/download_model.py --model cycle_gan --task horse2zebra_lite --stage supernet
```

##### Select the Best Model

This stage is almost the same as pix2pix.

* **[New!!!]** Evolution Searching.

  ```bash
  bash scripts/cycle_gan/horse2zebra_lite/evolution_search.sh
  ```


* **[Previous]** Brute Force Searching.

  ```shell
bash scripts/cycle_gan/horse2zebra_lite/search.sh
  ```
  
  You could also use our auxiliary script [select_arch.py](../select_arch.py) to select the architecture you want. The usage is the same as pix2pix.

##### Fine-tuning the Best Model

During our experiments, we observe that fine-tuning the model on horse2zebra increases FID.  **You may skip the fine-tuning.**

##### Export the Model

Extract a subnet from the supernet. We provide a code [export.py](../export.py) to extract a specific subnet according to a configuration description. For example, if the `config_str` of your chosen subnet is `16_16_24_16_32_64_16_24`, then you can export the model by this command:

```shell
bash scripts/cycle_gan/horse2zebra_lite/export.sh 16_16_24_16_32_64_16_24
```

## GauGAN Model Compression

The pipeline is almost identical to pix2pix. We will show the pipeline on `cityscapes` dataset.

##### Train an Original Full Teacher Model (if you already have the full model, you could skip it)

Train an original full teacher model from scratch.

```shell
bash scripts/gaugan/cityscapes/train_full.sh
```

We provide a pre-trained teacher model for each dataset. You could download the model using

```shell
python scripts/download_model.py --model gaugan --task cityscapes --stage full
```

and test the model by

```shell
bash scripts/gaugan/cityscapes/test_full.sh
```

##### "Once-for-all" Network Training

**Note:** If your original full model uses spectral norm, please remove it before the "once-for-all" network training. You could remove it in this way:

```bash
python remove_spectral_norm.py --netG spade \
  --restore_G_path logs/gaugan/cityscapes/full/checkpoints/latest_net_G.pth \
  --output_path logs/gaugan/cityscapes/full/export/latest_net_G.pth
```

Train a "once-for-all" network from scratch to search for the efficient architectures.

```shell
bash scripts/gaugan/cityscapes_lite/train_supernet.sh
```

We provide a pre-trained once-for-all network for each dataset. You could download the model by

```shell
python scripts/download_model.py --model gaugan --task cityscapes_lite --stage supernet
```

##### Select the Best Model

This stage is almost the same as pix2pix.

* **[New!!!]** Evolution Searching.

  ```bash
  bash scripts/gaugan/cityscapes_lite/evolution_search.sh
  ```


* **[Previous]** Brute Force Searching.

  ```shell
bash scripts/gaugan/cityscapes_lite/search.sh
  ```
  
  You could also use our auxiliary script [select_arch.py](../select_arch.py) to select the architecture you want. The usage is the same as pix2pix.

##### Fine-tuning the Best Model

(Optional) Fine-tune a specific subnet within the pre-trained "once-for-all" network. To further improve the performance of your chosen subnet, you may need to fine-tune the subnet. For example, if you want to fine-tune a subnet within the "once-for-all" network with `'config_str': 32_32_32_48_32_24_24_32`, use the following command:

```shell
bash scripts/gaugan/cityscapes_lite/finetune.sh 32_32_32_48_32_24_24_32
```

##### Export the Model

Extract a subnet from the supernet. We provide a code [export.py](../export.py) to extract a specific subnet according to a configuration description. For example, if the `config_str` of your chosen subnet is `32_32_32_48_32_24_24_32`, then you can export the model by this command:

```shell
bash scripts/gaugan/cityscapes_lite/export.sh 32_32_32_48_32_24_24_32
```