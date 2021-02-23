# SciXtracerPy usage

This document shows the usage of the scixtracer python API to manage an scientific data analysis experiment.


# import the library

Let's import the libraries we need in this usecase:

```python
import os
import scixtracerpy as sx 
from skimage import io
```

# create experiment 

first we define variables that contains the path of the data directories:

```python
origin_data_dir="path/to/raw_data_dir/"
workingdir="path/to/userdata/"
```

Let's create a new experiment with the minimal required metadata:
```python
experiment = sx.new_experiment(name="myexp", author="sprigent", uri=workingdir)
```

From any other script we can then open the experiment with the command:
```python
experiment = sx.open_experiment(os.path.join(origin_data_dir, "myexp"))
```


# import a data single

We can now import data files one by one using the ``import_data`` method. In this example we import all the ``tif`` files from a directory and create two tags. The tag ``Population`` is extracted from the filename using a keyord search in the filename. The second tak is a unique ID associated to each data.  

```python
files = os.listdir(origin_data_dir)
id = O
for file in files:
    if file.endswith('.tif'):
        # extract tags
        population = ""
        if file.contains("population1"):
            population = "population1"
        elif file.contains("population2"):
            population = "population1"         
        id+=1
        experiment.import_data(file, 
                               name=file, 
                               author="sprigent", 
                               date="2019-10-10", 
                               format="tif", 
                               tags={"Population": "population1", 
                                     "ID": id}
                               )
```


# import and tag data from dir

The SciXtracer python API embed methods to automate the import of data files from a directory

```python
experiment.import_dir(origin_data_dir, 
                      filter_="\.tif$", 
                      author='Sylvain Prigent', 
                      format_='tif', 
                      date='2019-03-17', 
                      copy_data=True)
```

Two methods are available to tag data by parsing the filenames:

```python
# Here we create the key Population, and retrieve the value population1 and population2 that should be writen in the filename
experiment.tag_from_name("Population", ['population1', 'population2'] )
```

```python
# Here we create the key Number, and retrieve the value from the characters located after the first _ of the filename
my_experiment.tag_using_seperator(tag="Number", separator="_", value_position=1)
```

We now have an experiment filled with raw data. Each raw data is tagged with a vocabulary (tag keys) that are specific to the experiment.

# query dataset and data

We can query the experiment using the files names or the tags.

## Get a dataset

```python
# here we query the raw dataset (called 'data')
dataset = experiment.get_dataset(name='data')
# dataset is a python object containing the dataset metadata
```

## Get a data from a dataset

We can get one or multiple data using queries on a dataset:

```python
#using the data name
rawdata = dataset.get_data("name=population1_001.tif")

#using a query
rawdata = dataset.get_data("Population=population1 AND Number=001")

#The query can return multiple data
rawdata_list = dataset.get_data("Population=population1")
```

Tags are associated only to the raw data but propagated to the derivated (ie processed) data.

```python
# query a processed dataset
processed_dataset = experiment.get_dataset('deconvolution')
# query data from the 'deconvolution' dataset using tags
processeddata_list = processed_dataset.get_data("Population=population1")
# or with a single result
processeddata = processed_dataset.get_data("name=population1_001.tif")
```

We can get the origin (ie rawdata) from any derivated (ie processed) data:

```python
rawdata = processeddata.get_origin()
```

If we have successive processing steps we can call the ``get_parent()`` method to get the direct parent data. 

For example, we have a rawdata image, we apply a deconvolution algorithm that creates a new processed data of the deconvluted image. And then we apply a watershed segmentation that gives a new processed data of the segmented image. Then, we can retrive all the intermediate images with the API:

```python
# first we get the whatershed image metadata:
watershed_data = experiment.get_dataset('watershed').get_data("name=population1_001.tif")

# then we get it parent data, which is the deconvuted image
deconvouted_image = watershed_data.get_parent()

# and finally the raw data
rawdata = deconvouted_image.get_parent()

# the raw data can be obtained directly by:
rawdata = watershed_data.get_orign()
```

# run a threshod in plain python

In this section we see how to generate metadata file for a new prcessed data that we created directly in python.

Let's consider the following example. We have a set of raw data in an experiment as created by the API showed above. We suppose having 20 images tagged with ``Population:population1`` and 20 images tagged with ``Population:population2``. We want to apply a threshold on these images with a threshold at ``100`` for the images with the tag ``Population:population1`` and ``125`` for the images with the tag ``Population:population2``.
All the result will be stored in a single dataset called ``threshold``

## Step 1: create the dataset

```python
th_dataset = experiment.new_dataset('threshold')
```

## Step 2: process images depending on tags

For images with population 1

```python
# create a run metadata
run1 = th_dataset.new_run(process='uniqueIdOfMyAlgorithm', inputs={"name": "image", "dataset"="data", "query":"Population=Population1"}, parameters={"threshold": 100})

# proces the images with tag population 1
rawdata_list = experiment.get_dataset('data').get_data("Population=population1")
for rawdata in rawdata_list:
    # create the processed metadata
    processed_data = th_dataset.new_data(run=run1, origin=rawdata, format='tif')
    # threshold the image
    im = io.imread(rawdata.uri)
    im_out = im > 100
    io.imsave(processed_data.uri, im_out)
```

we can do a copy&paste for ``population2``, or make one single loop to ovoid copy&paste

```python
# create a conditions list
conditions=[{"tag": "population1", "threshold":100}, {"tag": "population2", "threshold":125}]

for condition in conditions:
    # create a run metadata
    run = th_dataset.new_run(process='uniqueIdOfMyAlgorithm', inputs={"name": "image", "dataset"="data", "query":"Population="+condition["tag"]}, parameters={"threshold": condition["threshold"]})

    # proces the images with tag population 1
    rawdata_list = experiment.get_dataset('data').get_data("Population="+condition["tag"])
    for rawdata in rawdata_list:
        # create the processed metadata
        processed_data = th_dataset.new_data(run=run, origin=rawdata, format='tif')
        # threshold the image
        im = io.imread(rawdata.uri)
        im_out = im > condition["threshold"]
        io.imsave(processed_data.uri, im_out)
```
