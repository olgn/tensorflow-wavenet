# WaveNet for Drum Sample Creation

This is an edit of the ibab/tensorflow-wavenet implementation in TensorFlow for the creation of novel drum sounds.

To get it going, first ensure that TensorFlow is installed on your computer, as well as Theano and Librosa. See below for instructions on how to install the requirements besides TensorFlow. 

Once you have the code and packages installed, put some samples in the samples/training/ folder. A good idea is to put some kicks in the samples/training/kicks folder, for example. Then the program is run in the following way (thedetailed explanation is below, this is just the easy version to get something going quick):

```
python train.py --data_dir='./samples/kicks' --sample_size=88200 --num_steps=100000
```
note that you should change the data_dir variable to whatever folder you put the samples in, the sample_size defaults to 100000, but generally one second of data is enough for short kicks (the sample_size variable controls essentially the 'width' of the neural network's input), and num_steps to however many iterations you wish for the network to run at. Keep in mind that the system stores checkpoints along the way, so you can cancel the training process at any time and the network architectures will be stored in './logdir/train/some_date_string/' and will be labeled like 'model.ckpt-n' where n is the number of iterations for which the checkpoint was stored.

Once the network has trained, you can generate a sample by using the following code
```
python generate.py --samples=44100 --wav_out_path=./samples/generated/name_of_file.wav  ./logdir/train/some_date_string/model.ckpt-n
```
and this will use the network architecture stored in './logdir/train/some_date_string/model.ckpt-n' to generate a wav file of length 44100 samples, and with the path being './samples/generated/name_of_file.wav'. More detailed description of generate.py can be found below, as well as looking through the source code and the get_arguments() function that describes all of the parameters we can change on generate init. 

More to come!

# A TensorFlow implementation of DeepMind's WaveNet paper

[![Build Status](https://travis-ci.org/ibab/tensorflow-wavenet.svg?branch=master)](https://travis-ci.org/ibab/tensorflow-wavenet)

This is a TensorFlow implementation of the [WaveNet generative neural
network architecture](https://deepmind.com/blog/wavenet-generative-model-raw-audio/) for audio generation.

<table style="border-collapse: collapse">
<tr>
<td>
<p>
The WaveNet neural network architecture directly generates a raw audio waveform,
showing excellent results in text-to-speech and general audio generation (see the
DeepMind blog post and paper for details).
</p>
<p>
The network models the conditional probability to generate the next
sample in the audio waveform, given all previous samples and possibly
additional parameters.
</p>
<p>
After an audio preprocessing step, the input waveform is quantized to a fixed integer range.
The integer amplitudes are then one-hot encoded to produce a tensor of shape <code>(num_samples, num_channels)</code>.
</p>
<p>
A convolutional layer that only accesses the current and previous inputs then reduces the channel dimension.
</p>
<p>
The core of the network is constructed as a stack of <em>causal dilated layers</em>, each of which is a
dilated convolution (convolution with holes), which only accesses the current and past audio samples.
</p>
<p>
The outputs of all layers are combined and extended back to the original number
of channels by a series of dense postprocessing layers, followed by a softmax
function to transform the outputs into a categorical distribution.
</p>
<p>
The loss function is the cross-entropy between the output for each timestep and the input at the next timestep.
</p>
<p>
In this repository, the network implementation can be found in <a href="./wavenet/model.py">model.py</a>.
</p>
</td>
<td width="300"
<img src="images/network.png" width="300"></img>
</td>
</tr>
</table>

## Requirements

TensorFlow needs to be installed before running the training script.
TensorFlow 0.10 and the current `master` version are supported.

In addition, [librosa](https://github.com/librosa/librosa) must be installed for reading and writing audio.

To install the required python packages (except TensorFlow), run
```bash
pip install -r requirements.txt
```

## Training the network

You can use any corpus containing `.wav` files.
We've mainly used the [VCTK corpus](http://homepages.inf.ed.ac.uk/jyamagis/page3/page58/page58.html) (around 10.4GB, [Alternative host](http://www.udialogue.org/download/cstr-vctk-corpus.html)) so far.

In order to train the network, execute
```bash
python train.py --data_dir=corpus
```
to train the network, where `corpus` is a directory containing `.wav` files.
The script will recursively collect all `.wav` files in the directory.

You can see documentation on each of the training settings by running
```bash
python train.py --help
```

You can find the configuration of the model parameters in [`wavenet_params.json`](./wavenet_params.json).
These need to stay the same between training and generation.

### Global Conditioning
Global conditioning refers to modifying the model such that the id of a set of mutually-exclusive categories is specified during training and generation of .wav file.
In the case of the VCTK, this id is the integer id of the speaker, of which there are over a hundred.
This allows (indeed requires) that a speaker id be specified at time of generation to select which of the speakers it should mimic. For more details see the paper or source code.

### Training with Global Conditioning
The instructions above for training refer to training without global conditioning. To train with global conditioning, specify command-line arguments as follows:
```
python train.py --data_dir=corpus --gc_channels=32
```
The --gc_channels argument does two things: 1) It tells the train.py script that
it should build a model that includes global conditioning. 2) It specifies the
size of the embedding vector that is looked up based on the id of the speaker.

The global conditioning logic in train.py and audio_reader.py is "hard-wired" to the VCTK corpus at the moment in that it expects to be able to determine the speaker id from the pattern of file naming used in VCTK, but can be easily be modified.

## Generating audio

[Example output](https://soundcloud.com/user-731806733/tensorflow-wavenet-500-msec-88k-train-steps)
generated by @jyegerlehner based on speaker 280 from the VCTK corpus.

You can use the `generate.py` script to generate audio using a previously trained model.

### Generating without Global Conditioning
Run
```
python generate.py --samples 16000 model.ckpt-1000
```
where `model.ckpt-1000` needs to be a previously saved model.
You can find these in the `logdir`.
The `--samples` parameter specifies how many audio samples you would like to generate (16000 corresponds to 1 second by default).

The generated waveform can be played back using TensorBoard, or stored as a
`.wav` file by using the `--wav_out_path` parameter:
```
python generate.py --wav_out_path=generated.wav --samples 16000 model.ckpt-1000
```

Passing `--save_every` in addition to `--wav_out_path` will save the in-progress wav file every n samples.
```
python generate.py --wav_out_path=generated.wav --save_every 2000 --samples 16000 model.ckpt-1000
```

Fast generation is enabled by default.
It uses the implementation from the [Fast Wavenet](https://github.com/tomlepaine/fast-wavenet) repository.
You can follow the link for an explanation of how it works.
This reduces the time needed to generate samples to a few minutes.

To disable fast generation:
```
python generate.py --samples 16000 model.ckpt-1000 --fast_generation=false
```

### Generating with Global Conditioning
Generate from a model incorporating global conditioning as follows:
```
python generate.py --samples 16000  --wav_out_path speaker311.wav --gc_channels=32 --gc_cardinality=377 --gc_id=311 model.ckpt-1000
```
Where:

`--gc_channels=32` specifies 32 is the size of the embedding vector, and
must match what was specified when training.

`--gc_cardinality=377` is required
as 376 is the largest id of a speaker in the VCTK corpus. If some other corpus
is used, then this number should match what is automatically determined and
printed out by the train.py script at training time.

`--gc_id=311` specifies the id of speaker, speaker 311, for which a sample is
to be generated.

## Running tests

Install the test requirements
```
pip install -r requirements_test.txt
```

Run the test suite
```
./ci/test.sh
```

## Missing features

Currently there is no local conditioning on extra information which would allow
context stacks or controlling what speech is generated.


## Related projects

- [tex-wavenet](https://github.com/Zeta36/tensorflow-tex-wavenet), a WaveNet for text generation.
- [image-wavenet](https://github.com/Zeta36/tensorflow-image-wavenet), a WaveNet for image generation.
