# Reference Implementation for whisper-large-v3

## Automated command to run the benchmark via MLFlow

Please see the [new docs site](https://docs.mlcommons.org/inference/benchmarks/language/whisper/) for an automated way to run this benchmark across different available implementations and do an end-to-end submission with or without docker.

You can also do pip install mlc-scripts and then use `mlcr` commands for downloading the model and datasets using the commands given in the later sections.

## Prepare environment

### Docker

Build the docker image
```bash
docker build -t whisper:latest .
```

Run docker image in interactive mode
```bash
docker run -it -t whisper
```

### Local Environment Preparation

- **Prerrequisite:** Install conda.
```bash
mkdir -p ~/miniconda3
wget https://repo.anaconda.com/miniconda/Miniconda3-py312_24.5.0-0-Linux-x86_64.sh -O ~/miniconda3/miniconda.sh
bash ~/miniconda3/miniconda.sh -b -u -p ~/miniconda3
rm ~/miniconda3/miniconda.sh
~/miniconda3/bin/conda init
```

- Set the following helper variables
```bash
export ROOT=$PWD/inference
export WHISPER_FOLDER=$PWD/inference/speech2text
export LOADGEN_FOLDER=$PWD/inference/loadgen
```

- Clone the inference repository:
```bash
git clone --recurse-submodules https://github.com/mlcommons/inference.git \
 --depth 1 --branch speech2text_reference
```

- Create a conda environment:
```bash
conda create -y -n whisper python=3.12
conda activate whisper
conda install -y -c conda-forge libstdcxx-ng=12
```

- Install requirements and loadgen:
```bash
pip install --break-system-packages torch==2.7.0 torchaudio==2.7.0 torchvision --index-url https://download.pytorch.org/whl/cpu && \
pip install --break-system-packages pandas==2.2.2 toml==0.10.2 unidecode==1.3.8 inflect==7.3.1 librosa==0.10.2 py-libnuma==1.2 numpy==2.0.1 && \
pip install --break-system-packages sox==1.5.0 && \
pip install --break-system-packages setuptools-scm && \
pip install --break-system-packages -U openai-whisper
```

```
sudo apt-get install -y --no-install-recommends \
                    cmake \
                    libblas-dev \
                    liblapack-dev \
                    autoconf \
                    unzip \
                    wget \
                    git \
                    vim \
                    ca-certificates \
                    pkg-config \
                    build-essential \
                    numactl \
                    libnuma-dev \
                    libtcmalloc-minimal4 \
                    sudo \
                    ffmpeg \
                    sox
```

```bash
cd $LOADGEN_FOLDER
pip install -e .
```
```bash
git clone https://github.com/vllm-project/vllm vllm-cpu && \
cd vllm-cpu && \
git checkout main && \
git log -n1 && \
pip3 install --break-system-packages -r requirements/cpu.txt && \
VLLM_TARGET_DEVICE=cpu pip install --break-system-packages . --no-build-isolation
```


## Get Model
### MLCommons Download

**Official Model download using MLCFlow Automation**

You can download the model automatically via the below command
```
mlcr get,ml-model,whisper,_rclone,_mlc --outdirname=<path_to_download> -j
```

**Official Model download using native method**

You can use Rclone to download the preprocessed dataset from a Cloudflare R2 bucket.

To run Rclone on Windows, you can download the executable [here](https://rclone.org/install/#windows).
To install Rclone on Linux/macOS/BSD systems, run:
```
sudo -v ; curl https://rclone.org/install.sh | sudo bash
```
Once Rclone is installed, run the following command to authenticate with the bucket:
```
rclone config create mlc-inference s3 provider=Cloudflare access_key_id=f65ba5eef400db161ea49967de89f47b secret_access_key=fbea333914c292b854f14d3fe232bad6c5407bf0ab1bebf78833c2b359bdfd2b endpoint=https://c2686074cb2caf5cbaf6d134bdba8b47.r2.cloudflarestorage.com
```
You can then navigate in the terminal to your desired download directory and run the following command to download the model:

```
rclone copy mlc-inference:mlcommons-inference-wg-public/Whisper/model/ ./ -P
```

### External Download (Not recommended for official submission)

**External Model download using MLCFlow Automation**

You can download the model automatically via the below command
```
TBD
```

**External Model download using native method**

+ Requires Git Large Files Storage
```bash
export CHECKPOINT_PATH=whisper-large-v3
git lfs install
git clone https://huggingface.co/openai/whisper-large-v3 ${CHECKPOINT_PATH}
cd ${CHECKPOINT_PATH} && git checkout 06f233fe06e710322aca913c1bc4249a0d71fce1
```

## Get Dataset

### Publication/Attribution
["OpenSLR LibriSpeech Corpus"](http://www.openslr.org/12/) provides over 1000 hours of speech data in the form of raw audio.
We use dev-clean and dev-other splits, which are approximately 10 hours.

### Preprocessed

**Using MLCFlow Automation**
```
mlcr get,dataset,whisper,_preprocessed,_mlc,_rclone --outdirname=<path to download> -j
```

**Native method**

Download and install rclone as decribed in the [MLCommons Download section](#mlcommons-download)

You can then navigate in the terminal to your desired download directory and run the following command to download the dataset:
```
rclone copy mlc-inference:mlcommons-inference-wg-public/Whisper/dataset/ ./ -P
```

### Unprocessed

**Using MLCFlow Automation**
```
mlcr get,dataset,whisper,_unprocessed --outdirname=<path to download> -j
```

**Native method**

If your are using docker, we provide a script to download and preprocess the dataset from the source. You can download it by running:
```bash
./download_dataset.sh
```
Otherwise, you can manually run the following commands:

```bash
cd $WHISPER_FOLDER
export WORKSPACE_DIR=.
export DATA_DIR=${WORKSPACE_DIR}/data
export LIBRISPEECH_DIR=${DATA_DIR}/LibriSpeech
export UTILS_DIR=${WORKSPACE_DIR}/utils
mkdir -p ${LIBRISPEECH_DIR}

# Downloads all Librispeech dev paritions
python ${UTILS_DIR}/download_librispeech.py \
    ${UTILS_DIR}/inference_librispeech.csv \
    ${LIBRISPEECH_DIR} \
    -e ${DATA_DIR}

# Consolidates all Librispeech paritions into common dir
mkdir -p ${LIBRISPEECH_DIR}/dev-all
cp -r ${LIBRISPEECH_DIR}/dev-clean/* \
      ${LIBRISPEECH_DIR}/dev-other/* \
      ${LIBRISPEECH_DIR}/dev-all/

# Coverts original Librispeech flac to wav and creates manifest file
python ${UTILS_DIR}/convert_librispeech.py \
   --input_dir ${LIBRISPEECH_DIR}/dev-all \
   --dest_dir ${DATA_DIR}/dev-all \
   --output_json ${DATA_DIR}/dev-all.json

# Repackages Librispeech samples into samples approaching 30s
python utils/repackage_librispeech.py --manifest ${DATA_DIR}/dev-all.json \
	                              --data_dir ${DATA_DIR} \
				      --output_dir ${DATA_DIR}/dev-all-repack \
				      --output_json ${WORKSPACE_DIR}/data/dev-all-repack.json
```

## Docker Run

### Run Performance

We provide a script to do a performance run

```bash
./reference_mlperf_perf.sh
```
### Run Accuracy

```bash
./reference_mlperf_accuracy.sh
```

## Local Run

Setup the environment variables

```bash
cd $WHISPER_FOLDER
export WORKSPACE_DIR=.
export DATA_DIR=${WORKSPACE_DIR}/data
export MODEL_PATH=${WORKSPACE_DIR}/model
export MANIFEST_FILE="${DATA_DIR}/dev-all-repack.json"
export RUN_LOGS=${WORKSPACE_DIR}/run_output
export SCENARIO="Offline"

export NUM_CORES=$(($(lscpu | grep "Socket(s):" | awk '{print $2}') * $(lscpu | grep "Core(s) per socket:" | awk '{print $4}')))
export NUM_NUMA_NODES=$(lscpu | grep "NUMA node(s)" | awk '{print $NF}')
export CORES_PER_INST=$((${NUM_CORES} / ${NUM_NUMA_NODES}))
export OMP_NUM_THREADS=${CORES_PER_INST}
export INSTS_PER_NODE=1
export NUM_INSTS=$((${NUM_NUMA_NODES} * ${INSTS_PER_NODE}))

export START_CORES=$(lscpu | grep "NUMA node.* CPU.*" | awk "{print \$4}" | cut -d "-" -f 1 | paste -s -d ',')
```

### Run Performance

```bash
python reference_mlperf.py \
    --dataset_dir ${DATA_DIR} \
    --model_path ${MODEL_PATH} \
    --manifest ${MANIFEST_FILE} \
    --scenario ${SCENARIO} \
    --log_dir ${RUN_LOGS} \
    --num_workers ${NUM_INSTS}
```

### Run Accuracy

**Evaluate Accuracy using  MLCFlow Automation**

```
mlcr run,accuracy,mlperf,_librispeech_whisper,_int32 --result_dir=<Path to directory where files are generated after the benchmark run>
```

**Evaluate Accuracy using native method**

```bash
python reference_mlperf.py \
    --dataset_dir ${DATA_DIR} \
    --model_path ${MODEL_PATH} \
    --manifest ${MANIFEST_FILE} \
    --scenario ${SCENARIO} \
    --log_dir ${RUN_LOGS} \
    --num_workers ${NUM_INSTS} \
    --accuracy
```

## Accuracy Target

For official submissions, accuracy is required to be 99% of the reference accuracy:
```
Word Error Rate: 2.0671%, accuracy=97.9329%
```

## FAQ

Q: Whisper's native audio input duration is fixed at 30 seconds. Is it permitted to modify the loaded duration to match the sample's specific duration?

A: No, it is not permitted to modify the loaded duration even if continuing to meet the model's accuracy threshold. Samples must be zero-padded to ensure consistent computation and accuracy criteria.

