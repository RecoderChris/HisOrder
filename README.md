# Hisorder
Frontier-guided Distribution Graph Reordering

## What is it?
HisOrder is a graph reordering method, which improves graph locality to reduce the cache misses in graph processing. 

Unlike the previous reordering which depends on static characteristics in graph, HisOrder firstly profiles the traces of **his**torical graph processing (primarily the concurrent frontiers) to construct the locality metric between vertices, and then utilizes an unsupervised ML method (K-means at present) to excavate the clusters of high locality to guide graph **reorder**ing. Furthermore, since the learned clusters of vertices are more likely to be co-activated, HisOrder also fine-tunes the load balance in parallel graph processing with the clusters. 

![hisorder](img/hisorder.png)

For more details, please refer to [our paper](https://liu-cheng.github.io/). 

## Getting Started
### 0. Dependencies
At the minimum, HisOrder depends on the following software:
- make>=4.1
- g++>=7.5.0 (compliant with C++11 standard and OpenMP is required)
- Boost (>=1.58.0)

A possible command to satisfy all these dependencies:
```shell
apt-get install build-essentials libboost-dev libomp-dev
```

### 1. Compilation
To compile HisOrder, run the following on root directory:
```shell
make
```

### 2. Data Preparation
HisOrder takes graph in Compressed Sparse Row (CSR) format as both the input and output. 
In this way, after downloading or generating the graph in edgelist format (i.e. `src_vertex dst_vertex` format), we should convert the data into CSR format firstly for prepration. 
#### 2.1 Download Graph or Generate Graph
First of all, create a directory to preserve all the graph data. 
```shell
ROOT_DIR=`pwd`
TOOL_DIR=$ROOT_DIR/tools
DATA_DIR=$ROOT_DIR/dataset
mkdir -p dataset && cd dataset
```
To obtain the graph data, you can download real-world graph data in edgelist format. 
For instance, to download `soc-LiveJournal` graph from [SNAP large network collection](http://snap.stanford.edu/data/index.html): 
```shell
wget http://snap.stanford.edu/data/soc-LiveJournal1.txt.gz
gzip -d soc-LiveJournal1.txt.gz
```
Or you could generate RMAT graph using [`PaRMAT`](https://github.com/farkhor/PaRMAT). To generate a RMAT graph (named `R20.el`) with 1M vertices and 16M edges using 8 threads, you can run: 
```shell
cd ${TOOL_DIR}/PaRMAT/Release
make
./PaRMAT -nEdges 16777216 -nVertices 1048576 -output R20.el -threads 8 -noDuplicateEdges
mv R20.el $DATA_DIR
```
#### 2.2 Convert Edgelist into CSR format
We provide tools to support conversion from edgelist into CSR. 
For example, to convert RMAT graph `R20.el`, you could run:
```shell
cd ${TOOL_DIR}/el2csr
make
./el2csr ${DATA_DIR}/R20.el ${DATA_DIR}/R20.csr
```

### 3. Graph Reordering
The workflow of HisOrder could be sliced into three stages: sampling, embedding and reordering. 
#### 3.1 Sampling BFS Frontiers
We execute BFS from several random starters to obtain the frontier distribution. Since we take [GPOP](https://github.com/souravpati/GPOP) as one of targeted system, we modified GPOP to sample the frontiers of BFS. 

To sample frontiers for R20 graph from 15 random BFS requests, run:
```shell
cd $ROOT_DIR/system/GPOP/bfs
make
./bfs ${DATA_DIR}/R20.csr -rounds 15 # -rounds could define the number of BFS requests
```
Then the sampled frontiers of BFS execution will be preserved at `${DATA_DIR}/sample` directory.

It is worth noticing that the frontier distribution is independent to underlying systems. GPOP is used as an example. You can modify your graph processing system to collect the BFS frontiers as you wish. 

#### 3.2 Embedding
Then we embed the sampled frontier history into feature space:
```shell
cd $TOOL_DIR
python embedding.py --data R20 --feat_dim=10 --data_dir=${DATA_DIR}
```
You can input `python embedding.py --help` to see the meaning of these parameters. 

#### 3.3 Reordering using HisOrder
Please check the recipes before executing HisOrder, a possible file tree when using R20.el as an example is shown as follows:
```
$DATA_DIR
|-- feature
|   |-- R20_dim_10.feat
|-- R20.csr
|-- R20.el
|-- sample
|   |-- R20.bfs.0
|   ...
|   |-- R20.bfs.9
```
To run HisOrder: 
```shell
./hisorder -d ${DATA_DIR}/R20.csr -o ${DATA_DIR}/R20-his.csr -r ${DATA_DIR}/R20-his.map \
-a 2 -s 128 -t 8 -f 10 -k 20 -i ${DATA_DIR}/feature/R20_dim_10.feat 
```
The parameters in HisOrder is illustrated as follows:
```
    ./hisorder
    -d [ --data ]            input data path (.csr)
    -o [ --output ]          output file path (.csr)
    -r [ --output_map]       output mapping file (vertex mapping of reordering)
    -a [ --algorithm ](=0)   reordering algorithm (Including HisOrder and some baselines)
    -t [ --thread ]  (=20)   threads (number of threads)
    -s [ --size ]  (=1024)   partition size (in KB, normally=L2 Cache/2)
    -f [ --feat ]  (=10)     feature size (*Hisorder only)
    -k [ --kv ] (=20)        K value for K-means (*HisOrder Only)
    -i [ --input_feat ]      input feature file (*HisOrder Only)

    [ --algorithm ]
    hisorder_without_balance = 0,(*Out proposed method for single-thread)
    hisorder = 1, (*Our proposed method, default)
    hisorder_pcpm = 2, (*Out proposed method, designed for PCPM)
    random = 3, 
    sort = 4, 
    fbc = 5, 
    hc = 6, 
    dbg = 7, 
    corder = 8, 
```

## Evaluation
After generating the reordered CSR graph, we can test the graph processing performance under different CSR format. 
### Running Graph Algorithms
We take GPOP as example, firstly we change directory to GPOP:
```shell
cd ${ROOT_DIR}/system/GPOP
```
Taking BFS as an example:
```shell
cd BFS
```
Notice: please uncomment the `SAMPLE=1` line. And we run `make clean && make`, then we run bfs in GPOP using both HisOrder and original graph format:
```shell
./bfs ../../../dataset/R20.csr -s <start_vertex> -t <thread_num> -rounds <round_num> # original graph 
./bfs ../../../dataset/R20-his.csr -s <start_vertex> -t <thread_num> -rounds <round_num> -map ../../../dataset/R20-his.map # original graph 
```

### Benchmark Summary
We evaluate the effect of HisOrder on several representative graph processing systems, including Ligra (in-memory, vertex-centric), GPOP (in-memory, partition-centric), and GridGraph (storage-based out-of-core system). 

#### Testbed
| Item     |  Configurations                                           |
|:--------:|:---------------------------------------------------------:|
|    CPU   | 2 x Intel Xeon Platinum 8163 CPU@2.5GHz, <br>each with 24 Cores |
|  Threads |             Up to 96 threads(With hyperthread)            |
| Cache |               32KB L1d-Cache, 32KB L1i-Cache<br> 1MB L2 Cache, 32MB LLC |
|  Memory  |                      512GB DDR4 DIMM                      |
|    OS    |                     Ubunut 16.04.7 LTS                    |

#### Evaluation Results
We compared the performance of HisOrder with large number of existing graph reordering methods, including sort-by-degree, Degre-based Grouping  (**DBG**), Hub Clustering (**HC**), Frequency-based Clustering(**FBC**), Corder(**CO**), rabbit(**RBT**) and Gorder(**GO**). 
We show the average MTEPS of running BFS algorithm on 6 different graphs when using baseline (without reordering), SOTA (the best performance in existing method) and HisOrder. 

| Methods  | Ligra         | GPOP           | GridGraph     |
|----------|---------------|----------------|---------------|
| baseline | 5210 MTEPS     | 5266 MTEPS      | 272 MTEPS      |
| SOTA     | 5807 MTEPS(**CO**) | 5856 MTEPS(**RBT**) | 309 MTEPS(**DBG**) |
| HisOrder | **7149** MTEPS     | **7262** MTEPS      | **401** MTEPS      |

As we can see, HisOrder could improve the graph processing efficiency on different graph processing systems, and can beat the most effective existing reordering methods. 

Please refer to our paper to see the more details about the evaluation. 

## Future Work
1. Customize graph partition for heterogeneous computing architecture (On-going)

2. Partition neurons in large language model for fast LLM inference under resource-constraint environment (Future idea)

## Citation
If you find HisOrder is helpful to your research, please kindly cite our paper:
```

```

## Contact
[Xinmiao Zhang](mailto:zhangxinmiao20s@ict.ac.cn), [Cheng Liu](mailto:liucheng@ict.ac.cn)