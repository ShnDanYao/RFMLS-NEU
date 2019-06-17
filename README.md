# Contents

* [Code Overview](#code-overview)
* [Preprocessing a Dataset](#preprocessing-a-dataset) 
     *  [An Example: Preprocessing a General Task](#an-example-preprocessing-a-general-task)
     *  [An Example: Preprocessing a New Dataset](#an-example-preprocessing-a-new-dataset)
* [Training and Testing](#training-and-testing)
     *  [An Example: Running a General Task](#an-example-running-a-general-task)
     *  [An Example: Running a New Dataset](#an-example-running-a-new-dataset)
* [Detailed Overview](#detailed-overview)

## Preprocessing a Dataset
Preprocessing amounts to filtering (re-centering the signals to a base band based on their metadata) and equalization. Equalization only applies to WiFi signals, and is optional. Preprocessing also generates files needed by both training and testing; this is why all data needs to be preprocessed (whether WiFi, ADS-B, or a new type of dataset). 

Note that if a task contains both WiFi and ADS-B data, both will be preprocessed simutlaneously (i.e., in one execution of the  code with `--preprocess True` evoked). 
 
 The following arguments can be specified during preprocessing:
```bash
    --train_tsv       	path to the train tsv file
    --test_tsv        	path to the test tsv fle
    --task              set task name
    --root_wifi       	path to the raw wifi signals
    --root_adsb       	path to the raw adsb signals
    --out_root_data   	output path for permanently storing preprocessed data
    --out_root_list   	output path for permanently storing pickled data lists
    --wifi_eq		Specify whether wifi signals need to be equalized or not {True | False}. default: False
    --newtype_process	[New-Type Signal] process a dataset with New-Type Signal or not {True | False}. 
    			default: False
    --root_newtype      [New-Type Signal] path to the raw New-Type Signal {True | False}. 
    			default: False
    --newtype_filter 	[New-Type Signal] filter New-Type Signal or not {True | False}. 
    			default: False
    --signal_BW_useful  [New-Type Signal] This argument specifies the bandwidth (measured in Hz) used to transmit the actual
    					  information. Most wireless standards reserve portions of the bandwidth to the so-called
					  guard bands which are used to reduce interference across multiple bands. Guard bands are
					  located on the sides of the overall bandwidth, and the variable signal_BW_useful is used to
					  properly extract useful information from the recorder signal, thus removing any interference
					  generated by transmissions occurring at other bands. As an example, for WiFi signals at
					  2.4GHz, signal_BW_useful=16.5MHz
					  default: {16.5e6 | 17e6} depending on signal frequency
    --num_guard_samp    [New-Type Signal] This argument represents the number of guard samples before and after each packet
    				          transmission. Each packet transmission is preceded and followed by num_guard_samp samples that
					  do not contain any data transmission.
					  default: 2e6
```
Preprocessed data are stored under `/$out_root_data/$task/datatype {wifi, ADS-B, newtype}/`. There is no need to specify the datatype, which will be detected automatically according to `.tsv` files. 

Other generated files are stored under `/$out_root_list/$task/datatype {wifi, ADS-B, newtype, mixed}/`. This folder will contain five files needed by both training and testing: 
 * a `file_list.pkl` file containing path of all preprocessed example needed by training and testing, 
 * a `label.pkl` file containing a dictionary {path of preprocessed example: device name(e.g., 'wifi_100_crane-gfi_1_dataset-8965')},
 * a `device_ids.pkl` file containing a dictionary {device name(e.g., 'wifi_100_crane-gfi_1_dataset-8965'): device id(an integer)},
 * a `partition.pkl` file containing training and testing separation,
 * a `stats.pkl` file containing computed data statistics needed by training.
 
Specifically, the folder `/$out_root_data/$task/mixed/` contains generated files for all datatypes provided in `.tsv` files. For example, if there are two datatypes in `.tsv` files, e.g., `wifi` and `ADS-B`,  it will generate files for both `filtered wifi` and `raw ADS-B` signals; if there are three datatypes in `.tsv` files, `wifi`, `ADS-B` and `newtype`,  it will generate files for `filtered wifi`, `raw ADS-B`, and `{raw | filtered, depends on $newtype_filter} newtype` signals together. Notice that equalized and raw signals are in different domain and cannot be trained/tested together, the folder `~/mixed/` will contain generated files only for `filtered wifi` no matter argument `--wifi_eq` is set to `True` or not.

An example of running the preprocessing would be: 
```bash
$ python ./preprocessing/main.py \
	--task MyTask \
	--train_tsv /scratch/MyTask.train.tsv \
	--test_tsv /scratch/MyTask.test.tsv \
	--root_wifi /mnt/disk1/wifi/ \
	--root_adsb /mnt/disk2/adsb/ \
	--out_root_data ./data/test \
	--out_root_list ./data/test_list
```
This prints the following output:
```
Extracting data...
*************** Extraction .meta/.data according to .tsv ***************
Initialize 40 workers.
Processing tsv: /scratch/MyTask.train.tsv
Processing tsv: /scratch/MyTask.test.tsv
*************** Filtering WiFi signals ***************
There are 50 devices to filter
Processing folder: [...]
*************** Create partitions, labels and device ids for training. Compute stats also.***************
generating files for WiFi, ADS-B and mixed dataset
***************creating labels for dataset:wifi raw_samples***************
Created auxiliary files:
('Number of devices', 50)
('Number of examples', 13650)
Save files to: ./data/test_list/MyTask/wifi/raw_samples
creating partition files for dataset:wifi raw_samples
13650/13650 [00:00<00:00, 149579.36it/s]
compute stats for dataset:wifi raw_samples
10900/10900 [00:25<00:00, 422.02it/s]
***************creating labels for dataset:ADS-B***************
Created auxiliary files:
('Number of devices', 50)
('Number of examples', 13650)
Save files to:./data/test_list/MyTask/ADS-B/
creating partition files for dataset:ADS-B
13650/13650 [00:00<00:00, 155440.33it/s]
('Num Ex in Train', 10900)
('Num Ex in Test', 2750)
compute stats for dataset:ADS-B 
10900/10900 [00:20<00:00, 527.07it/s]
***************creating labels for dataset:mixed***************
Created auxiliary files:
('Number of devices', 100)
('Number of examples', 27300)
Save files to:./data/test_list/MyTask/mixed/
creating partition files for dataset:mixed
27300/27300 [00:00<00:00, 158303.85it/s]
('Num Ex in Train', 21800)
('Num Ex in Test', 5500)
compute stats for dataset:mixed 
21800/21800 [00:46<00:00, 470.44it/s]
Skipping model framework.
```
Note that the above command mounts folders `/scratch`, `/mnt`, and `/data`, as they are to be used for input arguments and persisting data.

To preprocess data using equalization, an additional `--wifi_eq True` argument needs to be passed.

### An Example: Preprocessing a New Dataset
An example of running the preprocessing for a new dataset with new signal type would be: 
```bash
$  python ./preprocessing/main.py \
	--task MyTask \
	--train_tsv /scratch/MyTask.train.tsv \
	--test_tsv /scratch/MyTask.test.tsv \
	--root_wifi /mnt/disk1/wifi/ \
	--root_adsb /mnt/disk2/adsb/ \
	--root_newtype /mnt/disk2/adsb/NewType/ \
	--newtype_process True \
	--newtype_filter True \
	--signal_BW_useful 16.5e6 \
	--num_guard_samp 2e-6 \
	--out_root_data ./data/test \
	--out_root_list ./data/test_list \
```
If the new dataset contains only `newtype` signals, there is no need to specify `--root_wifi` and `root_adsb`. Otherwise, please specify these two arguments accordingly.

The code presumes that the content of the meta/data files follows the syntax of Wifi signals.

Note that arguments `--signal_BW_useful` and `--num_guard_samp` need to be set for filtering a new signal type. There is no need to specify these two arguments for Wifi or ADSB signals.


## Training and Testing
Running a task consists of a `train_phase` and a `test_phase`. 

Both training and testing require that data is first pre-processed (so that, beyond filtering and/or equalizing, it is in a format understandable by our neural network), as described in the previous section. 

Command line arguments for training and testing include:

```bash
    --data_path       		path containing training, validation, testing, stats dictionaries.
    		      		        directory used for --out_root_list.
    --data_type	      		the data type, either 'wifi', 'ADS-B', 'newtype', or 'mixed'. 'mixed' trains the model jointly 					                      on all datatype present. default: wifi
    --model_flag      		set the model architecture {baseline | resnet1d}. default: baseline
    --exp_name                  create an exp_name folder to save logs, models etc				
```
The `--data_path` folder corresponds to the `--out_root_list` folder where the output from the preprocessing stage was stored. Running tests is a silent execution (it only prints errors in `stderr`). Trained model weights as well as classification outcomes are stored under `/results/$task/$data_type/$exp_name`. This folder will contain four files: 
 * a `json` file containing the model structure, 
 * an `hdf5` file containing the model weights, 
 * a `config` file containing the parameter configuration used during training and testing, and
 * a `log.out` file that contains a detailed output log, including the final test accuracy. The last line contains the per slice and per example/signal accuracy on the test set. 
 
For a full list of arguments of the framework you may use `--help`.

```bash

Model usage :
    --exp_name            create an exp_name folder to save logs, models etc
    --base_path           directory used for --out_root_list
    --stats_path          directory used for --out_root_list
    --save_path           path for saving the model and logs, it should be the parent folder of exp_name
    --file_type           Specify type {mat | pkl} of file you want to read
    --devices             set the number of devices in dataset. default: 50
    --train               enable training in model {True | False}. default: True
    --test                enable testing in model {True | False}. default: True
    --val_from_train .    If validation not present in partition file, generate one from the training set. default: False
    --cont                specify continue training/testing or not {True | False}. 
                          if testing a pre-trained model, please specify it as True. 
                          default: False. 
    --restore_model_from  enable to load pretrained model structure {True | False} 
                          default: False
    --restore_weight_from enable to load pre-trained model weight {True | False}. 
                          default: False
    --model_flag          set the model architecture {baseline | resnet1d}. default: baseline
    --slice_size          set the slice size. default: 256
    --dropout_flag        enable dropout {True | False}. default: False
    --fc_stack            set the number of fully connected layers. default: 3
    --cnn_stack           set the number of convultion layers. default: 5
    --K                   set the kappa value. default: 10
    --batch_size          set the batch size. default: 128
    --lr                  set the learning rate for optimizer. default: 0.0001
    --epochs              set the number of epochs. default: 25
    --batchnorm           enable batchnormalization between layers {True | False}.
                          default: False
    --add_padding         add padding if examples are smaller than slice size {True | False}.
                          default: True
    --normalize           enable to normalize data using mean and standard deviation from stats
                          files (if stats does not have this info, it is ignored) {True | False}.
                          default: True
    --early_stopping      enable early stopping for model performance {True | False}.
                          default: True
    --patience            set number of epochs to wait for improved trained accuracy.
                          default: 7
    --test_stride         specify the stride to use for testing. default: 16
```