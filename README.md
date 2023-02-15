# EigenPro3: Fast algorithm to train large general kernel models

*General kernel models* are of the form
$$f(x)=\sum_{i=1}^p \alpha_i K(x,z_i)$$
where $z_i$ are $p$ model centers. The model centers can be arbitrary, i.e., do not need to be a subset of the training data. EigenPro3 requires only $O(p)$ memory, and takes advantage of multiple GPUs.

The EigenPro3 algorithm is based on Projected dual-preconditioned Stochastic Gradient Descent. If fully decouples the model and training
A complete derivation for the training algorithm is given in the following paper  
**Title:** [Toward Large Kernel Models](https://arxiv.org/abs/2302.02605) (2023)  
**Authors:** Amirhesam Abedsoltan, Mikhail Belkin, Parthe Pandit.  
Recent studies indicate that kernel machines can often perform similarly  or better than deep neural networks (DNNs) on small datasets. The interest in kernel machines has been additionally bolstered by the discovery of their equivalence to wide neural networks in certain regimes. 
However, a key feature of DNNs is their ability to scale the model size and training data size independently, whereas in traditional kernel machines model size is tied to data size. Because of this coupling, scaling kernel machines to large data has been computationally challenging. 
We introduce EigenPro 3.0, an algorithm that provides a way forward  for constructing large-scale *general kernel models*. 

Read on for more details about the algorithm.

# Installation
```
pip install git+https://github.com/EigenPro/EigenPro3.git@testing
```
Tested on:
- pytorch >= 1.13 (not installed along with this package)

## Demo on CIFAR-10 dataset
Set an environment variable `DATA_DIR='/path/to/dataset/'` where the file `cifar-10-batches-py` can be found. If you would like to download the data, see instructions below the following code-snippet.
```python
import torch
from eigenpro3.utils import accuracy, load_dataset
from eigenpro3.datasets import CustomDataset
from eigenpro3.models import KernelModel
from eigenpro3.kernels import laplacian, ntk_relu

p = 5000 # model size

if torch.cuda.is_available():
    DEVICES = [torch.device(f'cuda:{i}') for i in range(torch.cuda.device_count())]
else:
    DEVICES = [torch.device('cpu')]

kernel_fn = lambda x, z: laplacian(x, z, bandwidth=20.0)
# kernel_fn = lambda x, z: ntk_relu(x, z, depth=2)

n_classes, (X_train, y_train), (X_test, y_test) = load_dataset('cifar10')

centers = X_train[torch.randperm(X_train.shape[0])[:p]]

testloader = torch.utils.data.DataLoader(
    CustomDataset(X_test, y_test.argmax(-1)), batch_size=512,
    shuffle=False, num_workers=16, pin_memory=True)

model = KernelModel(n_classes, centers, kernel_fn, X=X_train,y=y_train,devices = DEVICE_LIST)
model.fit(model.train_loaders, testloader, score_fn=accuracy, epochs=20)
```
### Downloading Data
```python
from torchvision.datasets import CIFAR10
import os
CIFAR10(os.environ['DATA_DIR'], train=True, download=True)
```
## Custom dataloader tutorial on FashionMNIST
Refer to [Custom Dataloader](https://github.com/EigenPro/EigenPro3/blob/testing/demos/Custom_dataloader.ipynb) tutorial to use your own dataloader.
# Limitations of EigenPro 2.0
EigenPro 2.0 can only train models of the form $$f(x)=\sum_{i=1}^n \alpha_i K(x,x_i)$$ where $x_i$ are $n$ training samples.

## Algorithm details
**EigenPro 3.0** solves the optimization problem,
$$\underset{\text{argmin}}{f\in\mathcal{H}}\quad \sum_{i=1}^n (f(x_i)-y_i)^2 \quad \text{s.t.}\quad f\in \mathcal{C}=\{f\mid f(x)=\sum_{i=1}^p \alpha_i K(x,z_i)\}$$
    
**EigenPro 3.0** applies a dual preconditioner, one for the model and one for the data. It applies a projected-preconditioned SGD
$$f^{t+1}=\mathrm{proj}_{C}(f^t - \eta\mathcal{P}(\nabla L(f^t)))$$
where $\nabla L$ is a Fréchet derivative, $\mathcal{P}$ is a preconditioner, and $\textrm{proj}_{\mathcal{C}}$ is a projection operator onto the model space.
