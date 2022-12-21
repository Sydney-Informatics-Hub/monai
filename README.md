# MonAI and 3D Slicer on Ronin
Workflow to run [MonAI](https://monai.io/) label from [3D Slicer](https://www.slicer.org/) on a [Ronin](https://ronin.sydneyuni.cloud/) virtual machine (which uses AWS in the backend).

The below instructions are for Linux. Windows may be okay too, but also requires installation of NVIDIA etc. If you are happy to use the preconfigured machine, jump straight to [3.Running the MonAI server](#3-run-monai-server)

## 1. Create a machine

Follow instructions here: https://sydneyuni.atlassian.net/wiki/spaces/RC/pages/1156153809/Machine+Management

You can use the "monai" "project package" as a base image and skip Step 2 below.

The most "powerful" GPU on Ronin as of Dec 2022 is a *P3.2XLARGE*, it has a single V100 card and costs **USD$4.23 per hour**. 
For "Demo" purposes using a *G4DN.XLARGE* may be the way to go as it costs **USD$0.68 per hour** for a single T4 GPU.

I am unaware if MonAI can utilise multiple GPU cards. 


## 2. Set up the base machine with GPU drivers and Docker

### Connect to your machine
```
ssh -i ~/.ssh/monaikey.pem ubuntu@monai-base.sydneyuni.cloud
```

### Install ubuntu drivers
```
sudo apt-get update
sudo apt-get -y install build-essential
sudo apt-get -y install linux-headers-$(uname -r)
sudo apt install -y ubuntu-drivers-common
# I had some error with "aplay" with the next ste[, I think I used a differnt VM and it worked.
sudo ubuntu-drivers autoinstall
sudo reboot
```

### After rebooting, re-connect to your machine
```
ssh -i ~/.ssh/monaikey.pem ubuntu@monai-base.sydneyuni.cloud
```

### Install Cuda
```
sudo apt install -y nvidia-cuda-toolkit
```

### Install Docker
```
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh ./get-docker.sh
```

### Install Nvidia docker
```
distribution=$(. /etc/os-release;echo $ID$VERSION_ID) \
      && curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
      && curl -s -L https://nvidia.github.io/libnvidia-container/$distribution/libnvidia-container.list | \
            sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
            sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list


sudo apt-get update
sudo apt-get install -y nvidia-docker2
sudo systemctl restart docker
```

## 3. Run MonAI server

### Connect to machine. 
```
ssh -i ~/.ssh/monaikey.pem ubuntu@monai-base.sydneyuni.cloud
```

### Start the monai server, e.g. 
```
sudo docker run -it --rm --gpus all --ipc=host --net=host -p 8000:8000 --name monai -v ~:/workspace/ projectmonai/monailabel:latest bash
monailabel apps --download --name radiology --output apps
monailabel datasets --download --name Task09_Spleen --output datasets
monailabel start_server --app apps/radiology --studies datasets/Task09_Spleen/imagesTr --conf models deepedit
```
This step will vary depending on you want to do (I think). With this setup the `-v ~:/workspace/` option, mounts all the data in the home folder of the VM into the docker container, presumably making it accessible by MonAI. Put data in that folder, or mount your data approriately.

MonAI Label will create a server at `http://0.0.0.0:8000` by default. Hence the port-fowarding, but I think the "host" flags in the other docker options make it available at 8000 on the host anyway. Regardless, this is where Slicer will have to look for a MonAI server.

Leave this terminal window open. You can make this "headless" and persistent as your needs vary, so you can turn things off, etc.

## 4. Run 3D Slicer

### Connect to machine
Connect with ssh and forward the port we will map for interacting with Slicer.
```
ssh -i ~/.ssh/monaikey.pem -L 8080:localhost:8080 ubuntu@monai-base.sydneyuni.cloud 
```

### Launch Slicer
```
sudo docker run -it --rm --gpus all --ipc=host --net=host -d -p 8080:8080 --name slicer stevepieper/slicer:5.0.3
```

As with MonAI, Slicer needs to "escape" from the Docker container and access the MonAI Docker host, hence the host and port flags (although I suspect there is some redundancy here).

Now wait a couple minutes (but no more than 2) and then on your local web-browser navigate to `http://localhost:8080/` in a browser, start an x11 session and go from there:
> View → Extension Manager → Search MONAI Label
> 
> Install MONAI Label plugin
> 
> Restart 3D Slicer
> 
> Then from the main screen search for MonaAI Label and click the Green Circle refresh button.

Note this particular Docker image is not the offical one, but the only one that was clear, it seems to be regularly updated, albeit this is a few versions behind the current. I tried some approaches not using Docker, but this was by far the easiest install process. 

# Extra notes.
To run general monai, use:
```sudo docker run --gpus all --rm -ti --ipc=host projectmonai/monai:latest``` But I am not sure how you use that.

You may need X11 forwarding -X (you must setup x11 locally with [xquartz](https://www.xquartz.org/) for example)
