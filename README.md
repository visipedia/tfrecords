# tfrecords
Convenience functions to create tfrecords that can be used with classification, detection and keypoint localization systems. The [create_tfrecords.py](create_tfrecords.py) file will help create the correct tfrecords to feed into those systems.

# Inputs

The data needs to be stored in an [Example protocol buffer](https://www.tensorflow.org/code/tensorflow/core/example/example.proto). The protocol buffer will have the following fields:

| Key | Value |
|-----|-------|
| image/id | string containing an identifier for this image. |
| image/filename | string containing a file system path to the of the image file. |
| image/encoded | string containing JPEG encoded image in RGB colorspace|
| image/height | integer, image height in pixels |
| image/width | integer, image width in pixels |
| image/colorspace | string, specifying the colorspace, e.g. 'RGB' |
| image/channels | integer, specifying the number of channels, e.g. 3 |
| image/format | string, specifying the format, e.g. 'JPEG' |
| image/extra | string, any extra data can be stored here. For example, this can be a string encoded json structure.
| image/class/label | integer specifying the index in a classification layer. The label ranges from [0, num_labels), e.g 0-99 if there are 100 classes. |
|  image/class/text | string specifying the human-readable version of the label e.g. 'White-throated Sparrow' |
| image/class/conf | float value specifying the confidence of the label. For example, a probability output from a classifier. |
| image/object/count | an integer, the number of object annotations. For example, this should match the number of bounding boxes. |
| image/object/area | a float array of object areas; normalized coordinates. For example, the simplest case would simply be the area of the bounding boxes. Or it could be the size of the segmentation. Normalized in this case means that the area is divided by the (image width x image height) |
| image/object/id | an array of strings indicating the id of each object. |
| image/object/bbox/xmin | a float array, the left edge of the bounding boxes; normalized coordinates. |
| image/object/bbox/xmax | a float array, the right edge of the bounding boxes; normalized coordinates. |
| image/object/bbox/ymin | a float array, the top left corner of the bounding boxes; normalized coordinates. |
| image/object/bbox/ymax | a float array, the top edge of the bounding boxes; normalized coordinates. |
| image/object/bbox/score | a float array, the score for the bounding box. For example, the confidence of a detector. |
| image/object/bbox/label | an integer array, specifying the index in a classification layer. The label ranges from [0, num_labels) |
| image/object/bbox/text | an array of strings, specifying the human readable label for the bounding box. |
| image/object/bbox/conf | a float array, the confidence of the label for the bounding box. For example, a probability output from a classifier. |
| image/object/parts/x | a float array of x locations for a part; normalized coordinates. |
| image/object/parts/y | a float array of y locations for a part; normalized coordinates. |
| image/object/parts/v | an integer array of visibility flags for the parts. 0 indicates the part is not visible (e.g. out of the image plane). 1 indicates the part is occluded. 2 indicates the part is visible. |
| image/object/parts/score | a float array of scores for the parts. For example, the confidence of a keypoint localizer. |

Take note:

* Many of the above fields can be empty. Most of the different systems using the tfrecords will only need a subset of the fields. 

* The bounding box coordinates, part coordinates and areas need to be *normalized*. For the bounding boxes and parts this means that the x values have been divided by the width of the image, and the y values have been divided by the height of the image. This ensures that the pixel location can be recovered on any (aspect-perserved) resized version of the original image. The areas are normalized by they area of the image. 

* The origin of an image is the top left. All pixel locations will be interpreted with respect to that origin. 

The [create_tfrecords.py](create_tfrecords.py) file has a convience function for generating the tfrecord files. You will need to preprocess your dataset and get it into a python list of dicts. Each dict represents an image and should have a structure that mimics the tfrecord structure above. However, slashes are replaced by nested dictionaries, and the outermost image dictionary is implied. Here is an example of a valid dictionary structure for one image:

```python
image_data = {
  "filename" : "/path/to/image_1.jpg", 
  "id" : "0",
  "class" : {
    "label" : 1,
    "text" : "Indigo Bunting",
    "conf" : 0.9
  },
  "object" : {
    "count" : 1,
    "area" : [.49],
    "id" : ["1"],
    "bbox" : {
      "xmin" : [0.1],
      "xmax" : [0.8],
      "ymin" : [0.2],
      "ymax" : [0.9],
      "label" : [1],
      "score" : [0.8],
      "conf" : [0.9]
    },
    "parts" : {
      "x" : [0.2, 0.5],
      "y" : [0.3, 0.6],
      "v" : [2, 1],
      "score" : [1.0, 1.0]
    }
  }
}
```

Not all of the fields are required. For example, if you just want to train a classifier using the whole image as an input, then your dictionaries could look like:
```python
image_data = {
  "filename" : "/path/to/image_1.jpg", 
  "id" : "0",
  "class" : {
    "label" : "1"
  }
}
```

If the `encoded` key is not provided. Then the `create` method will read in the image by using the `filename` value. In this case, it is assumed that image is stored in either jpg or png format. If `encoded` is provided, then it is required to provide `height`, `width`, `format`, `colorspace`, and `channels` as well. 

Once you have your dataset preprocessed, you can use the `create method` in [create_tfrecords.py](create_tfrecords.py) to create the tfrecords files. For example:

```python
# this should be your array of image data dictionaries. 
# Don't forget that you'll want to separate your training and testing data.
train_dataset = [...]

from create_tfrecords import create
failed_images = create(
  dataset=train_dataset,
  dataset_name="train",
  output_directory="/home/gvanhorn/Desktop/train_dataset",
  num_shards=10,
  num_threads=5
)
```

This call to the `create` method will use 5 threads to produce 10 tfrecord files, each prefixed with the name `train` in the directory `/home/gvanhorn/Desktop/train_dataset`. 

All images that cause errors will be returned to the caller. An extra field, `error_msg`, will be added to the dictionary for that image, and will contain the error message that was thrown when trying to process it. Typically an error is due to `filename` fields that don't exist. 

```python
print("%d images failed." % (len(failed_images),))
for image_data in failed_images:
  print("Image %s: %s" % (image_data['id'], image_data['error_msg']))
```
