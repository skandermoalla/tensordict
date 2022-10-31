# Disclaimer

TensorDict is at the alpha-stage, meaning that there may be bc-breaking changes introduced at any moment without warranty.
Hopefully that should not happen too often, as the current roadmap mostly involves adding new features and building compatibility 
with the broader pytorch ecosystem.

# TensorDict

`TensorDict` is a dictionary-like class that inherits properties from tensors, such as indexing, shape operations, casting to device etc.

The main purpose of TensorDict is to make code-bases more _readable_ and _modular_ by abstracting away tailored operations:
```python
for i, tensordict in enumerate(dataset):
    # the model reads and writes tensordicts
    tensordict = model(tensordict)
    loss = loss_module(tensordict)
    loss.backward()
    optimizer.step()
    optimizer.zero_grad()
```
With this level of abstraction, one can recycle a training loop for highly heterogeneous task.
Each individual step of the training loop (data collection and transform, model prediction, loss computation etc.)
can be tailored to the use case at hand without impacting the others. 
For instance, the above example can be easily used across classification and segmentation tasks, among many others.

## Installation

To install the latest stable version of tensordict, simply run
```bash
pip install tensordict
```
This will work with python 3.7 and upward as well as pytorch 1.12 and upward.

To enjoy the latest features, one can use
```bash
pip install tensordict-nightly
```

## Features

### General

A tensordict is primarily defined by its `batch_size` (or `shape`) and its key-value pairs:
```python
from tensordict import TensorDict
import torch
tensordict = TensorDict({
    "key 1": torch.ones(3, 4, 5),
    "key 2": torch.zeros(3, 4, 5, dtype=torch.bool),
}, batch_size=[3, 4])
```
The `batch_size` and the first dimensions of each of the tensors must be compliant.
The tensors can be of any dtype and device. Optionally, one can restrict a tensordict to
live on a dedicated device, which will send each tensor that is written there:
```python
tensordict = TensorDict({
    "key 1": torch.ones(3, 4, 5),
    "key 2": torch.zeros(3, 4, 5, dtype=torch.bool),
}, batch_size=[3, 4], device="cuda:0")
tensordict["key 3"] = torch.randn(3, 4, device="cpu")
assert tensordict["key 3"].device is torch.device("cuda:0")
```

### Tensor-like features

TensorDict objects can be indexed exactly like tensors. The resulting of indexing
a TensorDict is another TensorDict containing tensors indexed along the required dimension:
```python
tensordict = TensorDict({
    "key 1": torch.ones(3, 4, 5),
    "key 2": torch.zeros(3, 4, 5, dtype=torch.bool),
}, batch_size=[3, 4])
sub_tensordict = tensordict[..., :2]
assert sub_tensordict.shape == torch.Size([3, 2])
assert sub_tensordict["key 1"].shape == torch.Size([3, 2, 5])
```

Similarly, one can build tensordicts by stacking or concatenating single tensordicts:
```python
tensordicts = [TensorDict({
    "key 1": torch.ones(3, 4, 5),
    "key 2": torch.zeros(3, 4, 5, dtype=torch.bool),
}, batch_size=[3, 4]) for _ in range(2)]
stack_tensordict = torch.stack(tensordicts, 1)
assert stack_tensordict.shape == torch.Size([3, 2, 4])
assert stack_tensordict["key 1"].shape == torch.Size([3, 2, 4, 5])
cat_tensordict = torch.cat(tensordicts, 0)
assert cat_tensordict.shape == torch.Size([6, 4])
assert cat_tensordict["key 1"].shape == torch.Size([6, 4, 5])
```

TensorDict instances can also be reshaped, viewed, squeezed and unsqueezed:
```python
tensordict = TensorDict({
    "key 1": torch.ones(3, 4, 5),
    "key 2": torch.zeros(3, 4, 5, dtype=torch.bool),
}, batch_size=[3, 4])
print(tensordict.view(-1))  # prints torch.Size([12])
print(tensordict.reshape(-1))  # prints torch.Size([12])
print(tensordict.unsqueeze(-1))  # prints torch.Size([3, 4, 1])
```

One can also send tensordict from device to device, place them in shared memory,
clone them, update them in-place or not, split them, unbind them, expand them etc.

If a functionality is missing, it is easy to call it using `apply()` or `apply_()`:
```python
tensordict_uniform = tensordict.apply(lambda tensor: tensor.uniform_())
```

### TensorDict for functional programming using FuncTorch

We also provide an API to use TensorDict in conjunction with [FuncTorch](https://pytorch.org/functorch). 
For instance, TensorDict makes it easy to concatenate model weights to do model ensembling:
```python
from torch import nn
from tensordict import TensorDict
from copy import deepcopy
from tensordict.nn.functional_modules import FunctionalModule
import torch
from functorch import vmap
layer1 = nn.Linear(3, 4)
layer2 = nn.Linear(4, 4)
model1 = nn.Sequential(layer1, layer2)
model2 = deepcopy(model1)
# we represent the weights hierarchically
weights1 = TensorDict(model1.state_dict(), []).unflatten_keys(".")
weights2 = TensorDict(model2.state_dict(), []).unflatten_keys(".")
weights = torch.stack([weights1, weights2], 0)
fmodule, _ = FunctionalModule._create_from(model1)
# an input we'd like to pass through the model
x = torch.randn(10, 3)
y = vmap(fmodule, (0, None))(weights, x)
y.shape  # torch.Size([2, 10, 4])
```

## Nesting TensorDicts

It is possible to nest tensordict and switch easily between hierarchical and flat representations.
For instance, the following code will result in a single-level tensordict with keys `"key 1"` and `"key 2.sub-key"`:
```python
>>> tensordict = TensorDict({
...     "key 1": torch.ones(3, 4, 5),
...     "key 2": TensorDict({"sub-key": torch.randn(3, 4, 5, 6)}, [3, 4, 5])
... }, batch_size=[3, 4])
>>> tensordict = tensordict.unflatten_keys(separator=".")
```

## License
TorchRL is licensed under the MIT License. See [LICENSE](LICENSE) for details.