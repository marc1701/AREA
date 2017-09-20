# EigenScape Tools
## by Marc Ciufo Green

Acoustic Scene Classification system designed for research with the EigenScape database. The main features of the module provided here are:

- Tools enabling easier manipulation and segmentation of the EigenScape dataset.
- Function for extraction of spatial features using [Directional Audio Coding (DirAC) techniques][1].
- MultiGMMClassifier object for classification using a bank of Gaussian Mixture Models.
- BOF_audio_classify function to classify audio clips using the ['Bag-of-Frames' method][2].
- Functions for easy plotting of ROC curves and confusion matrices.


#### Requirements:
- Python 3.6 or later
- Python modules:
  - [numpy](http://www.numpy.org/)
  - [scipy](https://www.scipy.org/)
  - [scikit-learn](http://scikit-learn.org/stable/)
  - [resampy](https://github.com/bmcfee/resampy)
  - [librosa](http://librosa.github.io/librosa/)
  - [pandas](http://pandas.pydata.org/)
  - [matplotlib](https://matplotlib.org/)
  - [seaborn](https://seaborn.pydata.org/)
  - [soundfile](https://pysoundfile.readthedocs.io/en/0.9.0/)
  - [progressbar](https://pypi.python.org/pypi/progressbar2)

Tested using Python 3.6.2 on Windows 10 and macOS 10.12.5


#### Usage examples
##### Creating test setup
```python
import eigenscape
eigenscape.datatools.create_test_setup('../EigenScape/')
```
By default, this will split all audio files in the 'EigenScape' directory down into 30-second segments and shuffle each full recording into 4 folds for training and testing. These parameters can all be overridden:

```python
eigenscape.datatools.create_test_setup('../EigenScape/', seg_length=20, n_folds=8)
```
This will segment the audio into 20-second segments and shuffle the recordings into 8 folds (test audio clips will be from 1 single recording only). It is important to note that `seg_length` must be divisble by 600 seconds (10 minutes) and `n_folds` must be divisible by 8 (number of unique recordings per scene class in EigenScape).

Split audio files will be deposited in a folder named 'audio' and text files with information on the folds will be deposited in a folder named 'fold_info'.


##### Feature extraction
```python
data, indices, label_list = eigenscape.build_audio_featureset(
                              eigenscape.calculate_dirac,
                                dataset_directory='audio/')
```
This will use the DirAC feature extraction function built into the eigenscape module to calculate Azimuth, Elevation and Diffuseness estimates across 20 frequency bands by default, covering the frequency spectrum up to half the audio sampling frequency. `hi_freq`, `n_bands` keyword arguments can be used to override these defaults.

FIR filters are used to split the audio into subbands. 2048-tap filters are used by default, but this can also be overriden using the `filt_taps` keyword argument. This could speed up the feature extraction but lead to lower accuracies.

`eigenscape.calculate_mfccs` can also be substituted in order to use librosa MFCC extraction in place of DirAC.

`build_audio_featureset` returns:
- Features as a numpy array containing all the DirAC features for each frame of the audio in rows with class label numbers appended to the final column.
- A dictionary of row indices indicating the rows with features extracted from frames of each audio clip.
- A list of strings indicating the scene classes present in the feature set.

##### Bag-of-Frames classification
```python
from sklearn.preprocessing import StandardScaler

X = data[:, :-1]
y = data[:, -1] # extract data vectors and class targets from array

scaler = StandardScaler() # set up scaler object

train_info = eigenscape.extract_info('fold_info/fold4_train.txt')
test_info = eigenscape.extract_info('fold_info/fold4_test.txt')
# read in file lists (4th fold here)

train_indices = eigenscape.vectorise_indices(train_info)
# make vector of train data indices

X_train = X[train_indices]
y_train = y[train_indices]
# extract training data and labels from full arrays

classifier = eigenscape.MultiGMMClassifier() # set up multi GMM classifier

classifier.fit(scaler.fit_transform(X_train), y_train)
# train classifier on scaled training data and fit scaler to training data

y_test, y_score = eigenscape.BOF_audio_classify(
    classifier, scaler.transform(X), y, test_info, indices)
# classify entire audio clips (specified in test_info) by summing output from
# classifier object across all frames of the clip

```
The `BOF_audio_classify` function returns:
- `y_test` - an array with class labels for each full audio clip.
- `y_score` - an array containing probability scores from each GMM in the `MultiGMMClassifier` object.

The classifier object can be substituted for any scikit-learn object implementing the `decision_function` method.


##### Plotting results
```python
eigenscape.plot_confusion_matrix(y_test, y_score, label_list)
```
This will plot a confusion matrix based on the output from the classifier. This function returns:
- Confusion matrix as a numpy array
- Accuracies by class
- Overall accuracies
- scikit-learn classification report

`eigenscape.plot_roc` is also provided to plot ROC curves based on classifier output.

<!-- Need then to do semi-detailed comments in main files to indicate e.g. when special features such as passing in a custom classifier object etc. is done -->

[1]:http://www.aes.org/e-lib/browse.cfm?elib=14838
[2]:http://asa.scitation.org/doi/10.1121/1.2750160
