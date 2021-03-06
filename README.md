# Deep Networks with Stochastic Depth

This project is an implementation of the Stochastic Depth method for training neural Networks, as specified in the research paper
here: https://arxiv.org/abs/1603.09382. 

This implementation allows for usage of all standard caffe solvers, as well as stochastic depth, from the command line and c++ interface. It currently has a working example of a 54 resblock convolutional neural network. This network is identical to the 54 bock net specified in the stochastic depth paper.

I am a student affiliated with Killian Weinberger's research group at Cornell (the authors of the paper), but am not myself an author.


### Example Network Results
The example network was trained with both stochastic depth and regular depth, each on a Titan X GPU. Stochastic depth outperformed regular depth by 0.9%. Regular depth reached a max accuracy of 94.2187, while stochastic depth reached 95.1016. The time saving, however, weren't quite as good as those outlined in the paper. The net has a liner survival rate decay from 1 to 0.5, so in theory stochastic depth should train in 25% less time than regular depth. The example network only saw a 18.3% reduction in training time. This could be due to inefficiencies in the implementation and/or suboptimal memory management on the machine.

Here are links to graphs of the results:
- [Test Accuracy vs Iterations](results/graphs/test_accuracy.png)
- [Test Accuracy vs Time](results/graphs/test_accuracy_vstime.png)
- [Test Loss vs Iterations](results/graphs/test_loss.png)
- [Train Loss vs Iterations](results/graphs/train_loss.png)

# Getting Started

Follow the standard caffe installation procedure specified here: http://caffe.berkeleyvision.org/installation.html. 

#### Preprocessing the cifar10 Dataset

To reproduce the results in the paper, proper preprocessing is needed, including subtracting means, dividing by std, and padding 0s for data augmentation. 

First, get the cifar10 data in LEVELDB format. Run
`./data/cifar/get_cifar10/sh`
`./examples/cifar10/create_cifar10.sh`

Then, run the preprocessing script
`python examples/cifar10/preprocessing.py`

####Training

The solver and net prototxts are in the folder `examples/stochastic_depth`. The solver is `solver54.prototxt`, and the nets are in `residual_train54.prototxt` and `residual_test54.prototxt`. Remember to change the location of the ciphar10 database in the data layers of both net prototxt files to point to your cifar10 installation.

To train the example network using stochastic depth, run this command from your caffe root: 
- `./build/tools/caffe train_stochdep --solver=examples/stochastic_depth/solver54.prototxt` 

To train the example network without stochastic depth (like a normal caffe net), run this command from your caffe root: 
- `./build/tools/caffe train --solver=examples/stochastic_depth/solver54.prototxt` 

#### The Networks and Solvers
We generated the prototxt (network and solver) files needed for reproducing the cifar10 results. We also provided the scripts to generate networks of different architectures, so you can generate and run stochastic depth networks of any depth and width. If you're interested, take a look at 
`/examples/stochastic-depth/make_net.py`


# Implementation Details

In the current implementation, there are two c++ functions that must be replaced in order to train a different network with stochastic depth. These are:
- `void Net<Dtype>::ChooseLayers_StochDep()`
- `void Net<Dtype>::InitTestScalingStochdept()`

Their functionality is pretty simple, but implementing them can be a bit rough.

### `void Net<Dtype>::ChooseLayers_StochDep()`

This function is called by the solver in the function `void Solver<Dtype>::Step(int iters)` during every training iteration. It's job is to initialize a few data structures: 

- `vector<int> layers_chosen`  
- `vector<vector<Blob<Dtype>*> > bottom_vecs_stochdept_;`
- `vector<vector<Blob<Dtype>*> > top_vecs_stochdept_;`

##### `vector<int> layers_chosen` 
Internally, caffe stores it's layers in a vector, and it loops through this vector when doing forward and backward passes, calling each layer's individual forward and backward function. This vector is called ` vector<shared_ptr<Layer<Dtype> > > layers_`. `layers_chosen` must contain the indexes in `layers_` of all the layers that are *not* getting dropped in the current iteration. That is, all the surviving layers. The indeces must be ordered from least to greatest.

##### `vector<vector<Blob<Dtype>*> > bottom_vecs_stochdept_;`
This vector stores pointers to bottom blobs of the layers specified in  `layers_chosen`. Each vector of blobs in `bottom_vecs_stochdept_` correspond to the inputs to a layer in `layers_chosen`. 

##### `vector<vector<Blob<Dtype>*> > top_vecs_stochdept_;`
Similarly, each vector of blobs in this vector where the corresponding layer specified in `layers chosen` will store it's output. `top_vecs_stochdep

It follows that  `top_vecs_stochdep_[i] == bottom_vecs_stochdep_[i+1]`

You can take pointers from the vectors `top_vecs` and `bottom_vecs` to initilize the above data structures.



### `void Net<Dtype>::InitTestScalingStochdept()`
This function is called once in net instantiation, and is used to initialize the vector 
- `vector<double> test_scaling_stochdept_;`. 

When testing a stochastic depth network with all layers included, you must multiply the output of each resblock with it's survival rate. `test_scaling_stochdept_` is a vector specifying what to scale the output of each layer in `layers_chosen` by during testing, corresponding by index with `layers_chosen`. In my example network, most of the values in `test_scaling_stochdept_` are 1, whereas the output of the last layer in each resblock that gets dropped gets scaled by the suvival rate of that resblock (a linear function from 1 to 0.5).

Once these functions are set up you're ready to train your network.

# Future Work
I'm aware that this is not the most user friendly implementation. I have an idea about how to extend this implementation to work with the python interface to allow for quick and easy construction of stochastic depth networks. I will begin work on this soon. 

Feel free to reach out to me if you have any questions for me (or if you want to help with the python interface), I'm happy to help! 
