---
name: gee-download
description: Download satellite images or the basic info of cloud and snow/ice coverage using the gee_downloader tool. Use when user wants to download certain satellite images for some specific region from google earth engine.
user-invocable: true
allowed-tools: Read, Edit, Write, Bash, Glob
---

# before any action:
search for downloader's entry file gee_downloader/main.py, and ask the user to choose the right one if more than one are found; Ask the user to download it if it is not found; Ask the user to select the right installed conda env to run the tool.


# instructions
1. create a folder named 'gee_downloader' under the current project root directory to save the images
2. read and analyze the template [configure file](./assets/download.yaml) required by gee_downloader to run
3. create a new configure file under the directory of 'gee_downloader', and set 'save_dir' to the absolute path of 'gee_downloader'; set 'aoi' to the absolute path of the aoi shpfile under the $ {project_root}data/aoi foder; set 'mode' to 'info' 
4. ask the user to provide the basic info that required by the config file, such as aoi, backend, assets, and other parameters for a specific asset ..., and update the new config file
5. display the config file and ask the user to verfiy it
6. download the info csv by running gee_downloader with the new config file `python gee_downloader/main.py -c config.yaml`. For each polygon in the aoi shpfile, one csv file will be generated. 
7. Read all info csv files and concatnate them as one dataframe, then ask the user to provide the cloud and snow/ice percentage to filter out the good images, save the good images to a new csv file named 'good_images.csv'under the current project root directory. The column name for the cell should be 'name', and the date format should be YYYYMMDD.
8. copy the above used config.yaml as config_download.yaml, set 'mode' to 'info'; Ask user to specify the parameters required for the selected assets.
9. run gee_downloader to download images `python gee_downloader/main.py -c config_download.yaml` 