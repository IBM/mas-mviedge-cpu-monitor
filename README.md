# Install MVI Edge on Ubuntu VMWare (Mac) - Run in CPU mode - Integrate with MAS Monitor - Find Gold !
**Author:** Christophe Lucas **Last Updated:** 22 April 2024 <br>
**Disclaimer:** This tutorial is delivered as-is and is NOT formal IBM product documentation in any way
## Table of Contents
- [Introduction](#intro)
- [Prerequisites](#prereq)
- [Install Ubuntu on VMWare on Mac](#ubuntu)
    - [Download & Install Ubuntu](#iso)
    - [Update VMWare Settings ](#vm)
- [Install MVI Edge](#mvie)
    - [Install Docker](#docker)
    - [Install MVI Edge](#mviei)
        - [Get (free) IBM Cloud Entitlement Key with your IBM ID](#key)
        - [Login to docker and run vision-edge-inception](#dockergo)
        - [Enable CPU Mode in vision-edge.properties](#cpu)
        - [Run startedge.sh for first time](#runedge)
        - [Assign MVI Edge Unique Prefix](#prefix)
- [Integrate MVI Edge with MVI & Monitor](#mviemvimonitor)
    - [Setup MVI Edge to MVI Connection](#mviemvi)
    - [Setup MVI Edge to Monitor Connection](#mviemonitor)
- [Configure MVI Edge & Monitor for automated inspections](#config)
    - [Download Rocks MVI Model & Data Set, Create Input Source, Station & Inspection](#deploy)
    - [Configure Inspection, Run](#createmvie)
- [Create data items & Dashboard in Monitor - Find Gold !](#monitordata)
    - [Create PythonExpression data items](#python)
    - [Create KMeans Anomaly function on Gold](#kmeans)
    - [Create Inspection Dashboard and Hourly Dashboard](#dash)
    - [Optional - Make a cool Gold Anomaly Detection Dashboard](#optional)


<a id='intro'></a>
# Introduction 

In this tutorial, you will learn how to:
- install Maximo Visual Inspection Edge (aka MVI Edge) on a Ubuntu VMWare on your Mac
- setup MVI Edge to run in CPU only mode
- integrate MVI Edge with a MAS MVI instance and a MAS Monitor instance
- download and deploy a provided MVI Rock-recognizing TinyYOLO model to your MVI Edge
- setup MVI Edge station and inspections
- run MVI Edge inspections using the provided Rocks_Samples images data set and send inspection results to Monitor
- create data items in Monitor and dashboards to count the amount of Gold found during the inspections

**Expected Tutorial Duration:** 1st time = 3 hours, 2nd time = 27 minutes

Acronyms used & Product Documentation links: **MVI** = <a href="https://www.ibm.com/docs/en/maximo-vi/continuous-delivery" target="_blank">IBM Maximo Visual Inspection</a>, **MVI Edge** = <a href="https://www.ibm.com/docs/en/maximo-vi/continuous-delivery?topic=integrating-maximo-visual-inspection-edge" target="_blank">IBM Maximo Visual Inspection Edge</a>, **Monitor** = <a href="https://www.ibm.com/docs/en/maximo-monitor/continuous-delivery" target="_blank">IBM Maximo Monitor</a>, **MAS** = <a href="https://www.ibm.com/docs/en/maximo-monitor/continuous-delivery" target="_blank">IBM Maximo Application Suite</a>. 

One can find a story of how this tutorial logic was first used in this article <a href="https://medium.com/@xophelucas/how-we-built-an-ai-driven-ewaste-visual-inspection-demo-machine-4ed0b4442c77" target="_blank">How we built an AI-driven eWaste Visual Inspection Demo Machine</a>.<br>

<a id='prereq'></a>
# Prerequisites
To complete this tutorial, you will need:
- VMWare Fusion or similar software to create a Ubuntu image
- an IBM ID
- access to an instance of MAS with Administrator access to both MVI and Monitor MAS Applications
- access to the OpenShift Container Platform (OCP) of your MAS instance (to get Monitor's Analytics Service API key & token)
- either an (older) Mac runnning on 'Intel Processor' OR a (newer) Mac running on 'Apple silicon' with VMWare Fusion 13 minimum (see <a href="https://kb.vmware.com/s/article/2088571" target="_blank">Supported host operating systems for VMware Fusion and VMware Fusion Pro (2088571)</a>).

This tutorial was built using:
- VMWare Fusion 12.2.3 on a Mac (macOS Ventura 13.5.1, Radeon Pro 560X 4 GB Intel UHD Graphics 630 1536 MB)
- Ubuntu 20.04.6 LTS (Focal Fossa)
- Maximo Visual Inspection (aka MVI, 8.8.1) + Maximo Monitor (aka Monitor, 8.10.4) running on Maximo Application Suite (aka MAS 8.10.4) + Maximo Visual Edge (aka MVIE, 8.8)

**21 January 2024 Update**: This tutorial was run with a later Ubuntu version (22.04.3), and the latest MVI Edge (8.8.1). Except for some minor UI changes, the same steps still work.

<a id='ubuntu'></a>
# Install Ubuntu on VMWare on Mac
The following image highlights the main screens that you will encounter while executing this section:
![image](/images/MVIE1.jpg)

<a id='iso'></a>
## Download & Install Ubuntu
Let's first download and install Ubuntu in a VMWare:
1. Go to Ubuntu's official release page <a href="https://releases.ubuntu.com/20.04/" target="_blank">Ubuntu 20.04.6 LTS (Focal Fossa)</a> and download the <a href="https://releases.ubuntu.com/20.04/ubuntu-20.04.6-desktop-amd64.iso" target="_blank">ubuntu-20.04.6-desktop-amd64.iso</a> file.
2. Launch VMWare Fusion and drag and drop the `.iso` file you just downloaded on the `Install from disc or image`. Click `Continue`.
3. Enter a password for your VMWare. Click `Continue`. 
4. Click `Customize Settings` and change `Disk Size` to e.g. `40 GB` - you just want to be on the safe side in case you plan to collect or inspect many and/or heavy images. Click `Finish`. It will take up to 5 minutes for Ubuntu to be installed within the VMWare - you will see the progress on the screen. Final screen should be the Ubuntu login page as per bottom-right screen in image above.
5. In your `Virtual Machine` - `Network Adapter` menu, select `Bridge (Auto Detect)`. When you launch your VMWare, it should automatically connect to the Wifi your Mac is connected to.

<a id='vm'></a>
## Update VMWare Settings
For future usage comfort, let's update a couple of VMWare settings. From the desktop of your Ubuntu VMWare, right-click and select`Settings`:
1. In the `Date & Time` menu, select your local `Time zone`.
2. In the `Displays` menu, set a comfortable `Resolution` e.g. `(1024 x 768)`.
3. In the `Privacy - Screen Lock` choose your`Automatic Screen Lock` option.
4. Check that you can copy-text/drag-files (from your Mac) and paste-text/drop-files (into a Finder in your Ubuntu VMWare). If not, open a Terminal in your VMWare and run `sudo apt install open-vm-tools-desktop` to install VM tools (and restart the VMWare).

<a id='mvie'></a>
# Install MVI Edge
The following image highlights the main screens that you will encounter while executing this section:
![image](/images/MVIE2.jpg)
<a id='docker'></a>
## Install Docker
We are following the <a href="https://www.ibm.com/docs/en/maximo-vi/continuous-delivery?topic=planning-installing-docker-nvidia-docker2#installing-sitedatakeyworddocker-and-nvidia-docker2__section_p4b_qdm_bvb__title__1" target="_blank">Installing Docker® and nvidia-docker2 Procedure - Ubuntu</a> section of the MVI Documentation (select `On x86_64` sub section).<br><br>
Open a Terminal on your VMWare and from your home folder:

1. Run `sudo apt-get update`.
2. Run `sudo apt-get install apt-transport-https ca-certificates curl gnupg-agent software-properties-common`.
3. Run `curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -`.
4. Run `sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"`.
5. Run `sudo apt-get update`.
6. Run `sudo apt-get install docker-ce`.

**NOTE**: Because we will be using MVI Edge in CPU-only mode, it is NOT required to install `nvidia-docker2` as per section 2 of the <a href="https://www.ibm.com/docs/en/maximo-vi/continuous-delivery?topic=planning-installing-docker-nvidia-docker2#installing-sitedatakeyworddocker-and-nvidia-docker2__section_p4b_qdm_bvb__title__1" target="_blank">Installing Docker® and nvidia-docker2 Procedure - Ubuntu</a> documentation.


<a id='mviei'></a>
## Install MVI Edge
This section is a concatenated mix of the <a href="https://www.ibm.com/docs/en/maximo-vi/continuous-delivery?topic=edge-installing-uninstalling#installing-and-uninstalling__installing__title__1" target="_blank">Installing and uninstalling MVI Edge</a> documentation and <a href="https://github.com/IBM/vision-tools/blob/dev/edge/docs/installation/inception_internals.md" target="_blank">IBM Maximo Visual Inspection Edge Inception Internals</a> information.

<a id='key'></a>
### Get (free) IBM Cloud Entitlement Key with your IBM ID
Login to this <a href="https://myibm.ibm.com/products-services/containerlibrary" target="_blank">IBM Cloud Entitlement Key</a> using your IBM ID.<br>
An `Entitlement keys(1)` page will appear. On the bottom right, just click `Copy` the key. Paste and save it locally in e.g. a Notebook for further use. We'll call it `YOUR_ENTITLEMENT_KEY` in future sections and it should look like a long string e.g. `JJQk0gTWFya2V0blablablakNWU0Y2`. 

<a id='dockergo'></a>
### Login to docker and run vision-edge-inception
Open a Terminal in your Ubuntu VMWare and from your home directory:
1. Run `mkdir MVIE88`, then `cd MVIE88/`, then `mkdir vision-edge` then `cd  vision-edge/`.
2. From your `/MVIE88/vision-edge/` directory, run `sudo docker login cp.icr.io --username cp --password YOUR_ENTITLEMENT_KEY`. You should get a `Login Succeeded` at the end of the output. <br> 
3. Run the MVI Edge inception process, i.e. run:
```
sudo docker run --rm -v `pwd`:/opt/ibm/vision-edge -e hostname=`hostname -f` --privileged -u root cp.icr.io/cp/visualinspection/vision-edge-inception:8.8.0
```

Observe how that command created the following file structure:
```
MVIE88/
├── vision-edge/
│   ├── startedge.sh
│   ├── restartedge.sh
│   ├── stopedge.sh
│   ├── volume/
│   │   ├── bin
│   │   ├── data
│   │   ├── run
```

<a id='cpu'></a>
### Enable CPU Mode in vision-edge.properties
Under the `/MVIE88/vision-edge/volume/run/var/config` folder, edit the `vision-edge.properties` file to set MVI Edge in CPU-only mode.
1. Run `sudo gedit vision-edge.properties&` (or `sudo nano vision-edge.properties` if gedit not installed). That will open the file.
2. Locate the `DLE_ENABLE_CPU_FALLBACK=FALSE` line and replace it by `DLE_ENABLE_CPU_FALLBACK=TRUE`. Save.

<a id='runedge'></a>
### Run startedge.sh for first time
Under the `/MVIE88/vision-edge` folder, let's start MVI Edge for the first time: 
1. Run `sudo ./startedge.sh`. Write `YES` on License Agreement line (get there quickly by typing `q`). This can take up to 15 minutes depending on your internet download speed.
2. Notice how a new `monitor` directory appeared under `/MVIE88/vision-edge/volume/run/var/config/` after having run `startedge.sh` for the first time.

**IMPORTANT NOTE**: Do not forget to take note of the password for default user `masadmin` at the end of the output as it is the ONLY time you will see it.
```
******************************************************************
******************************************************************
-  The default username and password are:
   -  username: masadmin
   -  password: vp7KE^blabla(K2sf%AW
******************************************************************
******************************************************************
******************************************************************
**                                                              **
**  PLEASE SAVE THE PASSWORD - IT WILL NOT BE DISPLAYED AGAIN.  **
**                                                              **
******************************************************************
******************************************************************
Access the console at https://ubuntu
```
3. Launch https://ubuntu in Firefox and check the MVI Edge login page appears. Do not login yet.

<a id='prefix'></a>
### Assign MVI Edge Unique Prefix in /volume/run/var/config/monitor/monitor.json
Change the default `node_prefix` (that will be the root name of Monitor Device Types), i.e.:  
1. Under the `/MVIE88/vision-edge/volume/run/var/config/monitor` folder, Run `$sudo gedit monitor.json` to open the file. 
2. Find the line containing `"node_prefix": "MVI"` and change it to e.g. `"node_prefix": "MVIE88A_"`. Note that because the value of `node_prefix` is what will appear in front of all Device Types you will later create in Monitor, do NOT choose too-long a name (+ using an underscore `_` at the end is a good idea).
3. Under `/MVIE88/vision-edge` folder, run `sudo ./restartedge.sh`

<a id='mviemvimonitor'></a>
# Integrate MVI Edge with MVI & Monitor
The following image highlights the main screens that you will encounter while executing this section:
![image](/images/MVIE3.jpg)

<a id='mviemvi'></a>
## Setup MVI Edge to MVI Connection
First get your MVI Server endpoint and API Key:
1. On MVI Server, click the left `Services - API key` menu.
2. Copy both the values of `API key` and `API end point` and save locally.

Then, setup in MVI Edge:
1. Login for the first time with ID `masadmin` to `https://ubuntu`.
2. On the `Welcome to Maximo Visual Inspection` page, click `Next`. On the `Maximo Visual Inspection Settings` page, enter the `API key` value you just copied. For the `URL` value, enter the `API end point` value without the `/api` at the end - note that URL is also the URL of your MVI Server homepage - for example `https://mygeo.visualinspection.mvimas.gtm-pat.com`. Click `Save`, wait a second and you should see a green indicator appear next to the `API key` field.

<a id='mviemonitor'></a>
## Setup MVI Edge to Monitor Connection
First you will need the API Key & Token values of both the `Platform Service` and `Analytics Service` of your Monitor instance, as per <a href="https://www.ibm.com/docs/en/maximo-monitor/continuous-delivery?topic=reference-apis" target="_blank">Monitor APIs</a> instructions.
This requires access to the OpenShift Container Platform (OCP) of your MAS instance.<br>

In summary, to get the API Key & Token of the `Analytics Service`, do:
1. From the OCP console, click `Project`. In the `Search by name` field, start typing `mas-monitor` and locate the 1 e.g. `mas-masgeo-monitor` Project. Open the Project.
2. On the `Inventory` card, click `Secrets`. Search for and open `monitor-api`.
3. Copy and save `as_apikey` and `as_token` values.

In summary, to get the API Key & Token of the `Platform Service`, do:
1. From Monitor's home page, click `Open the Iot Tool`. This will open Monitor's Watson IoT Platform.
2. In the WIoTP tool, click `Apps` left menu. Click `Generate API key` top right button. Enter a description and set `API Key Expires` to `OFF` then click Next.
From the Role list, select `Backend Trusted Application`, then click `Generate Key` button. 
3. Copy and save the API key and authentication token that are displayed.<br>

Finally, let's now copy paste those values in MVI Edge:
1. Click MVI Edge `Settings - Maximo Monitor` menu.
2. On he `Monitor Dashboard URL` field, enter the homepage URL of your Monitor instance e.g. `https://mygeo.monitor.mygeomas.gtm-pat.com/`.
3. On the `Platform Service` and `Analytics Service` sections, enter the values you saved at start of this section. Click `Create Connection`.
4. Check that the connection has been well established by verifying that a `Generic Device Type` has been created on the `Connection` tab.

**NOTE** Notice that the `Generic Device Type` that we just created is also accessible in the Monitor application where, via the `Monitor` menu, you will now see a new `MVIE88A_Generic_Type`. THAT is the integration point between MVI Edge and Monitor.

<a id='config'></a>
# Configure MVI Edge & Monitor for automated inspections

<a id='deploy'></a>
## Download Rocks MVI Model & Data Set, Create Input Source, Station & Inspection
In this section, we will download and deploy a TinyYOLO model that was trained on an MVI server to recognise a set of 9 various rocks moving on a conveyor belt (`Calcite`, `Fluorite`, `Lepidolite`, `MilkyQuartz`, `RoseQuartz`, `Obsidian`, `Glass`, `RedJasper` and 1 special rock from Kalgoorlie in Western Australia which contains `Gold`). We will then download a sample Data Set containing 31 images of those rocks, create an Input Source using it, and import a ready-to-use Inspection.
1. Download the following <a href="https://ibm.box.com/s/630pkss1yj46cs2zaa9x10dl5002wpbs" target="_blank">Rocks_TinyYOLOv3.zip</a> MVI Model file, save it. Drag & Drop it from your Mac to within a directory (e.g. `Desktop`) on your Ubuntu VMWare.
2. In MVI Edge `Models` menu, click `Drag and drop file here or click to upload` and upload the `Rocks_TinyYOLOv3.zip` file. Select the Model and click `Deploy`. When in status `Deployed`, the model is ready to be used.
3. Download the following <a href="https://ibm.box.com/s/9xz49kr6bndtb18z9nm94tp6yuq6v73j" target="_blank">Rocks_Samples.zip</a> MVI sample data set, save it. Drag & Drop it from your Mac to within a directory (e.g. `Desktop`) on your Ubuntu VMWare. Unzip the file into a folder on your Ubuntu VMWare - note the folder will contain both `.jpg` and `.xml` files (containing the inference data associated to the images - disregard).
4. In MVI Edge `Input Sources` menu, click `Create` and select `Image Folder`. Enter `Rocks_Sample` in both `Folder Name` and `Input Source Name` fields. Click `Save`. Reopen the Input Source.  In the `Add files` box, drag and drop the 31 `.jpg` files (disregard the `.xml` files) that you unzipped in previous step 3.
5. In MVI Edge `Station` menu, click `Create`, name `Rocks_Station`, Save.
6. Download the following <a href="https://ibm.box.com/s/covolmh3uiewiifxsmrz1suiumpo88gz" target="_blank">Inspection001.json</a> Inspection Template file, save it. 
7. Open the `Rocks_Station` and click `Drag and drop file here or click to upload`, select the `Inspection001.json`. That will create an `Inspection001`.
![image](/images/MVIE4.jpg)

<a id='createmvie'></a>
## Configure Inspection, Run
Let's now configure the Inspection, then run it.
1. Open `Inspection001`. Set `Inspection mode` to `Inspecting`. If nothing appears in `Project`, click `New Project` and call it `Rocks_Project`. 
2. In `Inspecting data set`, if `Rocks_Inspection` does not appear, click `New dataset` and call it `Rocks_Inspection`.
3. Set `Deployed model location` to `Local`. In `Deployed model` section, the `Rocks_TinyYOLOv3` that you deployed in previous section should appear. 
4. Notice how just after you selected `Rocks_TinyYOLOv3`, a set of 11 rules appeared in the `Rules` section. Have a look at those rules and observe that there is 1 `Rule` per rock (e.g. `Calcite`, `Lepidolite` etc) which triggers a `Pass` result when `Confidence score` is `Greater than 0.5`, except for `Gold` and `MilkyQuartz` which have 2 asssociated Rules:  one which triggers a `Pass` when `Confidence score` is `Greater than 0.8` (i.e. `Gold` and `MilkyQuartz`), and another which triggers a `Fail` when `Confidence score` is `Less than 0.8` (i.e. `Gold_ToCheck` and `MilkyQuartz_ToCheck`). 
5. Leave `Device type` in the `Monitor` section blank for now.
6. In the `Input source`, select the `Rocks_Sample` that you created in previous section. Click `Edit input source` and then `Test input source` to check. 
7. Tick the `Time-based trigger` and set `Trigger interval in seconds` to `1`.
8. Click `Review Inspection`. Check that results are returned i.e. that rocks are being idenitifed. Click `Back`.
9. Click `Enable Inspection`. Then switch from the `Configuration` tab to the `Images` tab of your Inspection and regularly (e.g. every 7 seconds) use the top-right `Refresh` button. You will observe that the top-left `Total Images` count goes from 0 to 31, and that more and more pictures are being insspected as time goes by. Expect no more than 17 seconds for all the 31 pictures to have been infered. Click on any of them and notice how various rocks were detected, each with a certain confidence score.

If 1-9 worked well, it's now time to associate a Monitor Device Type to the Inspection:
1. In the `Settings - Maximo Monitor` menu, click `Create`. Name your Device Type `Rocks`, tick all boxes including `Device`, `MVI Version` and `Model UUID`. Click `Create`. Check in Monitor that a new Device Type called `MVIE88A_Rocks` has been created. 
2. Back to the `Configuration` tab of `Inspection001`, select the just created `Rocks` in `Maximo Monitor - Device Type` section, click OK. Notice how the `Rules` have been updated: the `Alert type - Maximo Monitor Settings` is now ticked and a `iot-2/type/MVIE88A_Rocks/id/MVIE88A_Inspection003/evt/result/fmt/json` topic has been created, as well as a `Alert message`. Our inspection results are ready to flow into Monitor. This action has created a Device called `MVIE88A_Inspection001` of Device Type `MVIE88A_Rocks`.

Now, set yourself up to the ability to trigger Inspections on-demand:
1. Open a new Firefox tab and go to the `Input Source` menu and select `Rocks_Samples`. Keep that tab always open. 
2. Notice the `Processed` and `Unprocessed` boxes there, and more specifically, the little `Refresh Wheel` next to `Processed`. The idea is that whenever `Unprocessed` is at `0`, it means that the Inspection has 'consumed' all available images. You then need to simply press that `Refresh Wheel` next to `Processed` and you will see that `Unprocessed` then gets back to `31` and fastly gets to `0`.  
3. So, click on the `Refresh Wheel`any time you want a new Inspection on the 31 data set to be trigerred. You'll need to do that once in a while during a couple of days if you want to start creating relevant Monitor dashboards ... as per next section.

I invite you to create more Inspections (e.g. `Inspection002`, `Inspection003`) using different sets of data. For that purpose, feel free to create new `Input Sources` using e.g. this <a href="https://ibm.box.com/s/bbkusziclafxwhhjcu3hihq3ql4ix7pj" target="_blank">Rocks_Samples_Big.zip</a>  file which contains 109 pictures (vs. just 31) - great for you to simulate anomalies by e.g. 'usually' running an Inspection with `Rocks_Samples.zip`, then once running it with `Rocks_Samples_Big.zip`.

![image](/images/MVIE5.jpg)

<a id='monitordata'></a>
# Create data items & Dashboard in Monitor - Find Gold !
The following image highlights the main screens that you will encounter while executing this section:
![image](/images/MVIE6.jpg)

<a id='python'></a>
## Create PythonExpression data items
Let's create data items that will allow us to count the number of times the various rocks have been detected.
1. In Monitor, go to the `Setup` menu and `Devices` tab. Select `MVIE88A_Rocks` and click top-right `Set up device type` button.
2. Click `Create metric+` link under the `Batch data metric (calculated)`. Select `PythonExpression` from the list. Keep defaults on fist screen and click `Next`.
3. On the `New data item` screen, in the `expression` field, enter `df['objectlabel']=='Gold'`. Click `Next`.
4. Unclick `Auto schedule` and set `Executing every` to `5 minutes` and `Calculating the last` to `5 days`. Name your data item `Gold`, click `Create`.

Repeat 1-4 for: `Calcite`, `Fluorite`, `Glass`, `Gold`, `Lepidolite`, `MilkyQuartz`, `Obsidian`,  `RedJasper`, `RoseQuartz`.

<a id='kmeans'></a>
## Create KMeans Anomaly function on Gold
We will now create an Anomaly Detection data item which, hopefully, should react when an anomalous quantity of Gold rocks is found (e.g. more or less than usual)
1. Click `Create metric+` link under the `Batch data metric (calculated)`. Select `KMeansAnomalyScore` from the list. Keep defaults on fist screen and click `Next`.
2. In the `input_item` fields, select `Gold`. Set `windowsize` to `12`. Click `Next`.
3. Unclick `Auto schedule` and set `Executing every` to `5 minutes` and `Calculating the last` to `5 days`. Name your data item `Gold_KMeans_Anomaly`, click `Create`.

<a id='dash'></a>
## Create Inspection Dashboard and Hourly Dashboard

Finally, we will create 1 Inspection-level Dashboard ...
1. In Monitor, go to the `Monitor` menu and select `MVIE88A_Inspection001`. On top, click `+` to create a new Dashboard. Name it `Overview`. Click `Configure dashboard`.
2. Add `Simple Bar` card. In `Group by` select `Time interval` and in `Data item` select `Calcite`. 
3. Clone card `Calcite` x 8 and create similar card for `MilkyQuartz` etc.
4. For the Gold, create a `Time series line` card and add `Gold` and `Gold_KMeans_Anomaly` data items.

... and 1 Summary Dashboard:
1. In Monitor, go to the `Monitor` menu and select `MVIE88A_Rocks`. On top, click `+` to create a new Dashboard.
2. In `Summary dashboard name`, enter `Hourly`. In `Time grain`, select `Hourly`. Tick all `Dimensions`. Click `Next`.
3. Select all `Data Items` and in the `Aggregation Method` column (1) for the `String` types, select `last` and `count`, (2) for `Metric (calculated)`, select `count`, `last`, `max`, `sum`, `std`, `mean`. Click `Configure` Dashboard. Leave as-is. Click `Create`. Wait 5 minutes. 

![image](/images/MVIE7.jpg)

<a id='optional'></a>
## Optional - Make a cool Gold Anomaly Detection Dashboard
Now that you have all the data items that are needed, you can start creating your own Dashboards.<br> 
Feel free to explore and try-out and e.g. reproduce this Summary dashboard just by looking at the picture.
![image](/images/MVIE8.jpg)
If you are reading this final line, you made it ! Well done, thanks for your time, hope you enjoyed and see you in future labs.

