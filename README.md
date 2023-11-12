[![CI](https://github.com/guillaumekln/faster-whisper/workflows/CI/badge.svg)](https://github.com/guillaumekln/faster-whisper/actions?query=workflow%3ACI) [![PyPI version](https://badge.fury.io/py/faster-whisper.svg)](https://badge.fury.io/py/faster-whisper)

# Faster Whisper transcription with CTranslate2

**faster-whisper** is a reimplementation of OpenAI's Whisper model using [CTranslate2](https://github.com/OpenNMT/CTranslate2/), which is a fast inference engine for Transformer models.

This implementation is up to 4 times faster than [openai/whisper](https://github.com/openai/whisper) for the same accuracy while using less memory. The efficiency can be further improved with 8-bit quantization on both CPU and GPU.

## Benchmark

For reference, here's the time and memory usage that are required to transcribe [**13 minutes**](https://www.youtube.com/watch?v=0u7tTptBo9I) of audio using different implementations:

* [openai/whisper](https://github.com/openai/whisper)@[6dea21fd](https://github.com/openai/whisper/commit/6dea21fd7f7253bfe450f1e2512a0fe47ee2d258)
* [whisper.cpp](https://github.com/ggerganov/whisper.cpp)@[3b010f9](https://github.com/ggerganov/whisper.cpp/commit/3b010f9bed9a6068609e9faf52383aea792b0362)
* [faster-whisper](https://github.com/guillaumekln/faster-whisper)@[cce6b53e](https://github.com/guillaumekln/faster-whisper/commit/cce6b53e4554f71172dad188c45f10fb100f6e3e)

### Large-v2 model on GPU

| Implementation | Precision | Beam size | Time | Max. GPU memory | Max. CPU memory |
| --- | --- | --- | --- | --- | --- |
| openai/whisper | fp16 | 5 | 4m30s | 11325MB | 9439MB |
| faster-whisper | fp16 | 5 | 54s | 4755MB | 3244MB |
| faster-whisper | int8 | 5 | 59s | 3091MB | 3117MB |

*Executed with CUDA 11.7.1 on a NVIDIA Tesla V100S.*

### Small model on CPU

| Implementation | Precision | Beam size | Time | Max. memory |
| --- | --- | --- | --- | --- |
| openai/whisper | fp32 | 5 | 10m31s | 3101MB |
| whisper.cpp | fp32 | 5 | 17m42s | 1581MB |
| whisper.cpp | fp16 | 5 | 12m39s | 873MB |
| faster-whisper | fp32 | 5 | 2m44s | 1675MB |
| faster-whisper | int8 | 5 | 2m04s | 995MB |

*Executed with 8 threads on a Intel(R) Xeon(R) Gold 6226R.*

## Requirements

* Python 3.8 or greater

Unlike openai-whisper, FFmpeg does **not** need to be installed on the system. The audio is decoded with the Python library [PyAV](https://github.com/PyAV-Org/PyAV) which bundles the FFmpeg libraries in its package.

### GPU

GPU execution requires the following NVIDIA libraries to be installed:

* [cuBLAS for CUDA 11](https://developer.nvidia.com/cublas)
* [cuDNN 8 for CUDA 11](https://developer.nvidia.com/cudnn)

There are multiple ways to install these libraries. The recommended way is described in the official NVIDIA documentation, but we also suggest other installation methods below.

<details>
<summary>Other installation methods (click to expand)</summary>

#### Use Docker

The libraries are installed in this official NVIDIA Docker image: `nvidia/cuda:11.8.0-cudnn8-runtime-ubuntu22.04`.

#### Install with `pip` (Linux only)

On Linux these libraries can be installed with `pip`. Note that `LD_LIBRARY_PATH` must be set before launching Python.

```bash
pip install nvidia-cublas-cu11 nvidia-cudnn-cu11

export LD_LIBRARY_PATH=`python3 -c 'import os; import nvidia.cublas.lib; import nvidia.cudnn.lib; print(os.path.dirname(nvidia.cublas.lib.__file__) + ":" + os.path.dirname(nvidia.cudnn.lib.__file__))'`
```

#### Download the libraries from Purfview's repository (Windows only)

Purfview's [whisper-standalone-win](https://github.com/Purfview/whisper-standalone-win) provides the required NVIDIA libraries for Windows in a [single archive](https://github.com/Purfview/whisper-standalone-win/releases/tag/libs). Decompress the archive and place the libraries in a directory included in the `PATH`.

</details>

## Installation

The module can be installed from [PyPI](https://pypi.org/project/faster-whisper/):

```bash
pip install faster-whisper
```

<details>
<summary>Other installation methods (click to expand)</summary>

### Install the master branch

```bash
pip install --force-reinstall "faster-whisper @ https://github.com/guillaumekln/faster-whisper/archive/refs/heads/master.tar.gz"
```

### Install a specific commit

```bash
pip install --force-reinstall "faster-whisper @ https://github.com/guillaumekln/faster-whisper/archive/a4f1cc8f11433e454c3934442b5e1a4ed5e865c3.tar.gz"
```

</details>

## Usage

```python
from faster_whisper import WhisperModel

model_size = "large-v2"

# Run on GPU with FP16
model = WhisperModel(model_size, device="cuda", compute_type="float16")

# or run on GPU with INT8
# model = WhisperModel(model_size, device="cuda", compute_type="int8_float16")
# or run on CPU with INT8
# model = WhisperModel(model_size, device="cpu", compute_type="int8")

segments, info = model.transcribe("audio.mp3", beam_size=5)

print("Detected language '%s' with probability %f" % (info.language, info.language_probability))

for segment in segments:
    print("[%.2fs -> %.2fs] %s" % (segment.start, segment.end, segment.text))
```

**Warning:** `segments` is a *generator* so the transcription only starts when you iterate over it. The transcription can be run to completion by gathering the segments in a list or a `for` loop:

```python
segments, _ = model.transcribe("audio.mp3")
segments = list(segments)  # The transcription will actually run here.
```

### Word-level timestamps

```python
segments, _ = model.transcribe("audio.mp3", word_timestamps=True)

for segment in segments:
    for word in segment.words:
        print("[%.2fs -> %.2fs] %s" % (word.start, word.end, word.word))
```

### VAD filter

The library integrates the [Silero VAD](https://github.com/snakers4/silero-vad) model to filter out parts of the audio without speech:

```python
segments, _ = model.transcribe("audio.mp3", vad_filter=True)
```

The default behavior is conservative and only removes silence longer than 2 seconds. See the available VAD parameters and default values in the [source code](https://github.com/guillaumekln/faster-whisper/blob/master/faster_whisper/vad.py). They can be customized with the dictionary argument `vad_parameters`:

```python
segments, _ = model.transcribe(
    "audio.mp3",
    vad_filter=True,
    vad_parameters=dict(min_silence_duration_ms=500),
)
```

### Logging

The library logging level can be configured like this:

```python
import logging

logging.basicConfig()
logging.getLogger("faster_whisper").setLevel(logging.DEBUG)
```

### Going further

See more model and transcription options in the [`WhisperModel`](https://github.com/guillaumekln/faster-whisper/blob/master/faster_whisper/transcribe.py) class implementation.

## Community integrations

Here is a non exhaustive list of open-source projects using faster-whisper. Feel free to add your project to the list!

* [whisper-ctranslate2](https://github.com/Softcatala/whisper-ctranslate2) is a command line client based on faster-whisper and compatible with the original client from openai/whisper.
* [whisper-diarize](https://github.com/MahmoudAshraf97/whisper-diarization) is a speaker diarization tool that is based on faster-whisper and NVIDIA NeMo.
* [whisper-standalone-win](https://github.com/Purfview/whisper-standalone-win) contains the portable ready to run binaries of faster-whisper for Windows.
* [asr-sd-pipeline](https://github.com/hedrergudene/asr-sd-pipeline) provides a scalable, modular, end to end multi-speaker speech to text solution implemented using AzureML pipelines.
* [Open-Lyrics](https://github.com/zh-plus/Open-Lyrics) is a Python library that transcribes voice files using faster-whisper, and translates/polishes the resulting text into `.lrc` files in the desired language using OpenAI-GPT.
* [wscribe](https://github.com/geekodour/wscribe) is a flexible transcript generation tool supporting faster-whisper, it can export word level transcript and the exported transcript then can be edited with [wscribe-editor](https://github.com/geekodour/wscribe-editor)

## Model conversion

When loading a model from its size such as `WhisperModel("large-v3")`, the correspondig CTranslate2 model is automatically downloaded from the [Hugging Face Hub](https://huggingface.co/guillaumekln).

We also provide a script to convert any Whisper models compatible with the Transformers library. They could be the original OpenAI models or user fine-tuned models.

For example the command below converts the [original "large-v3" Whisper model](https://huggingface.co/openai/whisper-large-v3) and saves the weights in FP16:

```bash
pip install transformers[torch]>=4.35.0

ct2-transformers-converter --model openai/whisper-large-v3 --output_dir whisper-large-v3-ct2 \
    --copy_files tokenizer.json --quantization float16
```

* The option `--model` accepts a model name on the Hub or a path to a model directory.
* If the option `--copy_files tokenizer.json` is not used, the tokenizer configuration is automatically downloaded when the model is loaded later.

Models can also be converted from the code. See the [conversion API](https://opennmt.net/CTranslate2/python/ctranslate2.converters.TransformersConverter.html).

### Load a converted model

1. Directly load the model from a local directory:
```python
model = faster_whisper.WhisperModel("whisper-large-v3-ct2")
```

2. [Upload your model to the Hugging Face Hub](https://huggingface.co/docs/transformers/model_sharing#upload-with-the-web-interface) and load it from its name:
```python
model = faster_whisper.WhisperModel("username/whisper-large-v3-ct2")
```

## Comparing performance against other implementations

If you are comparing the performance against other Whisper implementations, you should make sure to run the comparison with similar settings. In particular:

* Verify that the same transcription options are used, especially the same beam size. For example in openai/whisper, `model.transcribe` uses a default beam size of 1 but here we use a default beam size of 5.
* When running on CPU, make sure to set the same number of threads. Many frameworks will read the environment variable `OMP_NUM_THREADS`, which can be set when running your script:

```bash
OMP_NUM_THREADS=4 python3 my_script.py
```
