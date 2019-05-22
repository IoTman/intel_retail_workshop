# Decoding a video stream using Intel(R) Media SDK (Windows)
In this tutorial you will learn the basic principles behind decoding a video stream using the Intel(R) Media SDK. You will understand how to configure the Intel(R) Media SDK pipeline to decode a 4K 30fps AVC stream initially using a software decode implementation and then optimising the code to utilise hardware based decoding. We will also look at decoding a 4K 10-bit HEVC stream.

## Getting Started

1. **Click the File Explorer shortcut on the taskbar**
2. **Navigate to the `C:\Users\intel\Desktop\Retail\03-MediaSDK\msdk_decode` folder**
3. **Double click the *mdk_decode.sln* file and select Visual Studio 2017 to open the file.** 
    - If presented with a security warning stating "You should only open projects from a trustworthy source. ...", uncheck Ask me for every project in this solution and click OK
    - If presented with a Personalization Account window, click Reenter your credentials

4. **Expand the *msdk_decode* project in the *Solution Explorer* in the right pane.**
5. **Double click on the *msdk_decode.cpp* file to load the main application code.**

![Open Decode Project](images/msdk_decode_1.jpg)

## Understanding The Code
Take a look through the existing code using the comments as a guide. This example shows the minimum API usage to decode a H.264 stream.

The basic flow is outlined below:

 1. Specify input file to decode
 2. Initialize a new Media SDK session and decoder
 3. Configure basic video parameters (e.g. codec)
 4. Create buffers and query parameters
    - Allocate a bit stream buffer to store encoded data before processing
    - Read the header from the input file and use this to populate the rest of the video parameters
    - Run a query to check for the validity and SDK support of these parameters
 5. Allocate the surfaces (video frame working memory) required by the decoder
 6. Initialize the decoder
 7. Start the decoding process:
    - The first loop runs until the entire stream has been decoded
    - The second loop drains the decoding pipeline once the end of the stream is reached
8. Clean-up resources (e.g. buffers, file handles) and end the session.

## Build & Run The Code

1. **Click Build on the menu bar in Visual Studio 2017**
2. **Click Build Solution**

![Build Solution](images/msdk_decode_2.jpg)
 - Make sure the application built successfully by checking the **Output** pane at the bottom of the window.

If it doesn't contain similar messages as seen below, rebuild the solution: 
1. **Click Build on the menu bar**
2. **Click Rebuild Solution**

![Check Build](images/msdk_decode_3.jpg)

### Run the application using the Performance Profiler:
1. **Click *Debug->Performance Profiler...***
2. **Check the *CPU Usage* and *GPU Usage* boxes**
3. **Click *Start* to begin profiling.**

![Performance Profiler](images/msdk_decode_4.jpg)
 - A console window will appear running the application whilst the profiling tool records usage data in the background.

![Application Running](images/msdk_decode_5.jpg)

4. **Wait for the application to finish decoding the video stream and then take note of the *execution time* printed in the console window**
5. **Press ENTER to close the command window and stop the profiling session.**

![Stop Application](images/msdk_decode_6.jpg)

6. **Observe the *GPU Utilization* and *CPU* graphs in the Diagnostics session pane.**

As a result of CPU based decoding taking place, the CPU usage is high while GPU usage is low.

![CPU Usage](images/msdk_decode_7.jpg)

7. **Close the Performance Profiler tab**
8. **Click No on Save Changes Dialog**

## Hardware Decoding
Where possible we want to use hardware based decoding for improved efficiency and speed. The Intel(R) Media SDK is able to select the best decode implementation based on the platform capabilities, first checking to see if hardware can be used and falling back to software if not.

1. **Replace *line 36* in msdk_decode.cpp with the following line to change Media SDK implementation from software to hardware:**
 
 ``` cpp
 mfxIMPL impl = MFX_IMPL_SOFTWARE
 ```

with 

``` cpp
mfxIMPL impl = MFX_IMPL_AUTO_ANY;
```

2. **Click Build on the menu bar**
3. **Click Build Solution**
4. **Click *Debug->Performance Profiler...***
5. **Check the *CPU Usage* and *GPU Usage* boxes**
6. **Click *Start* to begin profiling**
7. **Note the *execution time***

In this scenario, the **GPU** utilisation is high while **CPU** usage is low.

8. **Close the console window**

![GPU Details](images/msdk_decode_8.jpg)

## Further Optimisation
The current code uses **system memory** for the working surfaces as this is the implementation provided by the default allocator when creating an Intel(R) Media SDK session. Allocating surfaces in video memory is highly desirable since this eliminates copying them from the system memory to the video memory when decoding which leads to improved performance. To achieve this we have to provide an external allocator with the ablility to manage video memory using DirectX.

First we need to modify the preprocessor definitions to tell the build system we will be using DirectX based memory allocation.

1. **Right-click on the *msdk_decode* project in the *Solution Explorer* window**
2. **Select *Properties* in the context menu**


![Properties](images/msdk_decode_10.jpg)

 3. **Navigate to *Configuration Properties -> C/C++ -> Preprocessor***
 4. **Select *Preprocessor Definitions*.** 
 5. **Click the arrow and select *<Edit...>* from the options that appear.**

![Preprocessor Edit](images/msdk_decode_11.jpg)

 6. **Type *DX_D3D* into the list of definitions and click *OK* to close the window.**
 7. **Click *Apply***
 8. **Click *OK* to close the *Properties* window.**

![Modify Definitions](images/msdk_decode_12.jpg)

 Create a variable for the external allocator and pass this into our existing **Initialize** function.
 
 9. **Replace the following line in *2. Initiazlize Intel Media SDK session*:**
 
 ``` cpp
sts = Initialize(impl, ver, &session, NULL);
```
 with
 
``` cpp
mfxFrameAllocator mfxAllocator;
sts = Initialize(impl, ver, &session, &mfxAllocator);
```
 
 Update the IO pattern specified in the video parameters to tell the decoder we are using video memory instead of system memory.
 
  10. **Change the following line:**
 ```cpp
 mfxVideoParams.IOPattern = MFX_IOPATTERN_OUT_SYSTEM_MEMORY;
 ```
 to:
```cpp
mfxVideoParams.IOPattern = MFX_IOPATTERN_OUT_VIDEO_MEMORY;
```

Leverage our new allocator when allocating surface memory for our decoder. 
 
 11. **Replace *Section 5* with the code below.**
```
//5. Allocate surfaces for decoder
mfxFrameAllocResponse mfxResponse;
sts = mfxAllocator.Alloc(mfxAllocator.pthis, &Request, &mfxResponse);
MSDK_CHECK_RESULT(sts, MFX_ERR_NONE, sts);

// Allocate surface headers (mfxFrameSurface1) for decoder
mfxFrameSurface1** pmfxSurfaces = new mfxFrameSurface1 *[numSurfaces];
MSDK_CHECK_POINTER(pmfxSurfaces, MFX_ERR_MEMORY_ALLOC);
for (int i = 0; i < numSurfaces; i++) {
    pmfxSurfaces[i] = new mfxFrameSurface1;
    memset(pmfxSurfaces[i], 0, sizeof(mfxFrameSurface1));
    memcpy(&(pmfxSurfaces[i]->Info), &(mfxVideoParams.mfx.FrameInfo), sizeof(mfxFrameInfo));
    pmfxSurfaces[i]->Data.MemId = mfxResponse.mids[i];      // MID (memory id) represents one video NV12 surface
}
```
Destroy our allocator once decoding is completed. 
 
12. **Add the following line of code after the surface deletion *for loop* in *section 8*:**
```
mfxAllocator.Free(mfxAllocator.pthis, &mfxResponse);
```
13. ***Remove* the following line from our cleanup code as it is no longer required:**
```
MSDK_SAFE_DELETE_ARRAY(surfaceBuffers);
```
 
14. **Click Build on the menu bar**
15. **Click Build Solution**
16. **Click *Debug->Performance Profiler...***
17. **Check the *CPU Usage* and *GPU Usage* boxes**
18. **Click *Start* to begin profiling**
19. **Note the *execution time***
20. **Close the console window which should now be significantly improved.**

 - Note the **GPU** is now **fully utilised** whilst the application is running as it is no longer having to wait for frames to be copied from system memory.

![GPU Usage](images/msdk_decode_13.jpg)

## HEVC 4K 10-bit
"What about the latest 4K 10-bit HEVC video streams" I hear you ask? Support for both decode and encode of such streams was introduced with 7th Gen Intel(R) Core(TM) Processors and the Intel(R) Media SDK has full support for both. We will now make the small code modifications necessary to decode a sample 4K 10-bit HEVC stream.

 First we need to update our input source to the 4K 10-bit HEVC sample. This sample has an average bitrate of over 40Mbps, similar to that of a 4K Ultra HD Blu-ray.
 
1. **Change the input file in section *1. Open input file***
 
from:

```cpp
char path[] = "..\\bbb_sunflower_2160p_30fps_normal.h264";
```

to:

``` cpp
char path[] = "..\\jellyfish-60-mbps-4k-uhd-hevc-10bit.h265";
```
Update the codec in our decode video parameters from **MFX_CODEC_AVC** to **MFX_CODEC_HEVC**.:
 
2. **Change the codec ID in section *3. Set Required video paramaters for decode***
 
 from:
 
 ```cpp
 mfxVideoParams.mfx.CodecId = MFX_CODEC_AVC;
 ```
 
 to:
 
``` cpp
mfxVideoParams.mfx.CodecId = MFX_CODEC_HEVC;
```
 - HEVC support is provided as a plugin to the Intel(R) Media SDK which needs to be manually loaded at runtime. 
 
 3. **Add the following code to *section 3* to load the HEVC plugin.**
```
// Load the HEVC plugin
mfxPluginUID codecUID;
bool success = true;
codecUID = msdkGetPluginUID(impl, MSDK_VDECODE, mfxVideoParams.mfx.CodecId);

if (AreGuidsEqual(codecUID, MSDK_PLUGINGUID_NULL)) {
    printf("Failed to get plugin UID for HEVC.\n");
    success = false;
}

printf("Loading HEVC plugin: %s\n", ConvertGuidToString(codecUID));

// If we successfully got the UID, load the plugin
if (success) {
    sts = MFXVideoUSER_Load(session, &codecUID, ver.Major);
    if (sts < MFX_ERR_NONE) {
        printf("Loading HEVC plugin failed!\n");
        success = false;
    }
}
```
Before we proceed to test the code, let's play the sample using the **ffplay** utility leveraging only the CPU. Since this utilty uses the CPU alone, You will notice that it struggles to decode the high bitrate stream fast enough to render at a smooth 30fps.

4. **Open a *Command Prompt* window** 
5. **Navigate to the *03-MediaSDK* folder** 
```
cd C:\Users\intel\Desktop\Retail\03-MediaSDK
```
6. **Play the 4K 10-bit HEVC Jellyfish video with FFplay**:
```
ffplay.exe jellyfish-60-mbps-4k-uhd-hevc-10bit.h265
```
7. **Press the *Esc* key to stop playback at any time**

8. **Click Build on the menu bar in Visual Studio 2017**
9. **Click Build Solution**
10. **Click *Debug->Performance Profiler...***
11. **Check the *CPU Usage* and *GPU Usage* boxes**
12. **Click *Start* to begin profiling**
13. **Note the *execution time***
14. **Close the console window**
15. **Check that the GPU was indeed used to decode the stream**
16. **Review the *CPU** and *GPU** utilisation graphs.** 

As you can see the GPU decoding performance comfortably fulfills the 30fps requirement for smooth playback.

> If you missed some steps or didn't have time to finish the tutorial the completed code is available in the **msdk_decode_final** directory.

## Conclusion
In this tutorial we looked at the Intel(R) Media SDK decoding pipeline and ways to optimise decoding performance on Intel platforms. We explored the performance and power advantages with decoding using the GPU rather than using a software based decoder running on the CPU. We also looked at the advantages of using video memory for our working surfaces instead of system memory to avoid unnecessary memory transfers.

