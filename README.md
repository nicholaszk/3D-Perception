# 3D-Perception
Using RGB-D cameras to collect data, point clouds (PCs) were created, segmented, and filtered against known object PCs to recognize and sort objects in a scene  
  
## Project: Perception Pick & Place

---


# Required Steps for a Passing Submission:
1. Extract features and train an SVM model on new objects (see `pick_list_*.yaml` in `/pr2_robot/config/` for the list of models you'll be trying to identify). 
2. Write a ROS node and subscribe to `/pr2/world/points` topic. This topic contains noisy point cloud data that you must work with.
3. Use filtering and RANSAC plane fitting to isolate the objects of interest from the rest of the scene.
4. Apply Euclidean clustering to create separate clusters for individual items.
5. Perform object recognition on these objects and assign them labels (markers in RViz).
6. Calculate the centroid (average in x, y and z) of the set of points belonging to that each object.
7. Create ROS messages containing the details of each object (name, pick_pose, etc.) and write these messages out to `.yaml` files, one for each of the 3 scenarios (`test1-3.world` in `/pr2_robot/worlds/`).  [See the example `output.yaml` for details on what the output should look like.](https://github.com/udacity/RoboND-Perception-Project/blob/master/pr2_robot/config/output.yaml)  
8. Submit a link to your GitHub repo for the project or the Python code for your perception pipeline and your output `.yaml` files (3 `.yaml` files, one for each test world).  You must have correctly identified 100% of objects from `pick_list_1.yaml` for `test1.world`, 80% of items from `pick_list_2.yaml` for `test2.world` and 75% of items from `pick_list_3.yaml` in `test3.world`.
9. Congratulations!  Your Done!


## [Rubric](https://review.udacity.com/#!/rubrics/1067/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
  
[//]: # (Image References)

[image1]: ./perceptionPics/trainSvmScore.JPG
[image2]: ./perceptionPics/normalizedConfusionMatrix.JPG
[image3]: ./perceptionPics/nonNormalConfusionMatrix.JPG
[image4]: ./perceptionPics/completePerception.JPG
  
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

### Exercise 1, 2 and 3 pipeline implemented
#### Pipeline for filtering and RANSAC plane fitting implemented.  
  
##### 1. Voxel Grid Downsampling 
A voxel grid filter takes a spatial average of the cloud points within each voxel. Setting a leaf size sets the size of each vocel that you want to average out. If I specify a leaf size of 0.5, for example, the downsampling process would give me one pixel that represents the average point of all pixels in a 0.5m x 0.5m x 0.5m cube. Obviously 0.5m is huge, so I tested a range of 0.004m - 0.006m.
  
##### 2. Statistical Outlier Filtering
This process assumes crude planes based on where a majority of local cloud points are located relevant to each other. Then outlier points are discarded. This process helps eliminate noise in a point cloud due to non-perfect hardware and other environmental influences.

##### 3. PassThrough filter
An axis is chosen, and minimum and maximum values along that axis are chosen to isolate a region of interest containing only the points of interest (table and the objects on the table). All other points are discarded. In this case, I performed PassThrough on two axes as follows:  
* Y-axis: axis_min = -0.5,   axis_max = 0.5  
* Z-axis: axis_min =  0.6,   axis_max = 1.1   

##### 4.RANSAC Plane Fitting
[Random Sample consensus](https://en.wikipedia.org/wiki/Random_sample_consensus) is used to remove the table itself from the scene. The RANSAC algorithm assumes that all data in a dataset is divided between inliers and outliers. Inliers can be defined by a model with a specific set of parameters while outliers cannot and should be discarded.

##### 5.Extracting Indices - Inliers and Outliers
Once we have determined where the table is located, we can separate point clouds: 
```
#Table
extracted_inliers = cloud_filtered.extract(inliers, negative=False) 
#Objects
extracted_outliers = cloud_filtered.extract(inliers, negative=True) 
```
  

#### Pipeline including clustering for segmentation implemented.  

##### 1. Euclidean Clustering
I used a PCL library function called EuclideanClusterExtraction() to perform a DBSCAN cluster search through the "Objects" 3D point cloud (extracted_outliers).
 
DBSCAN stands for Density-Based Spatial Clustering of Applications with Noise. The algorithm creates clusters by grouping data points that are within some threshold distance `d` from the nearest point in the data. There is also a user-specified minimum and maximum cluster size. If the minimum is too low, we may see some of our objects split into two or more clusters. If the maximum is too small, we will see some object cloud points that are not included in their cluster. I chose the following criteria:  

```
    ec.set_ClusterTolerance(0.01)
    ec.set_MinClusterSize(50)
    ec.set_MaxClusterSize(50000) # I'm really just setting the maximum to NONE by specifying a large number 
```

##### 2. Create Cluster-Mask Point Cloud to visualize each cluster separately
Here I simply applied a unique color to each of the unique objects to aid with cluster visualization.  

#### Features extracted and SVM trained.  Object recognition implemented.
  
##### Extract features
I created histograms of the objects' colors using HSV instead of RGB. I also created histograms of the objects' normals. Both histograms were normalized and concatenated together. Based on my experimenting in the exercises, I found 60 - 80 histogram bins to be optimal for accuracy. Beyond 80 histogram bins, accuracy actually decreased.

```
# Extract histogram features
        chists = compute_color_histograms(ros_cluster, using_hsv=True)
        normals = get_normals(ros_cluster)
        nhists = compute_normal_histograms(normals)
        # Compute the associated feature vector
        feature = np.concatenate((chists, nhists))
```

##### Train data with SVM.
The script capture_features.py was used to obtain multiple captures of the following objects:
```
models = [ \
        'biscuits',
        'soap',
        'soap2',
        'book',
        'glue',
        'sticky_notes',
        'eraser',
        'snacks']
```
Then we used train_svm.py to train our support vector machine by creating profiles off each object's multiple captures worth of histograms. For each object, I required 60 captures. My SVM achieved an accuracy of 90%, but if I had taken 200+ captures of each object, I expect I could have achieved a rating of 95%+. Here are the results in a confusion matrix:  
  
![alt text][image1]
![alt text][image2]
![alt text][image3]
    

  
### Pick and Place Setup
  

#### 1. For all three tabletop setups (`test*.world`), perform object recognition, then read in respective pick list (`pick_list_*.yaml`). Next construct the messages that would comprise a valid `PickPlace` request output them to `.yaml` format.
  
![alt text][image4]
  
 
I used a bin size of 60 for both my HSV color histograms as well as my normals histograms. I took 60 captures of each of the 8 items that could potentially be on the table. As I mentioned before, with more captures my accuracy would get even higher. I mentioned taking 200+ captures, and I think with enough time I would strive to really get about 1000 captures each.As for the bin size, I think I would increase that number to about 80. With more time spent optimizing these two parameters (even if it took hours and multiple trials, I think the accuracy could get extremely reliable.


