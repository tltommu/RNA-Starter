# RNA Folding Inference Model

This repository contains the inference pipeline for predicting RNA reactivity using a model trained in the [RNA starter kernel](https://www.kaggle.com/code/iafoss/rna-starter). This code processes RNA sequences, applies a transformer model, and generates predictions for RNA reactivity.

## Table of Contents

- [Installation](#installation)
- [Usage](#usage)
  - [Data Preparation](#data-preparation)
  - [Model Definition](#model-definition)
  - [Inference](#inference)
- [Results](#results)
- [Contributing](#contributing)
- [License](#license)

## Installation

Ensure you have the following dependencies installed:

```bash
pip install pandas numpy tqdm scikit-learn torch
```

Additionally, ensure you have the datasets and models:

- Dataset: Stanford Ribonanza RNA Folding
- Model weights: example0_0.pth

## Usage

### Data Preparation

The dataset is processed using a custom PyTorch `Dataset` class. This class handles the RNA sequences, applies padding, and generates masks.

```python
import pandas as pd
import torch
from torch.utils.data import Dataset

class RNA_Dataset_Test(Dataset):
    def __init__(self, df, mask_only=False):
        self.seq_map = {'A':0,'C':1,'G':2,'U':3}
        df['L'] = df.sequence.apply(len)
        self.Lmax = df['L'].max()
        self.df = df
        self.mask_only = mask_only
    
    def __len__(self):
        return len(self.df)
    
    def __getitem__(self, idx):
        id_min, id_max, seq = self.df.loc[idx, ['id_min','id_max','sequence']]
        mask = torch.zeros(self.Lmax, dtype=torch.bool)
        L = len(seq)
        mask[:L] = True
        if self.mask_only: return {'mask':mask},{}
        ids = np.arange(id_min,id_max+1)
        
        seq = [self.seq_map[s] for s in seq]
        seq = np.array(seq)
        seq = np.pad(seq,(0,self.Lmax-L))
        ids = np.pad(ids,(0,self.Lmax-L), constant_values=-1)
        
        return {'seq':torch.from_numpy(seq), 'mask':mask}, {'ids':ids}
```

### Model Definition

The model is a transformer-based neural network with sinusoidal positional encoding. It includes embedding, positional encoding, transformer encoder layers, and a final projection layer.

```python
import torch.nn as nn
import math

class SinusoidalPosEmb(nn.Module):
    def __init__(self, dim=16, M=10000):
        super().__init__()
        self.dim = dim
        self.M = M

    def forward(self, x):
        device = x.device
        half_dim = self.dim // 2
        emb = math.log(self.M) / half_dim
        emb = torch.exp(torch.arange(half_dim, device=device) * (-emb))
        emb = x[...,None] * emb[None,...]
        emb = torch.cat((emb.sin(), emb.cos()), dim=-1)
        return emb

class RNA_Model(nn.Module):
    def __init__(self, dim=192, depth=12, head_size=32):
        super().__init__()
        self.emb = nn.Embedding(4, dim)
        self.pos_enc = SinusoidalPosEmb(dim)
        self.transformer = nn.TransformerEncoder(
            nn.TransformerEncoderLayer(d_model=dim, nhead=dim//head_size, dim_feedforward=4*dim,
                dropout=0.1, activation=nn.GELU(), batch_first=True, norm_first=True), depth)
        self.proj_out = nn.Linear(dim, 2)
    
    def forward(self, x0):
        mask = x0['mask']
        Lmax = mask.sum(-1).max()
        mask = mask[:, :Lmax]
        x = x0['seq'][:, :Lmax]
        
        pos = torch.arange(Lmax, device=x.device).unsqueeze(0)
        pos = self.pos_enc(pos)
        x = self.emb(x)
        x = x + pos
        
        x = self.transformer(x, src_key_padding_mask=~mask)
        x = self.proj_out(x)
        
        return x
```

### Inference

Load the test data, initialize the model, and run inference to generate predictions.

```python
import gc
from tqdm.notebook import tqdm

# Load test data
df_test = pd.read_parquet('path/to/test_sequences.parquet')
ds = RNA_Dataset_Test(df_test)
dl = torch.utils.data.DataLoader(ds, batch_size=256, shuffle=False, drop_last=False, num_workers=2)
del df_test
gc.collect()

# Load model
device = 'cuda' if torch.cuda.is_available() else 'cpu'
model = RNA_Model().to(device)
model.load_state_dict(torch.load('path/to/model.pth', map_location=device))
model.eval()

# Inference
ids, preds = [], []
for x, y in tqdm(dl):
    with torch.no_grad():
        p = model(x)
        p = p.cpu().numpy()
    
    for idx, mask, pi in zip(y['ids'].cpu(), x['mask'].cpu(), p):
        ids.append(idx[mask])
        preds.append(pi[mask[:pi.shape[0]]])

ids = torch.concat(ids)
preds = torch.concat(preds)

# Save results
df = pd.DataFrame({'id':ids.numpy(), 'reactivity_DMS_MaP':preds[:,1], 'reactivity_2A3_MaP':preds[:,0]})
df.to_csv('submission.csv', index=False, float_format='%.4f')
```

## Results

The generated predictions are saved in `submission.csv`, containing RNA reactivity values for two different methods: `reactivity_DMS_MaP` and `reactivity_2A3_MaP`.

## Contributing

Contributions are welcome! Please feel free to submit a pull request or open an issue.

## License

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
