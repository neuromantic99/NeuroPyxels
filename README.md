# NeuroPyxels: loading, processing and plotting Neuropixels data in python

[![PyPI Version](https://img.shields.io/pypi/v/npyx.svg)](https://pypi.org/project/npyx/)

NeuroPyxels (npyx) is a python library built for electrophysiologists using Neuropixels electrodes. It features a suite of core utility functions for loading, processing and plotting Neuropixels data.

This package results from the needs of an experimentalist who could not stand MATLAB, hence wrote himself a suite of functions to emotionally bear with doing neuroscience. There isn't any dedicated preprint available yet, so if you enjoy this package and use it for your research, please star [the github repo](https://github.com/m-beau/NeuroPyxels) (click on the top-right star button!) and cite [this paper](https://www.nature.com/articles/s41593-019-0381-8). Cheers!

## Documentation:
Npyx works in harmony with the data formatting employed by [SpikeGLX](https://billkarsh.github.io/SpikeGLX/) used in combination with [Kilosort](https://github.com/MouseLand/Kilosort) and [Phy](https://phy.readthedocs.io/en/latest/).

<ins>Npyx is fast because it never computes the same thing twice</ins> - in the background, it saves most relevant outputs (spike trains, waveforms, correlograms...) at **./npyxMemory**, from where they are simply reloaded if called again. An important parameter controlling this behaviour is **`again`** (boolean), by default set to False: if True, the function will recompute the output rather than loading it from npyxMemory. This is important to be aware of this behaviour, as it can lead to mind boggling bugs. For instance, if you load the train of unit then re-spikesort your dataset, e.g. you split unit 56 in 504 and 505, the train of the old unit 56 will still exist at ./npyxMemory and you will be able to load it even though the unit is gone!

Most npyx functions take at least one input: **`dp`**, which is the path to your Kilosort/phy dataset. You can find a [full description of the structure of such datasets](https://phy.readthedocs.io/en/latest/sorting_user_guide/#installation) on phy documentation.

Other typical parameters are: **`verbose`** (whether to print a bunch of informative messages, useful when debugging), **`saveFig`** (boolean) and **`saveDir`** (whether to save the figure in saveDir for plotting functions)

Importantly, dp can also be the path to a merged dataset, generated with npyx.merge_datasets() - every function will (normally) run as smoothly on folders as any regular kilosort dataset folder. See below for more details.

More precisely, every function requires the files `myrecording.ap.meta`, `spike_times.npy` and `spike_clusters.npy`. Then particular functions will require particular files: loading waveforms with `npyx.spk_wvf.wvf` or extracting your sync channel with `npyx.io.get_npix_sync` require the raw data `myrecording.ap.bin`, extracting the spike sorted group of your units `cluster_groups.tsv` and so on.

Example use cases are:
### Load synchronization channel
```python
from npyx.io import get_npix_sync
dp = 'path/to/dataset'
onsets, offsets = get_npix_sync(dp)
# onsets/offsets are dictionnaries
# whose keys are ids of sync channel where signal was detected,
# and values the times of up (onsets) or down (offsets) threshold crosses in seconds.
```
### Get good units from dataset
```python
from npyx.gl import get_units
dp = 'path/to/dataset'
units = get_units(dp, quality='good')
```
### Load spike times from unit u
```python
from npyx.spk_t import trn
dp = 'path/to/dataset'
u=234
t = trn(dp, u) # gets all spikes from unit 234, in samples
```

### Load waveforms from unit u
```python
from npyx.io import read_spikeglx_meta
from npyx.spk_t import ids, trn
from npyx.spk_wvf import get_peak_chan, wvf, templates
dp = 'path/to/dataset'
u=234
# returns a random sample of 100 waveforms from unit 234, in uV, across 384 channels
waveforms = wvf(dp, u, n_waveforms=100, t_waveforms=82) # return array of shape (100, 82, 384) by default
waveforms = wvf(dp, u, periods='regular')
waveforms = wvf(dp, u, spike_ids=None)
# Get the unit peak channel (channel with the biggest amplitude)
peak_chan = get_peak_chan(dp,u)
# extract the waveforms located on peak channel
w=waves[:,:,peak_chan]

# Extract waveforms of spikes occurring between
# 900 and 1000s in the recording, because that's when your mouse scratched its butt
fs=read_spikeglx_meta['sRateHz']
t=trn(dp,u)/fs # convert in s
ids=ids(dp,u)[(t>900)&(t<1000)]
waves = wvf(dp, u, spike_ids=ids)

# If you want to load the templates instead (lighter)
temp = templates(dp,u) # return array of shape (n_templates, 82, n_channels)
```

### Compute auto/crosscorrelogram between 2 units
```python
from npyx.corr import ccg
dp = 'path/to/dataset'
# returns ccg between 234 and 92 with a binsize of 0.2 and a window of 80
c = ccg(dp, [234,92], cbin=0.2, cwin=80)
```

### Plot correlograms and waveforms from unit u
```python
# all plotting functions return matplotlib figures
from npyx.plot import plot_wvf
dp = 'path/to/dataset'
u=234
# plot waveform, 2.8ms around center, on 8 channels around peak channel,
# with no single waveforms in the background (sample_lines)
fig = plot_wvf(dp, u, Nchannels=8, t_waveforms=2.8, sample_lines=0)
```

```python
# plot ccg between 234 and 92
fig = plot_ccg(dp, [u,92], cbin=0.2, cwin=80, as_grid=True)
```
![ccg](images/ccg.png)

### Merge datasets acquired on two probes simultaneously
```python
# The three recordings need to include the same sync channel.
from npyx.merger import Merger
dps = ['same_folder/lateralprobe_dataset',
       'same_folder/medialprobe_dataset',
       'same_folder/anteriorprobe_dataset']
probenames = ['lateral','medial','anterior']
dp_dict = {p:dp for p, dp in zip(dps, probenames)}
merged = Merger(dp_dic)
dp=merged.dp_merged
# This will merge the 3 datasets (only relevant information, not the raw data) in a new folder at
# same_folder/prophyler_lateralprobe_dataset_medialprobe_dataset_anteriorprobe_dataset
# which can then be used as if it were a single dataset by all npyx functions.
# The only difference is that units now need to be called as floats, of format unit_id.dataset_id.
# lateralprobe, medial probe and anteriorprobe dataset_ids will be 0,1 and 2.
t = trn(dp, 92.1) # get spikes of unit 92 in dataset 1 i.e. medialprobe
fig=plot_ccg(dp,[10,0,92.1,cbin=0.2,cwin=80]) # compute CCG between 2 units across datasets
```

There isn't any better doc atm - email [Maxime Beau](mailto:maximebeaujeanroch047@gmail.com) (PhD Hausser lab, UCL at time of writing) if you have any questions!

<br/>

## Installation:

Using a conda environment is very much advised. Instructions here: [manage conda environments](https://docs.conda.io/projects/conda/en/latest/user-guide/tasks/manage-environments.html)

Npyx supports Python 3.7+.

- as a user
  - from pip (normally up to date)
  ```bash
  conda create -n my_env python=3.7
  conda activate my_env
  pip install npyx
  python -c 'import npyx' # should not return any error
  # If it does, install any missing dependencies with pip (hopefully none!)
  ```
  - from the remote repository (always up to date - still private at time of writing, pip is a prerelease)
  ```bash
  conda activate env_name
  pip install git+https://github.com/Npix-routines/NeuroPyxels@master
  ```
- as a superuser (recommended if plans to work on it/regularly pull upgrades)
  > Tip: in an ipython/jupyter session, use `%load_ext autoreload` then `%autoreload` to make your local edits active in your session without having to restart your kernel. Amazing for development.
    ```bash
    conda activate my_env
    cd path/to/save_dir # any directory where your code will be accessible by your editor and safe. NOT downloads folder.
    git clone https://github.com/Npix-routines/NeuroPyxels
    cd NeuroPyxels
    python setup.py develop # this will create an egg link to save_dir, which means that you do not need to reinstall the package each time you pull an udpate from github.
    ```
    and pull every now and then:
    ```bash
    conda activate env_name
    cd path/to/save_dir/NeuroPyxels
    git pull
    # And that's it, thanks to the egg link no need to reinstall the package!
    ```
<br/>

## Developer cheatsheet

Useful link to [create a python package from a git repository](https://towardsdatascience.com/build-your-first-open-source-python-project-53471c9942a7)


### Push local updates to github:
```bash
# ONLY ON DEDICATED BRANCH

cd path/to/save_dir/NeuroPyxels
git checkout DEDICATED_BRANCH_NAME # ++++++ IMPORTANT
git add.
git commit -m "commit details - try to be specific"
git push origin DEDICATED_BRANCH_NAME # ++++++ IMPORTANT

# Then pull request to master branch using the online github green button! Do not forget this last step, to allow the others repo to sync.
```

### Push local updates to PyPI (Maxime)
First change the version in ./setup.py in a text editor
```python
setup(name='npyx',
      version='1.0',... # change to 1.1, 1.1.1...
```
Then delete the old distribution files before re-generating them for the new version using twine:
```bash
rm -r ./dist
rm -r ./build
rm -r ./npyx.egg-info
python setup.py sdist bdist_wheel # this will generate version 1.1 wheel without overwriting version 1.0 wheel in ./dist
```
Before pushing them to PyPI (older versions are saved online!)
```bash
twine upload dist/*

Uploading distributions to https://upload.pypi.org/legacy/
Enter your username: your-username
Enter your password: ****
Uploading npyx-1.1-py3-none-any.whl
100%|████████████████████████████████████████████████████████| 156k/156k [00:01<00:00, 96.8kB/s]
Uploading npyx-1.1.tar.gz
100%|█████████████████████████████████████████████████████████| 150k/150k [00:01<00:00, 142kB/s]

```
