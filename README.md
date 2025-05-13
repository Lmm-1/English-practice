
# English Practice
## Features

## Content Ingestion Functionality <!-- by Long Qingting -->

CatraMMS provides a flexible content ingestion pipeline supporting multiple ingestion methods:

1. **Local File Upload**
    Users can upload local files through a simple file picker interface. The script example at `CatraMMS/scripts/examples/ingestOfImage/helper/ingestionWorkflow.sh` demonstrates this operation, requiring users to configure:
    - User/API keys for authentication
    - Metadata (title, tags, retention policy)
    - File format validation

    Command example:
    ```bash
    if [ $# -ne 8 ]; then
    echo "Usage: $0 <mmsUserKey> <mmsAPIKey> <title> <tag> <ingester> <profileset> <retention> <fileFormat> ($#)"
    exit 1
    fi

    mmsAPIHostName=mms-api.cibortv-mms.com
    mmsUserKey=$1
    mmsAPIKey=$2
    title=$3
    tag=$4
    ingester=$5
    profileSet=$6
    retention=$7
    fileFormat=$8

    sed "s/\${title}/$title/g" ./helper/ingestionWorkflow.json | sed "s/\${tag}/$tag/g" | sed "s/\${ingester}/$ingester/g" | sed "s/\${profileSet}/$profileSet/g" | sed "s/\${retention}/$retention/g" | sed "s/\${fileFormat}/$fileFormat/g" > ./helper/ingestionWorkflow.json.new

    responseCode=$(curl -o ./helper/ingestionWorkflowResult.json -w "%{response_code}" -k -s -X POST -u $mmsUserKey:$mmsAPIKey -d @./helper/ingestionWorkflow.json.new -H "Content-Type: application/json" https://$mmsAPIHostName/catramms/1.0.1/workflow)
    if [ "$responseCode" -ne "201" ]; then
             echo "$(date +%Y-%m-%d-%H:%M:%S): FAILURE, Ingestion response code: $responseCode"
             exit 2
    fi

    rm ./helper/ingestionWorkflow.json.new

    #print ingestionJobKey
    jq '.tasks[] | select(.type == "Add-Content") | .ingestionJobKey' ./helper/ingestionWorkflowResult.json
![alt text](images/图像上传.png)
![alt text](images/视频上传.png)


2. **Cloud Storage Integration**
    Supports direct content import from third-party cloud providers (e.g., Google Drive, Dropbox) via sourceURL configuration. While no specific cloud SDKs are used, the system supports fetching content from external storage URLs:

    ```json
    "parameters": {
           "sourceURL": "http://myhost/example.mp4",  // Supports HTTP/HTTPS/FTP/FTPS protocols
           // ...
    }


3. **Batch Import**
    Allows users to import multiple files at once, supporting various file formats. The script at CatraMMS/scripts/examples/ingestionOfStreamingURL/ingestionOfStreamingURL.sh demonstrates batch import by processing files containing multiple titles and streaming URLs sequentially:

    ```bash
    if [ $# -lt 8 ]; then
            echo "Usage: $0 <mmsUserKey> <mmsAPIKey> <tag> <ingester> <retention> <encodersPool> <encodingProfilesSet> <streamingURLFile>"
            echo "The current parameters number is: $#, it shall be 9"
            paramIndex=1
            for param in "$@"
            do
                echo "Param #$paramIndex: $param";
                paramIndex=$((paramIndex + 1));
            done
            exit 1
    fi

    mmsUserKey=$1
    mmsAPIKey=$2
    tag=$3
    ingester=$4
    retention=$5
    encodersPool=$6
    encodingProfilesSet=$7
    streamingURLFile=$8

    mmsAPIHostName=mms-api.catramms-cloud.com

    while read titleAndtreamingURL; do
            if [ "$titleAndtreamingURL" = "" ]; then
                continue
            fi

    title=$(echo $titleAndtreamingURL | cut -d ";" -f 1)
    streamingURL=$(echo $titleAndtreamingURL | cut -d ";" -f 2)
    encodedStreamingURL=${streamingURL//\//"\\/"}
    encodedStreamingURL=${encodedStreamingURL//\&/"\\&"}

    sed "s/\${title}/$title/g" ./helper/ingestionWorkflow.json | sed "s/\${streamingURL}/$encodedStreamingURL/g" | sed "s/\${tag}/$tag/g" | sed "s/\${ingester}/$ingester/g" | sed "s/\${retention}/$retention/g" | sed "s/\${encodersPool}/$encodersPool/g" | sed "s/\${encodingProfilesSet}/$encodingProfilesSet/g" > ./helper/ingestionWorkflow.json.new
    curl -o ./helper/ingestionWorkflowResult.json -k -s -X POST -u $mmsUserKey:$mmsAPIKey -d @./helper/ingestionWorkflow.json.new -H "Content-Type: application/json" https://$mmsAPIHostName/catramms/1.0.1/workflow
    done < "$streamingURLFile"
![alt text](images/批量上传.png)
        

4. **Automated Ingestion**
    Implements scheduled ingestion through:

    Filesystem monitoring: Initially designed using incrontab (inotify-based), later adapted to use cron-triggered scripts due to mounted directory limitations
    Watch folder pattern: Periodically scans designated directories for new files



## Content Handling Capability <!-- by Long Qingting -->

1. **Multimedia Format Transcoding**
    The system provides professional media transcoding services, supporting conversion between various video container formats, including but not limited to transcoding source files into standardized container formats such as MP4 and AVI. In the CatraMMS/API/src/FFMPEGEncoderTask.cpp implementation, the downloadMediaFromMMS function establishes a complete transcoding pipeline, specifically designed to handle the download and transcoding process of streaming media content based on the HLS protocol, efficiently converting .m3u8 playlist format streaming content into industry-standard MP4 container format.

    ```cpp
    string FFMPEGEncoderTask::downloadMediaFromMMS(
           int64_t ingestionJobKey, int64_t encodingJobKey, shared_ptr<FFMpegWrapper> ffmpeg, string sourceFileExtension, string sourcePhysicalDeliveryURL,
           string destAssetPathName
    )
    {
           string localDestAssetPathName = destAssetPathName;
           bool isSourceStreaming = false;
           if (sourceFileExtension == ".m3u8")
             isSourceStreaming = true;

    if (isSourceStreaming)
    {
            bool regenerateTimestamps = false;
            localDestAssetPathName = localDestAssetPathName + ".mp4";
            ffmpeg->streamingToFile(ingestionJobKey, regenerateTimestamps, sourcePhysicalDeliveryURL, localDestAssetPathName);
    }
    else
    {
            FFMpegProgressData progressData;
            progressData._ingestionJobKey = ingestionJobKey;
            progressData._lastTimeProgressUpdate = chrono::system_clock::now();
            progressData._lastPercentageUpdated = -1.0;

            CurlWrapper::downloadFile(
                sourcePhysicalDeliveryURL, localDestAssetPathName, progressDownloadCallback2, &progressData, 500,
                std::format(", ingestionJobKey: {}", ingestionJobKey),
                3 // maxRetryNumber
            );
    }

    return localDestAssetPathName;
    }
![alt text](images/格式转换.png)

2. **Media File Compression**
    Allowing users to employ codecs for performing lossy/lossless compression on image (JPEG/PNG) and video (H.264/HEVC) files, significantly reducing bitrate and file size. Although the current codebase does not explicitly contain compression algorithm implementations, the system leverages the FFmpeg multimedia framework, utilizing its built-in libx264/libx265 encoders, CRF (Constant Rate Factor) quality control parameters, and preset systems to achieve efficient transcoding workflows. Developers can optimize rate-distortion (R-D) performance by adjusting quantization parameters (QP), GOP (Group of Pictures) structure, and other professional video encoding parameters.

    Take general compression (H.264 + AAC, balancing image quality and file size) as an example:

    ```bash
    ffmpeg -i Guangwai.mp4 -c:v libx264 -crf 23 -preset medium -c:a aac -b:a 128k Guangwai_compressed.mp4

3. **Metadata Extraction**
    The system automatically extracts file metadata (e.g., title, author, creation date) and stores it in a database to facilitate subsequent management and retrieval. In the buildAddContentIngestionWorkflow function located in CatraMMS/API/src/FFMPEGEncoderTask.cpp, the system processes JSON objects containing metadata. The extracted metadata can be stored within these objects, enabling efficient content management and querying capabilities.
    ```cpp
    string FFMPEGEncoderTask::buildAddContentIngestionWorkflow(
            int64_t ingestionJobKey, string label, string fileFormat, string ingester,
            string sourceURL, string title, json userDataRoot,
            json ingestedParametersRoot, int64_t encodingProfileKey,
            int64_t variantOfMediaItemKey
    )
    {
            json addContentRoot;
            string field = "label";
            addContentRoot[field] = label;
            field = "type";
            addContentRoot[field] = "Add-Content";

            son addContentParametersRoot;

            // ... 处理元数据相关逻辑

            if (userDataRoot != nullptr)
            {
            field = "userData";
            addContentParametersRoot[field] = userDataRoot;
            }

            // ...

            return JSONUtils::toString(workflowRoot);
    }

4. **Batch Processing**
    Supports simultaneous processing of multiple files to enhance workflow efficiency. Integrated with the aforementioned batch import functionality, after importing multiple files, users can perform batch operations such as format conversion, compression, and metadata extraction. Leveraging scripting loops and concurrency mechanisms, the system enables parallel processing of multiple files. For instance, in CatraMMS/scripts/examples/ingestionOfStreamingURL/ingestionOfStreamingURL.sh, the script reads a file containing multiple content entries and sequentially ingests and processes each item, significantly improving overall operational efficiency.





<!--by 罗娜-->
Media Management System

Media Management System functionalities

1.manage "WorkSpaces" for the media contents. Every "Workspace" is a repository of media contents. Example 1: a Workspace could contain all the 'News' contents, another all the 'Sport' contents, ... Example 2: a Workspace could contains all the contents of 'User A', another Workspace could contains all the contents of 'User B', ...

2.manage custom workflows (an example of workflow: ingest two videos, concatenate them, cut the resulting video, overlay a logo on it, encode it using different profiles, ...)

3.manage libraries of Workflows. For example it is possible to create a workflow to retrieve a Picture from a video. It will be composed by several Tasks, a Task looking for a Face into the Video, if it does not find any face (OnError event), there is another Task getting a Picture at a specified instance (i.e.: 30 seconds since the starting of the Video). Then we have the encoding of the generated picture and so on. All that could be saved as a 'Workflow As Library' and it can be used as a simple Task in other Workflows.

4.manage custom metadata associated to each content

5.manage CrossReferences among the Media Contents (i.e.: ImageOfVideo, ImageOfAudio, FaceOfVideo, CutOfVideo, CutOfAudio, ...)

6.manage an array of Encoders, priority of each encoding and dedicated Encoders for each Workspace

7.implement media functionalities (i.e.: encoding, cut (KeyFrameSeeking, FrameAccurateWithoutEncoding and FrameAccurateWithEncoding),

8.concatenate, overlay, slideShow, video speed down, picture in picture, check streaming for monitoring, ...)

9.multi tracks support in any fuctionalities (LiveRecorder, Concat, Cut, ChangeFileFormat, ...)

10.multi bitrate support

11.integrated with Kaltura

12.ngest VOD and Live YouTube contents

13.record live IP streaming, even from YouTube
       
14.live Recorder Virtual VOD

15.implement a proxy for live IP streaming and Media On Demand toward CDN or making live streaming available through HLS/DASH requests

16.Certified CDN: CDN77, AWS

17.Live-Grid (Live-Mosaic) to compose/merge several videos in one video (useful for example for monitoring/surveillance)

18.manage delivery of contents through the creation of authorizations by path or by parameter (tokens, time to live, max retries)

19.post media on socials (facebook, youtube, ...)

20.implement computer-vision (face recognition, face identification, ...)

21.implement image processing

22.player
(1).edit video through user mark in-out. Once a cut is done, in case the cut is not accurate, it is possible easily to tune the cut to improve it. 

(2).edit audio through user mark in-out. Once a cut is done, in case the cut is not accurate, it is possible easily to tune the cut to improve it.

(3).edit image cropping through user selection. Once a crop is done, in case the crop is not accurate, it is possible easily to tune the cut to improve it.

23.library available (actually only a java version is available)
(1).to call easily any MMS REST API

(2).to help building by a program any complex Workflow

(3).support for ldap integration (if needed)

24.Media content might be a video, an audio, an image, a playlist.

25.It is designed to be used as a Service where a User has to register first and then will be able to use all the features of the platform. Once a User is registered, a Workspace is created for him and all the created contents will be associated to the Workspace.

26.Currently CatraMMS is using FFMpeg to encode the contents, ImageMagick and opencv for image processing and computer vision functionalities.

27.CatraMMS provides a detailed REST APIs from which it is possible to do any kind of activity toward the platform. Even a GUI/APP can be built based on the APIs.



    Here follow a presentation of the MMS and the Physical Architecture of the CatraMMS:

      

            ┌─────────────┐
            │    End User │
            └──────┬───────┘
                   │
            ┌──────▼───────┐
            │  User By API │
            └──────┬───────┘
                   │
            ┌──────▼───────┐
            │     GUI      │
            └──────┬───────┘
                   │
            ┌──────▼───────┐
            │  Client APP  │
            └──────┬───────┘
                   │
            ┌──────▼───────┐   ┌────────────────┐
            │ Transcoder   │   │  MMS Engine    │
            │   Servers    │   │    Servers     │
            └──────┬───────┘   └────────┬───────┘
                   │                    │
            ┌──────▼───────┐   ┌────────▼───────┐
            │    API       │   │   Clone tt     │
            │   Servers    │   │   https        │
            └──────────────┘   └────────────────┘


Key System Features

MMS is a fully-featured, highly flexible media management platform that covers the entire lifecycle from ingestion, processing, and analysis to distribution. It is particularly suitable for enterprises or developers handling large-scale media resources. Its core advantages include:

(1)Modular Design: Workspaces, workflows, and APIs can all be customized as needed.

(2)Intelligence & Automation: Enhances efficiency by combining AI and rule-based engines.

(3)Open Integration: Supports deep customization through REST APIs and Java libraries.

<!--by 罗娜-->



<!--by 韦淑静-->

Tutorial for Using the Main Functions of a Media Management System

Special Effects Processing Function

(1) Picture Overlay on Video Implementation Logic: This is implemented by defining a task chain in the workflow configuration file. In the file, specify the GroupOfTasks type and nest the Add-Content subtask to load the video source. Then employ FFmpeg or other professional video processing libraries to configure overlay parameters, enabling precise picture overlay.
Application Scenarios: It is widely used in advertising production (e.g., embedding brand logos) and film post - production (e.g., adding atmospheric effect layers).
Operational Steps:
Material Preparation: Prepare the pictures to be overlaid and the target video files.
Configuration File Writing: Define task types and parameters (e.g., overlay position, transparency, etc.).
Task Execution: Submit the configuration file and the system will automatically synthesize the effects.
The operation is as shown below: Prepare the video file and the pictures to be overlaid. Select the workflow editor on the interface (refer to the following illustration).

![图片叠加至视频](</iamges/picture overlay  video-01.png>)


In the workflow, add the necessary subtasks. The functions are visible in the left - hand task library panel (as shown below). Add the subtask for picture overlay on video, specify the task type and parameters, and save the properties.

![图片叠加至视频](</iamges/picture overlay  video-02.png>)
![图片叠加至视频](</iamges/picture overlay  video-03.png>)

<!--by 韦淑静-->


(2) Slideshow Creation Implementation Logic: This involves integrating video frame processing and splicing functions. Multiple images or video clips are sequenced, and a smooth slideshow video is generated by customizing the duration of each material.
Application Scenarios: Suitable for teaching videos, product showcases, event recaps, etc., it supports the orderly presentation of multiple materials.
Operational Steps:
Material Collection: Organize and sort the images or video clips.
Parameter Configuration: Define the material sequence, playback duration, and transition effects in the configuration file.
Task Execution: Submit the configuration for automatic slideshow video generation by the system.
Operation is as shown below: Add the slideshow function subtask in the left - hand task library panel. Then define the material sequence and playback duration parameters, and save the properties.

![制作幻灯片](</iamges/make slides-01.png>)
![制作幻灯片](</iamges/make slides-02.png>)

<!--by 韦淑静-->


Video Speed Adjustment Function: Video Acceleration or Deceleration

Implementation Logic: This function is developed based on the MMSEngine module and relies on FFmpeg and other professional video processing libraries. The core principle involves adjusting the playback speed by modifying the video frame rate or recalculating timestamps. Increasing the frame rate speeds up the video, while decreasing it slows down the video.
Application Scenarios: In film editing, video acceleration can be used to quickly transition between scenes and create a tense atmosphere, whereas deceleration can highlight details. In sports analysis, slowing down video footage of athletes' movements allows professionals to analyze techniques frame by frame, thereby enhancing the effectiveness of training.

Operational Steps:
Material Preparation: Identify the video file that requires speed adjustment and ensure its format is compatible.
Configuration File Creation: Add a "video speed adjustment" task node in the configuration file and set the speed parameters (e.g., 2x acceleration, 0.5x deceleration).
Task Execution: Submit the configuration file to the system for automatic speed adjustment and generation of the target video.

Operation is as shown below: Add a video speed adjustment subtask in the left - hand task library panel, select the speed type and value, and save the properties.

![视频速度调整](</iamges/video speed-01.png>)
![视频速度调整](</iamges/video speed-02.png>)

Connect all subtasks to build a simple workflow and execute it (operation is as shown below).

![工作流子任务配置](/iamges/工作流配置步骤-01.png)
![工作流子任务配置](/iamges/工作流配置步骤-02.png)

After executing the workflow, the platform page displays the corresponding JSON document, showing all added subtasks.

![工作流子任务配置](/iamges/工作流配置步骤-03.png)
![工作流子任务配置](/iamges/工作流配置步骤-04.png)
![工作流子任务配置](/iamges/工作流配置步骤-05.png)

Finally, we can verify whether the workflow executed earlier was successful. Return to the homepage, click the "settings" icon in the upper right corner of the video, select the relevant workflow, and the system will automatically match the workflow name. Then, click the list icon and select "details" to clearly view the subtasks just added, including picture overlay on video, slideshow creation, and video acceleration. Finally, I need to review the entire content to check for any omissions or adjustments needed, ensuring its professionalism and smoothness, and then prepare to formally reply to the user.

![工作流子任务配置](/iamges/工作流配置步骤-06.png)
![工作流子任务配置](/iamges/工作流配置步骤-07.png)
![工作流子任务配置](/iamges/工作流配置步骤-08.png)
![工作流子任务配置](/iamges/工作流配置步骤-09.png)

<!--by 韦淑静-->


Other Functions

(1) File Storage Management
Function Implementation: File storage management encompasses file ingestion and storage path management. File ingestion is accomplished through the Add - Content task within the workflow configuration.
Function Value: It ensures precise file ingestion, efficient storage, and proper organization, thereby enhancing file management efficiency and simplifying subsequent retrieval, usage, and maintenance. In video processing scenarios, it facilitates the proper storage of video files and related resources, ensuring smooth editing and encoding processes.

(2) Workflow Management
Function Implementation: Workflow management is based on workflow configuration files and supporting processing functions. The configuration file, in JSON format, clearly defines essential information such as workflow type, task content, and parameter settings.
Function Value: It enables automated and coordinated system task processing. By defining workflows, related tasks can be combined in a set order and rules, improving work efficiency and accuracy. In a video processing system, workflows covering tasks like video uploading, encoding, and special - effects addition can be created, allowing the system to automatically complete the entire process according to the configuration.

(3) System Configuration Management
Function Implementation: System configuration management involves various configurations, including databases and encoding priorities. At the code level, flexible configuration management is achieved using JSON configuration files and database operations.
Function Value: It enables customized system configuration based on different requirements and environments. By adjusting parameters, system behavior and performance can be optimized, and its adaptability can be enhanced. In a video processing system, encoding priorities can be configured according to video quality requirements to meet diverse user needs.


<!--by 韦淑静-->

