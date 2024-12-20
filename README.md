# FedMeZO
This repository contains the official implementation for the work **“On the convergence of Zeroth-Order Federated Tuning for Large Language Models”**, accepted by *KDD'2024 ([paper](https://dl.acm.org/doi/10.1145/3637528.3671865))*.

> The confluence of Federated Learning (FL) and Large Language Models (LLMs) is ushering in a new era in privacy-preserving natural language processing. However, the intensive memory requirements for fine-tuning LLMs pose significant challenges, especially when deploying on clients with limited computational resources. To circumvent this, we explore the novel integration of Memory-efficient Zeroth-Order Optimization within a federated setting, a synergy we term as FedMeZO. Our study is the first to examine the theoretical underpinnings of FedMeZO in the context of LLMs, tackling key questions regarding the influence of large parameter spaces on optimization behavior, the establishment of convergence properties, and the identification of critical parameters for convergence to inform personalized federated strategies. Our extensive empirical evidence supports the theory, showing that FedMeZO not only converges faster than traditional first-order methods such as FedAvg but also significantly reduces GPU memory usage during training to levels comparable to those during inference. Moreover, the proposed personalized FL strategy that is built upon the theoretical insights to customize the client-wise learning rate can effectively accelerate loss reduction. We hope our work can help to bridge theoretical and practical aspects of federated fine-tuning for LLMs, thereby stimulating further advancements and research in this area.

The purpose of this implementation is to provide an empirical support for our theoretical analysis.

## Project Structure

```python
.
├── federatedscope
│   ├── core                     # Federated learning backend modules
│   │   ├── trainers            
│   │   │   ├── trainer.py       # The strategies' implementation
│   │   │   ├── ...                          
│   ├── llm                      # Federated fine-tuning LLMs modules 
│   │   ├── baseline             # Scripts for LLMs
│   │   │   ├── frozen           # The frozen training scripts
│   │   │   ├── dynamic          # The dynamic training scripts
│   │   ├── dataloader           # Federated fine-tuning dataloader
│   │   ├── dataset              # Federated fine-tuning dataset
│   │   ├── model                # LLMs and Adapter
│   │   ├── trainer              # Fine-tuning with accerating operators
│   │   │   ├── mezo_trainer.py  # The trainer of LLMs by FedMeZO
│   │   │   ├── trainer.py       # The trainer of LLMs by BP-Based FedAvg
│   │   ├── ...
│   ├── main.py                  # Running interface
│   ├── ... ...          
├── ... ...                      
└── setup.py                     # The installation of this project
```

## Installation

```python
# Create virtual environments with conda
conda create -n fedmezo python=3.9
conda activate fedmezo

# Install Pytorch (e.g., Pytorch==2.0.0)
conda install pytorch==2.0.0 torchvision==0.15.0 torchaudio==2.0.0 pytorch-cuda=11.7 -c pytorch -c nvidia

# Install FedMeZO with editable mode
pip install -e .[llm]
```

To maintain the same environment settings as this work (PyTorch==1.10.1), you can follow the installation method below:
```python
# Create virtual environments with conda
conda create -n fedmezo python=3.9
conda activate fedmezo

# Install FedMeZO with editable mode
pip install -e .[llm]

# Install Pytorch 1.10.1
pip install torch==1.10.1+cu111 torchvision==0.11.2+cu111 torchaudio==0.10.1 -f https://download.pytorch.org/whl/cu111/torch_stable.html
```


## Model and Data Preparation

1. To run experiments on [LLaMA-3B](https://huggingface.co/openlm-research/open_llama_3b_v2) or other LLMs, you can manually modify `YOUR_MODEL_HERE` in `./federatedscope/llm/dataloader/dataloader.py` and `./federatedscope/llm/model/model_builder.py`

```python
# ./federatedscope/llm/dataloader/dataloader.py
... ...
tokenizer = AutoTokenizer.from_pretrained(
    YOUR_MODEL_HERE,
    cache_dir=cache_dir,
    model_max_length=tok_len,
    padding_side="right",
    use_fast=False,
    )
... ...

# ./federatedscope/llm/model/model_builder.py
... ...
return AutoModelForCausalLM.from_pretrained(YOUR_MODEL_HERE, **kwargs)
... ...
```

2. To run experiments on [Alpaca](https://github.com/bacoco/stanford_alpaca) / [GSM8K](https://github.com/openai/grade-school-math) / [Dolly-15K](https://github.com/databrickslabs/dolly) / [CodeAlpaca](https://github.com/sahil280114/codealpaca), you can use default settings. Or unzip the downloaded datasets in directory `./federatedscope/llm/dataset` and manually modify the code in `./federatedscope/llm/dataloader/dataloader.py` ([Alpaca](https://github.com/bacoco/stanford_alpaca) as an example below)

```python
... ...
elif dataset_name.lower() == 'alpaca':
    fp = './federatedscope/llm/dataset/alpaca/alpaca_data.json'
    list_data_dict = load_json(fp)
    dataset = LLMDataset(list_data_dict, tokenizer)
... ...
```

## Running Examples

We provide several example scripts to conduct the experiments. There is a unique configuration `train.train_strategy` in FedMeZO framework, that defines the training strategy. This includes four strategies: `'frozen'`, which uses a static learning rate; `'random'`, which sets the learning rate randomly each round; `'round-wise'`, which employs a dynamic strategy based on the difference in loss per round; `'five-round'`, which uses a dynamic strategy based on the average loss difference every five rounds; and `'model-diff'`, which applies a dynamic strategy based on the difference in parameter's update per round. For more detailed strategy settings, please refer to our paper.
```
...
train:
  # The strategy of training
  train_strategy: 'frozen'

  local_update_steps: 30
  batch_or_epoch: batch
  optimizer:
    lr: 0.00001
    weight_decay: 0.0
  is_enable_half: True
...
```

1. To check whether the environment is **successfully installed**, you can use the following example to test whether the program can run.
```python
python federatedscope/main.py --cfg federatedscope/llm/baseline/mezo_testcase.yaml
```

2. **Frozen BP-Based FedAvg** on `Alpaca` with `IID-Splitter` / `GSM-8K` with `IID-Splitter` / `Dolly-15K` with `Meta-Splitter` / `CodeAlpaca` with `LDA-Splitter`
```python
# Alpaca with IID-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/alpaca/alpaca_bpbased_iid.yaml

# GSM-8K with IID-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/gsm8k/gsm8k_bpbased_iid.yaml

# Dolly-15K with Meta-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/dolly/dolly_bpbased_meta.yaml

# CodeAlpaca with LDA-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/code/code_bpbased_lda.yaml
```
   
3. **Frozen FedMeZO** on `Alpaca` with `IID-Splitter` / `GSM-8K` with `IID-Splitter` / `Dolly-15K` with `Meta-Splitter` / `CodeAlpaca` with `LDA-Splitter`
```python
# Alpaca with IID-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/alpaca/alpaca_mezo_iid.yaml

# GSM-8K with IID-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/gsm8k/gsm8k_mezo_iid.yaml

# Dolly-15K with Meta-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/dolly/dolly_mezo_meta.yaml

# CodeAlpaca with LDA-Splitter
python federatedscope/main.py --cfg federatedscope/llm/baseline/frozen/code/code_mezo_lda.yaml
```

4. **Dynamic FedMeZO** on `Alpaca` with `IID-Splitter` by **strategy** `'random'` / `'round-wise'` / `'five-round'` / `'model-diff'`
```python
# 'frozen' strategy
python federatedscope/main.py --cfg federatedscope/llm/baseline/dynamic/alpaca_frozen.yaml

# 'random' strategy
python federatedscope/main.py --cfg federatedscope/llm/baseline/dynamic/alpaca_random.yaml

# 'round-wise' strategy
python federatedscope/main.py --cfg federatedscope/llm/baseline/dynamic/alpaca_round-wise.yaml

# 'five-round' strategy
python federatedscope/main.py --cfg federatedscope/llm/baseline/dynamic/alpaca_five-round.yaml

# 'model-diff' strategy
python federatedscope/main.py --cfg federatedscope/llm/baseline/dynamic/alpaca_model-diff.yaml
```

## References
If you find our work useful for your research or development, please kindly cite the following [paper](https://dl.acm.org/doi/10.1145/3637528.3671865).
```
@inproceedings{ling2024convergence,
  title={On the convergence of zeroth-order federated tuning for large language models},
  author={Ling, Zhenqing and Chen, Daoyuan and Yao, Liuyi and Li, Yaliang and Shen, Ying},
  booktitle={Proceedings of the 30th ACM SIGKDD Conference on Knowledge Discovery and Data Mining},
  pages={1827--1838},
  year={2024}
}
```
