# Front End Project -- A Journey

Here, we document our journey!


## Table of Contents
1. [Learning Journey](#learning-journey)
2. [Structure](#structure)
    1. [Algorithm Module](#algorithm-module)
    2. [Source Module](#source-module)
    3. [IO Module](#io-module)
    4. [Modularization](#modularization)
3. [Edgybees Deployment Notes](#edgybees-deployment-notes)
4. [Icarus Deployment Notes](#icarus-deployment-notes)
5. [Future Development](#future-development)



## Learning Journey
Jan 6th
1. Git commands
   ```bash
   # Clone the repository
   git clone https://github.com/AaronTanKY/riftCVProgress.git
   
   # Add, commit, push changes
   git add . 
   git commit -m "Message here"
   git push origin main

   # Pull changes
   git pull
   ```

2. CD into the directory "RiftCVProgress"

3. Pull any changes
   ```bash
   git pull
   ```

4. Checkout into the branch you want to see (main is empty) [the names are prety self-explanatory]
    ```bash
    # master contains a combination of both IcarusWindows and EdgybeesDeployment
    git checkout master

    # IcarusWindows is a version of master that works for multiple rtsp inputs and multple flask outputs ONLY for the feed without object detection (because of some PyTorch issue on windows)
    git checkout IcarusWindows

    # EdgybeesDeployment is a version of master that only works for single UDP input and single UDP output, replaces the flask server in the IO-module with a gstreamer pipeline so that the it immediately outputs without the need to type in the flask url anywhere
    git checkout EdgybeesDeployment
    ```

5. Now, since the project isn't dockerized (to the next intern, do that if you have the time HAHA), you will have to create your own virtual environments for the modules. BUT FIRST, let's take a look at the structure of the repository. I will be looking at the master branch.

## Structure
```bash
RiftCVProgress
├── cmd_line
├── README.md
├── Algorithm-Module
│   ├── objDetect_LatentAI
│   ├── objDetect_YOLOv5
│   └── RIFT FTP
├── IO-Module
│   ├── EdgyBees_Gstreamer
│   ├── Flask_MultiInput
│   ├── Flask_SingleInput
│   └── protobuf_stash
└── Source-Module
    ├── Source_Multi
    └── Source_Single
```

### Algorithm Module

Right now, the Algo Module suppots __LatentAI__ and __YOLOv5__. The codes to run the following files can be found in the src folders, under \_\_init\_\_.py. 

### IO Module

The most complete IO Module is in the folder __Flask_MultiInput__, and __EdgyBees_Gstreamer__ contains only the Gstreamer portion of the code meant for Edgybees computer deployment, while __Flask_SingleInput__ is a simplified version, as it only supports one input and one output without multiprocessing. Technically, you can run everything from __Flask_MultiInput__. Once again, the codes to run the following files can be found in the src folders, under \_\_init\_\_.py. 

### Source Module

The Source module supports both single (__Source_Single__) and multi-input (__Source_Multi__). Again, technically, the single source module can be replaced by the multi source module, but they are not entirely compatible just get (because of different gRPCs, etc.) You can make them all into one file if you wanted to! Once again, the codes to run the following files can be found in the src folders, under \_\_init\_\_.py. 

### Modularization

In each of these folders (e.g. __objDetect_LatentAI__), you have to create a virtual environment (that's the whole point of modularization in the first place!) To do that, just quickly search up how to create and source virtual environments, and do that. The file name for your virtual environments should be __venv__, because that is what is being git ignored, which means it won't be pushed onto github and take up space. There is a requirements.txt file, which you can just do 

```bash
pip install -r requirements.txt
```

in order to install all of the dependencies required. If something can't be installed, honestly just run the \_\_init__file and see what you missed and install them.

## Edgybees Deployment Notes

This is mainly for people in Wallaby right now as they are reading this trying to figure out what is going on. And there are a few things to take note of

### Facing startup problems with services?

1. Check if everything is up and running
```
sudo systemctl status riftcv_algo.service riftcv_source.service riftcv_io.service
```
2. If they are __status: active__ but you can't seem to get display on Artemis, check the io service and you'll know if something is off. 
```
sudo systemctl status riftcv_io.service
```
3. If you see the following below, everything is going fine. But if you don't see any retrieving of frames of metadata, it means the gstreamer is not outputting, and you have to restart the service
```
Added frame to queue, queue size:
Added metadata to queue, queue size:
Retrieved frame from queue, queue size:
Retrieved metadata from queue, queue size:
```
### Restarting the scripts
1. First step is simple, just do a
```
sudo systemctl restart riftcv_algo.service riftcv_source.service riftcv_io.service
```
2. However, I find that this works better:
```bash
sudo systemctl stop riftcv_algo.service riftcv_source.service riftcv_io.service
## WAIT 5-10 SECONDS ##
sudo systemctl start riftcv_algo.service riftcv_source.service riftcv_io.service
```
3. But so far, what works FOR SURE is this. Open up 3 terminals in your __home__ directory and then do this:
```bash
# Terminal 1: Run LatentAI object detection!
# Step 1
source RiftCVProgress/Algorithm-Module/objDetect_LatentAI/venv/bin/cuda_activate
# Step 2
TVM_TENSORRT_CACHE_DIR=RiftCVProgress/Algorithm-Module/objDetect_LatentAI/src/algo/detection/latentai/trt-cache python3 RiftCVProgress/Algorithm-Module/objDetect_LatentAI/src/algo/detection/latentai/__init__.py
```
```bash
# Terminal 2: Run camera input module!
# Step 1
source RiftCVProgress/Source-Module/Source_Single/venv/bin/activate
# Step 2
python3 RiftCVProgress/Source-Module/Source_Single/src/source/single/main_server.py
```
```bash
# Terminal 3: Run edgybees flask server (wait for 5 seconds before starting!)
# Step 1
source RiftCVProgress/IO-Module/EdgyBees_Gstreamer/venv/bin/activate
# Step 2
python3 RiftCVProgress/IO-Module/EdgyBees_Gstreamer/src/io/edgybees/__init__.py
```
After all this, you should see that you get everything going fine __HOPEFULLY__!!!

### Need to change IP addresses?
Sorry! But for now, IP addresses are hard coded. Each of the points below are different cases. So scroll to your use case!
1. If you want to change MQTT IPs or GRPC ports, change the __CONFIG.yml__ files in each of the necessary folders

2. Change FTP IP (For Zheng Wei): line 223 in RiftCVProgress/Algorithm-Module/objDetect_LatentAI/src/algo/detection/latentai/_\_init__.py

3. Change input Source IP from EdgyBees (For Edwin): line 71, under Cam() initialization, in RiftCVProgress/Source-Module/Source_Single/src/source/single/main_server.py

4. Input RTSP and not UDP (directly from ridgerun): Under class Cam(AbstractCam), under super init, follow the comment out and uncommenting instructions!

5. Change output IP from RiftCV (For Jun Qi Artemis): line 126, under udpsink host, in RiftCVProgress/IO-Module/EdgyBees_Gstreamer/src/io/edgybees/_\_init__.py

## Icarus Deployment Notes
### Accessing streams!
Everything should be auto startup. There should be 2 command line windows open, one saying that flask server is activated, one saying that gRPC is hosted at port 50050.

Open web browser and type
```
127.0.0.1:50049/feed/<insert_droneip_here>
```
Right now this doesn't support object detection because of issues with PyTorch...

### Debugging
1. Source Module:
```
RiftCVProgress/IO-Module/Flask_MultiInput/src/io/multi/_\_init__.py
```

2. IO Module
```
RiftCVProgress/Source-Module/Source_Multi/src/source/multi/main_server2.py
```
## Future Development
These are in no order of importance

1. Make all the IPs dynamic (not hard coded)

2. Combine all the files in Source Module

3. Combine all the files in IO Module 

4. Have Source Module accept RTSP streams

5. Have Algo Module have an abstract base class like MPCam in Source Module

6. Dockerize everything