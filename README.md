# A Tutorial on HATS for Stock Price Prediction

**By: Gabriel Pereira Pinto (@GabrielJardimPP) and Tim Ruel (@TimRuel)**

This README is intended as an explanation of the HATS paper by authors Raehyun Kim, Chan Ho So, Minbyul Jeong, Sanghoon Lee, Jinkyu Kim, and Jaewoo Kang. All code and intellectual property is credited to these authors.

## Link to Presentation Video

https://www.youtube.com/watch?v=98iH7jBehHE

## Tutorial Steps
1. Understanding HATS
2. Creating a Python Environment with the Correct Packages
3. Creating the Stock Dataset
4. Creating the Model's Layers
5. Training the HATS Model
6. Results

## Understanding HATS

[HATS](https://github.com/dmis-lab/hats), or **H**ierarchical **At**tention Network for **S**tock data, is a graph neural network (GNN) model that was developed by members of the Data Mining & Information Systems Laboratory at Korea University. The primary purpose of the model is to use the history of many companies' stock prices in combination with their various interconnected relationships in order to improve performance on stock price and index prediction tasks. 

The creators of HATS gathered historical data from 423 stocks in the S&P 500, obtaining for each stock the closing price, the volume traded for, and a 'technical' price based on book value. A total of 85 relationships between 423 companies were generated by parsing [WikiData](https://www.wikidata.org/wiki/Wikidata:Main_Page).

At a high level, their trained GNN receives the historical stock information and the relationships between companies and outputs one of the following for a given stock:

* A prediction for the state of a particular stock price at a given time in the future. The possible states are `up`, `down`, and `neutral`. This is the node classification task.

*  A prediction for the state of an aggregation of stock prices, i.e. a market index like the S&P 500. The possible states are `up`, `down`, and `neutral`. This is the graph classification task.

In this README, we will focus on the node classification task. First, let us conceptualize each of the building blocks of HATS. In the image below, taken from the original HATS paper, we see a high-level representation of the network's architecture:

![High_level_img](/figs/general_framework.png)

The authors first generated a latent representation of the historical data for the group of stocks using an encoder. They then incorporated this latent representation into the relationship data, assigning this new feature to each node of the relationship graph. These nodes represent individual companies. Finally, using methods of GNNS to generate an updated representation of the whole graph, they are able to accomplish both node and graph classification tasks.

### Encoders

In deep learning theory, an encoder generally means a type of neural network that takes inputs and returns a transformed, simpler version of them in a latent dimension. In the paper, they refer to this module as a *feature extraction module*.

In HATS, the encoder of choice is a Long Short-Term Memory (LSTM) model with a dropout layer. For a particular batch, each of which corresponds to a particular day  in a 250-day training window, we pass to the network a 50-day window feature time series for each stock. The information is then encoded as a vector of latent features for each stock and each time step (day).

The assumption is that this new representation of the historical data is able to combine the information of the market in such a way that it summarizes the evolution of each stock feature over the 50-day sliding window.

### Graph Neural Networks
A graph <!-- $G$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=G"> is a set of vertices<!-- $V$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=V"> (also called nodes) connected by a set of edges <!-- $E$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=E">. We denote this as <!-- $G=(V,E)$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=G=(V,E)">. A graph neural network (GNN) seeks to learn meaningful relationships within a graph. 

Node classification is a common application for them, and the first use of GNN in HATS is to generate a embedded representation of the graphs for each different relationship type.

In the HATS architecture, the encoding resulting from the application of the feature extraction model to the historical is computed for each stock. For each pair of neighboring nodes *i* and *j* and for each relationship type *r*, the corresponding encoded feature vectors of *i* and *j* are concatenated with the embedding vector of *r*'s graph and the one-hot-encoded vector corresponding to the neighbors of *i* and *j* into a vector <!-- $x_{ij}^r$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=x_{ij}^r">.

![Lower_Level_img](/figs/HATS.png)

This vector is then used to calculate an attention score <!-- \alpha_{ij}^r --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=\alpha_{ij}^r"> , which weights <!-- $x_{ij}^r$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=x_{ij}^r"> by taking the *Softmax* function of a linear combination of the <!-- $x_{ij}^r$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=x_{ij}^r">. This attention score serves as the weight of the original encoded feature vectors <!-- e_j --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=e_j">  in order to generate a vector representation of relation *r* for company *i* by the transformation <!-- s_i^r = \sum_{j\in V_i^r} \alpha_{ij}^r e_j. --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=s_i^r = \sum_{j\in V_i^r} \alpha_{ij}^r e_j">.

 Here, <!-- s_i^r = V_i^r --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=s_i^r = V_i^r"> represents the set of neighboring nodes of *j* for relationship *r*.

Each <!-- s_i^r --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=s_i^r"> is then concatenated with <!-- e_i --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=e_i">  and the relation embedding to produce <!-- \tilde{x}^r_i --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=\tilde{x}^r_i">. This vector will be used as the input to a *relation attention layer*, which generates weights <!-- \tilde{\alpha}_{i}^r--> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=\tilde{\alpha}_{i}^r"> to aggregate the representations of relations <!-- s_i^r--> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=s_i^r"> by constructing the vector<!-- \tilde{e}_i = \sum_r \tilde{\alpha}_{i}^r s_i^r--> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=\tilde{e}_i = \sum_r \tilde{\alpha}_{i}^r s_i^r">.

Finally, the representation of each node is defined as <!-- \bar{e}_i = e_i + \tilde{e}_i--> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=\bar{e}_i=e_i%2B\tilde{e}_i">. The <!-- \bar{e}_i--> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=\bar{e}_i"> are the vectors that will ultimately  be passed to the next layer in order to make the classifications for the state of a node or graph.

The authors applied a fully connected layer and outputted its *Softmax*, performing gradient descent with a cross-entropy loss function to update the weights.

## Creating a Python Environment with the Correct Packages

A Python environment containing all the required packages to run HATS can be created in Anaconda with the commands:

```conda
conda create --name <envname> python=3.6.13
conda activate <envname>
conda install numpy=1.15.1 tensorflow=1.11.0 pandas scikit-learn
```

## Creating the Stock Dataset

The first step in implementing HATS is to download the dataset. In the paper's GitHub repository, the authors provide a bash script that downloads both the stocks and relationships between companies datasets. 

The stock data is comprised of historical data from 431 companies in the S&P 500, contemplating the periods between 2013/02/08 and 2019/06/17 (1174 trading days in total). 

The relationship data is gathered from [WikiData](https://www.wikidata.org/wiki/Wikidata:Main_Page), and describes the relationship between companies in terms of 85 different types of relations. Since there are few edges (relations) between each company (nodes), the authors augmented the dataset by considering two companies connected if there is an indirect path with a maximum length of 2 between the two companies. The authors provide a table describing the different types of relations.  

After cloning the HATS repository, you can check that inside the `/Node Classification/data` folder there is a file called `download.sh`. Using the following command on a terminal will download a compressed version of the dataset:

```bash
bash download.sh
```

After decompressing the file, you can use the code the authors provided to get pre-formatted batches of the data. Let's take an in-depth look at the `config.py` file located in `/Node Classification/src`.

In this file, the authors used the functionality of the `argparse` class in order to make their `main.py` file, which executes the actual training and testing of HATS, executable on the command line. In order to do so, it is necessary to pass these arguments to your `main.py` call:

```bash
make test_phase=1 save_dir=save
```

Notice that since we're focusing on the node classification task, we don't have to worry about the `data_type='S5CONS'` argument.

If you want to change any of the hyperparameters of the model, you can pass them to the function call as follows:
```bash
make test_phase=1 save_dir=save argument_1=val_arg_1
```

If you prefer to use their code interactively, however, another option is to add an empty string (or a string with the correct arguments call) to the output of the `get_args` function:

```python
# Taken from /Node Classification/src/config.py

import os
import argparse

def get_args():
    argp = argparse.ArgumentParser(description='Stock Movement Prediction',
                                     formatter_class=argparse.ArgumentDefaultsHelpFormatter)

    # Add arguments to the the argument class e.g.:
    argp.add_argument('--arg1', type=int, default= 0)
    argp.add_argument('--arg2', type=str, default="string")

    # and add an empty string to the return argument
    return argp.parse_args("")

```
If you use the empty string, you will obtain the full list of default arguments that are passed to `get_args` as  when you call `get_args`.

From there, you can access batches/ relations from their dataset by using the following lines of code (you might have to change your working directory):

```python
# Taken from main.py
import os, time, argparse
import numpy as np
import pandas as pd

from config import get_args
from dataset import StockDataset

os.chdir(my_wd)

config = get_args()
dataset = StockDataset(config)

# Make a batch to try code out 
# arguments mean you take from the train_dataset, 
# lookback is the same as in config.lookback (size of sliding window)

# stocks, classification of returns, returns
x, y, rt = next(dataset.get_batch("train", config.lookback)) 

# To get the data on the matrix of relationship between companies:
mat_rel = dataset.rel_mat
```

Here, each `x` is a list of length 249, one for each day in the training phase. Each element of that list is an `np.ndarray` with `x[n].shape = (n_stocks,lookback,n_features)`, where:

* `n_stocks` is the number of stocks of the S&P500 for each trading day (their default is 423)
* `lookback` is the size of the sliding window. You can control this by passing a different `--lookback` argument (the default value is 50).
* `n_features` is the number of features considered for each stock. You can control which features are used by passing a different `--feature_list` argument.

As for the relation matrix `mat_rel`, it is an `np.ndarray` with `mat_rel.shape = (len(rel_list),  n_stocks,n_stocks)`, where
`rel_list` is a list of all the different relations that should be considered. You can change the list of relations by changing the default `--use_rel_list` argument. 

You can also access a `np.ndarray` containing neighbors of a given node <!-- $i$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=i"> for a relation <!-- $r$ --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=r"> by accessing the following variable in your instance of the `StockDataset` class:

```python 
dataset.neighbors[r][i]
```

Note `dataset.neighbors` is a nested list with sizes `(n_relations, n_stocks)`.

## Creating the Model's Layers

The `HATS` class is initialized as

```python
import numpy as np
import os
import random
import re
import time

from base.base_model import BaseModel
import tensorflow as tf

class HATS(BaseModel):
    def __init__(self, config):
        super(HATS, self).__init__(config)
        self.n_labels = len(config.label_proportion)
        self.input_dim = len(config.feature_list)
        self.num_layer = config.num_layer
        self.keep_prob = 1-config.dropout
        self.max_grad_norm = config.grad_max_norm
        self.num_relations = config.num_relations
        self.node_feat_size = config.node_feat_size
        self.rel_projection = config.rel_projection
        self.feat_attention = config.feat_att
        self.rel_attention = config.rel_att
        self.att_topk = config.att_topk
        self.num_companies = config.num_companies
        self.neighbors_sample = config.neighbors_sample


        self.build_model()
        self.init_saver()
```

The `BaseModel` class is a wrapper class that defines methods to save/load the checkpoint of the current session and tracks the global and epoch step counters. The authors also define methods for `HATS` below, each of which is related to the building blocks of the full model.

### Feature Extraction Layer

```python
def get_state(self, state_module):
    if state_module == 'lstm':
        cells = [tf.contrib.rnn.BasicLSTMCell(self.node_feat_size) for _ in range(1)]
        dropout = [tf.contrib.rnn.DropoutWrapper(cell, input_keep_prob=self.keep_prob,
                                    output_keep_prob=self.keep_prob) for cell in cells]
        lstm_cell = tf.nn.rnn_cell.MultiRNNCell(dropout, state_is_tuple=True)
        outputs, state = tf.nn.dynamic_rnn(lstm_cell, self.x, dtype=tf.float32)
        state = tf.concat([tf.zeros([1,state[-1][-1].shape[1]]), state[-1][-1]], 0) # zero padding

    return state
```

The feature extraction layer uses an encoder-type module to generate a meaningful representation from the historical price series at time step $t$. In HATS, the authors use an LSTM with a dropout layer, taking the hidden states as outputs to be passed to the GNN module.

### Relation Embedding and State Attention Layer

```python
def create_relation_onehot(self, ):
    one_hots = []
    for rel_idx in range(self.num_relations):
        one_hots.append(tf.one_hot([rel_idx],depth=self.num_relations))
    return tf.concat(one_hots,0)
```

`create_relation_onehot` creates a one-hot vector for each relation type and stacks them on top of one another.

```python
def to_input_shape(self, emb):
    emb_ = []
    for i in range(emb.shape[0]):
        exp = tf.tile(tf.expand_dims(tf.expand_dims(emb[i], 0),0),[self.num_companies, self.neighbors_sample,1])
        emb_.append(tf.expand_dims(exp,0))
    return tf.concat(emb_,0)
```

`to_input_shape` puts the one-hot encoding into the right shape. Notice that this method is superseded when calculating the final relation representation for each node *i*.

```python
def get_relation_rep(self, state):
    # input state [Node, Original Feat Dims]
    with tf.variable_scope('graph_ops'):
        if self.feat_attention:
            neighbors = tf.nn.embedding_lookup(state, self.rel_mat)
            # exp_state [1, Nodes, 1, Feat dims]
            exp_state = tf.expand_dims(tf.expand_dims(state[1:], 1), 0)
            exp_state = tf.tile(exp_state, [self.num_relations, 1, self.max_k, 1])
            rel_embs = self.to_input_shape(self.rel_emb)

            # Concatenated (Neighbors with state) :  [Num Relations, Nodes, Num Max Neighbors, 2*Feat Dims]
            att_x = tf.concat([neighbors, exp_state, rel_embs], -1)

            score = tf.layers.dense(inputs=att_x, units=1, name='state_attention')
            att_mask_mat = tf.to_float(tf.expand_dims(tf.sequence_mask(self.rel_num, self.max_k), -1))
            att_score = tf.nn.softmax(score, 2)
            all_rel_rep = tf.reduce_sum(neighbors*att_score, 2) / tf.expand_dims((tf.to_float(self.rel_num)+1e-10), -1)

    return all_rel_rep
```

For each relation type and for each pair of connected nodes within that relation, `get_relation_rep` concatenates the following:
* the features outputted by the feature extraction module
* the relation embeddings
* the one-hot encoded set of neighbors of each node *i* for relation *r*

Notice, however, that the number of related nodes to any company *i* never exceeds the value of `max_k`. The method then generate the attention scores of each pair of connected nodes for a given relation. Finally, for each feature of a given node, we calculate the vector representation for the given relation as <!-- s_i^r = \sum_{j\in V_i^r} \alpha_{ij}^r e_j --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=s_i^r = \sum_{j\in V_i^r} \alpha_{ij}^r e_j"> for a relation *r*.


where <!-- s_i^r --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=s_i^r"> is the summarized relation information vector, <!-- e_j --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=e_j"> is the representation of the current state of company *j*, and <!-- \alpha_{ij}^r --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=\alpha_{ij}^r"> is the attention score of each pair of connected nodes for relation type *r*.

### Relation Attention Layer

```python
def aggregate_relation_reps(self,):
    def to_input_shape(emb):
        # [R,N,K,D]
        emb_ = []
        for i in range(emb.shape[0]):
            exp = tf.tile(tf.expand_dims(emb[i], 0),[self.num_companies,1])
            emb_.append(tf.expand_dims(exp,0))
        return tf.concat(emb_,0)
    with tf.name_scope('aggregate_ops'):
        # all_rel_rep : [Num Relations, Nodes, Feat dims]
        if self.rel_attention:
            rel_emb = to_input_shape(self.rel_emb)
            att_x = tf.concat([self.all_rel_rep,rel_emb],-1)
            att_score = tf.nn.softmax(tf.layers.dense(inputs=att_x, units=1,
                                    name='relation_attention'), 1)
            updated_state = tf.reduce_mean(self.all_rel_rep * att_score, 0)
        else:
            updated_state = tf.reduce_mean(self.all_rel_rep, 0)
    return updated_state
```

`aggregate_relation_reps` first redefines the input shape for an embedding vector. Then:
* `att_x` is a concatenation of the vector representations <!-- $s_i^{r} --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=s_i^{r}"> with <!-- $e_i --> <img style="transform: translateY(0.1em); background: white;" src="https://render.githubusercontent.com/render/math?math=e_i"> (the representation of the current state of company *i*) and the embedding vector for relation *r*.
* `att_score` is calculated from this concatenation.
* `updated_state` calculates the mean of the weighted attention scores across the different relations for node *i*.

If `self.rel.attention == FALSE`, then `updated_score` is calculated directly from the concatenation in `get_relation_rep`.
### Outputting the Results and Calculating Loss

```python
def build_model(self):
    # Initializes dropout probability for the dropout layer
    self.keep_prob = tf.placeholder_with_default(1.0, shape=())
    # Initializes all the input variables/ global vars related to the inputs
    # x [num company, lookback]
    self.x = tf.placeholder(tf.float32, shape=[None, self.config.lookback, self.input_dim])
    self.y = tf.placeholder(tf.float32, shape=[None, self.n_labels])
    self.rel_mat = tf.placeholder(tf.int32, shape=[None, None, self.neighbors_sample]) # Edge, Node, Node
    self.rel_num = tf.placeholder(tf.int32, shape=[None, None]) # Edge, Node
    self.max_k = tf.placeholder(tf.int32, shape=())

    self.rel_emb = self.create_relation_onehot() # one-hot encoding for the relations

    state = self.get_state('lstm') # finds state representation for nodes at a certain time
    
    # Graph operation
    # node attention step -> vector representation of relation r for company i
    self.all_rel_rep = self.get_relation_rep(state) 

    # [Node, Feat dims]
    # relation attention step -> update vector representation of for company i considering all relations
    rel_summary = self.aggregate_relation_reps()
    updated_state = rel_summary+state[1:] # updated state

    # Output layer -> prediction in one of the classes
    logits = tf.layers.dense(inputs=updated_state, units=self.n_labels,
                            activation=tf.nn.leaky_relu, name='prediction')

    self.prob = tf.nn.softmax(logits)
    self.prediction = tf.argmax(logits, -1)
    
    # Calculates the loss related to the output, and updates the weights
    with tf.name_scope("loss"):
        self.cross_entropy = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(labels=self.y, logits=logits))
        reg_losses = tf.get_collection(tf.GraphKeys.REGULARIZATION_LOSSES)
        loss = self.cross_entropy
        update_ops = tf.get_collection(tf.GraphKeys.UPDATE_OPS)
        with tf.control_dependencies(update_ops):
            self.train_step = tf.train.AdamOptimizer(self.config.lr).minimize(loss, global_step=self.global_step_tensor)
        correct_prediction = tf.equal(tf.argmax(logits, -1), tf.argmax(self.y, -1))
        self.accuracy = tf.reduce_mean(tf.cast(correct_prediction, tf.float32))
```
This method adds placeholders for the input variables and combines all the layers described above with an appropriate loss function to generate the computational graph for HATS.

We've commented the code to emphasize what each method call does. After the specification of the model, we can go ahead and train it.

## Training the HATS Model

Training the HATS model requires the creation of two Python classes, `BaseTrain` and `Trainer`.

### BaseTrain

`BaseTrain` is initialized with the following arguments:
* `sess` -  A TensorFlow session.
* `model` - An object of class `HATS`.
* `data` - An object of class `StockDataset`. 
* `config` - Meant to accept the `get_args()` function in `config.py`. Defines various arguments to be passed to other functions. 
* `logger` - Meant to accept the `set_logger` function in `logger.py`. Keeps track of metadata such as file locations as the model is being trained.
* `evaluator` - An object of class `Evaluator`.

Each of these arguments is also defined as an attribute in the initialization for the `BaseTrain`:

```python
class BaseTrain:
    def __init__(self, sess, model, data, config, logger, evaluator):
        self.model = model
        self.logger = logger
        self.config = config
        self.sess = sess
        self.data = data
        self.evaluator = evaluator
        self.best_f1 = {'topk':dict(), 'all':{'macroF1':0}}
```

The attribute `best_f1` is meant for keeping track of F1-scores when testing HATS' performance against other models.

After `BaseTrainer` is initialized, there is only one method that is actually defined inside it: `train`. 

```python
def train(self):
        te_loss_hist, te_acc_hist, te_acc_k_hist = [], [], []
        prev = -1
        for cur_epoch in range(self.model.cur_epoch_tensor.eval(self.sess), self.config.n_epochs + 1, 1):
            loss, report_all, report_topk = self.train_epoch()
            self.sess.run(self.model.increment_cur_epoch_tensor)
            .
            .
            .
```

The rest of the method adds result to the logs and does the evaluations at the evaluation set, printing the results for appropriate epochs.

### Trainer

The class `Trainer` inherits its attributes from `BaseTrain`:

```python
class Trainer(BaseTrain):
    def __init__(self, sess, model, data, config, logger, evaluator):
        super(Trainer, self).__init__(sess, model, data, config, logger, evaluator)
        self.keep_prob = 1-config.dropout
```

It has the following methods:
* `sample_neighbors`
* `train_epoch`
* `get_rel_multi_hot` - UNUSED
* `create_feed_dict`
* `train_step`

#### sample_neighbors

 `sample_neighbors` is defined as 

```python
def sample_neighbors(self):
    k = self.config.neighbors_sample
    if self.config.model_type == 'HATS':
        neighbors_batch = []
        for rel_neighbors in self.data.neighbors:
            rel_neighbors_batch = []
            for cpn_idx, neighbors in enumerate(rel_neighbors):
                short = max(0, k-neighbors.shape[0])
                if short: # less neighbors than k
                    neighbors = np.expand_dims(np.concatenate([neighbors, np.zeros(short)]),0)
                    rel_neighbors_batch.append(neighbors)
                else:
                    neighbors = np.expand_dims(np.random.choice(neighbors, k),0)
                    rel_neighbors_batch.append(neighbors)
            neighbors_batch.append(np.expand_dims(np.concatenate(rel_neighbors_batch,0),0))
```

For each node (company) in the dataset,`sample_neighbors` takes a random sample of size *k* (the default is set to 20 in `config.py`) from the node's set of neighbors defined by a given relation type. The method uses two `for` loops to cycle through all of the relation types and nodes in the dataset. Each sample is stored as a 1 by *k* array, where each element in the array corresponds to the index of a neighbor that was selected to be in the sample. If *k* is greater than the total number of neighbors a node has, then all of the neighbors are selected into the sample, and then zeros are appended to the array until its column dimension is equal to *k*. 

#### train_epoch

`train_epoch` is defined as

```python
def train_epoch(self):
    all_x, all_y, all_rt = next(self.data.get_batch('train', self.config.lookback))
    neighbors = self.sample_neighbors()
    labels = []
    losses, accs, cpt_accs, pred_rates, mac_f1, mic_f1, exp_rts = [], [], [], [], [], [], []
    accs_k, cpt_accs_k, pred_rates_k, mac_f1_k, mic_f1_k, exp_rts_k = [], [], [], [], [], []

    for x, y, rt in zip(all_x, all_y, all_rt):
        loss, metrics = self.train_step(x, y, rt, neighbors)

        metrics_all, metrics_topk = metrics
        losses.append(loss)
        pred_rates.append(metrics_all[0])
        accs.append(metrics_all[1][0])
        cpt_accs.append(metrics_all[1][1])
        mac_f1.append(metrics_all[2])
        mic_f1.append(metrics_all[3])
        exp_rts.append(metrics_all[4])
        pred_rates_k.append(metrics_topk[0])
        accs_k.append(metrics_topk[1][0])
        cpt_accs_k.append(metrics_topk[1][1])
        mac_f1_k.append(metrics_topk[2])
        mic_f1_k.append(metrics_topk[3])
        exp_rts_k.append(metrics_topk[4])

    report_all = [np.around(np.array(pred_rates).mean(0),decimals=4), np.mean(accs), np.mean(cpt_accs), np.mean(mac_f1), np.mean(mic_f1), np.mean(exp_rts)]
    report_topk = [np.around(np.array(pred_rates_k).mean(0),decimals=4), np.mean(accs_k), np.mean(cpt_accs_k), np.mean(mac_f1_k), np.mean(mic_f1_k), np.mean(exp_rts_k)]
    loss = np.mean(losses)
    cur_it = self.model.global_step_tensor.eval(self.sess)
    return loss, report_all, report_topk
```

This method takes a batch from the dataset, supplies it to the `train_step` method, and then calculates the loss and various performance metrics on the training and evaluation datasets.

#### train_step

`train_step` is defined as

```python 
def train_step(self, x, y, rt, neighbors):
    # batch is whole dataset of a single company
    feed_dict = self.create_feed_dict(x, y, neighbors)
    _, loss, pred, prob = self.sess.run([self.model.train_step, self.model.cross_entropy,
                                self.model.prediction, self.model.prob],
                                    feed_dict=feed_dict)
    label = np.argmax(y, 1)
    metrics = self.evaluator.metric(label, pred, prob, rt)

    return loss, metrics
```

This method performs gradient descent with a cross entropy loss function, updates the weights, and outputs the loss and performance metrics.

`create_feed_dict` creates the feeding dictionary to be passed to the Tensorflow session.


## Results

As previously mentioned, the authors classified each closing price into an `up` (moderate to large upward movement), `neutral` (small movement in either direction) or `down` (moderate to large downward movement) class.

In order to evaluate the results of the network, they divided the historical dataset into overlapping regions of eight smaller phases of 400 days each:  250 days for training, 50 for evaluation, and 100 for testing. The results were evaluated based on the profitability of a trading strategy informed by HATS, in which a trader would buy the 15 stocks most likely to be in the `up` class and short the 15 stocks most likely to be in the `down` class.

The success of the trading strategy above is measured using the return and Sharpe ratio of the portfolios. The authors also measure the effectiveness of the algorithm in terms of accuracy/ F1 score.

![Results](/figs/Results.png)

Their key findings can be summarized below:

* HATS outperformed all the models against which it was compared.
* The choice of relation type(s) can have a drastic effect on the prediction performance of the model.
* Using relational data does not always yield good results in stock market prediction.
* There is usually noise in fully connected networks.
* It is difficult to select meaningful relationship between companies and there is much room for improvement.

HATS was able to produce a 9.61% average daily return which represents a significant improvement over the LSTM baseline of a 4.32% average daily return. HATS also achieved a 1.9914 annualized Sharpe ratio, which is considerably better than the LSTM baseline of 1.1523.

The authors analyzed the attentention scores of the relations, in order to understand which types of links are important for the prediction task. 

![Relations](/figs/Relations.png)

Notice that the relations with the highest scores are the ones that represent dominant-subordinate relationships, but some high scores relations represent industrial dependencies. Most relations with low scores represent location links.

