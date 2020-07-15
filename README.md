## MediaPipe to openFrameworks
#### Notes on how to connect Google's [MediaPipe ML Framework](https://github.com/google/mediapipe) to openFrameworks

MediaPipe is a cross-platform framework for building multimodal applied machine learning pipelines. I want to be able to use it in external applications.

This tutorial walks through how to stream MediaPipe data out over UDP, so any external app and receive and use the data.

![Demo Gif](https://github.com/madelinegannon/example-mediapipe-udp/blob/master/assets/example-mediapipe-udp.gif)

I show how to modify the mediapipe example _mediapipe/examples/desktop/hand_tracking_ to add in a new node that recieves hand tracking data as input, broadcasts that data over UDP on port 8080, and then passes the tracking data on to the rest of the graph as output.

> Tested on macOS Mojave (10.14.6) and openFrameworks 0.10.1

---
Beigin by installing MediaPipe on your system using [google's instructions](https://google.github.io/mediapipe/). 

Then install and setup Google [Protobufs](https://developers.google.com/protocol-buffers) for openFrameworks using my previous [tutorial](https://github.com/madelinegannon/protobuf_tutorial).

If you've never used [Bazel](https://bazel.build/) before, the build system and organization of Mediapipe can be _really_ confusing. I try to go through step-by-step below, but you can find more information in the [MediaPipe Docs](https://mediapipe.readthedocs.io/en/latest/index.html).

---

#### We're going to modify MediaPipe's desktop hand tracking example to stream out landmarks and bounding rectangles over UDP.
![MediaPipe to openFrameworks](https://github.com/madelinegannon/example-mediapipe-udp/blob/master/assets/example-mediapipe-udp.png)

## 1. Update the Graph Definition to Include our new PassThrough Calculator
_Modify mediapipe/graphs/hand_tracking/hand_tracking_desktop_live.pbtxt_
```
# Add New Node
node {
  calculator: "MyPassThroughCalculator"
  input_stream: "LANDMARKS:hand_landmarks"
  input_stream: "NORM_RECT:hand_rect"
  input_stream: "DETECTIONS:palm_detections"
  output_stream: "LANDMARKS:hand_landmarks_out"
  output_stream: "NORM_RECT:hand_rect_out"
  output_stream: "DETECTIONS:palm_detections_out"
}

# Modify input_stream names of next node

# Subgraph that renders annotations and overlays them on top of the input
# images (see renderer_cpu.pbtxt).
node {
  calculator: "RendererSubgraph"
  input_stream: "IMAGE:input_video"
  input_stream: "LANDMARKS:hand_landmarks_out"
  input_stream: "NORM_RECT:hand_rect_out"
  input_stream: "DETECTIONS:palm_detections_out"
  output_stream: "IMAGE:output_video"
}
```
`NOTE` You can visualize the graph to test that inputs and outpus match up at https://viz.mediapipe.dev/

When you add in the custom MyPassThroughCalculator your graph should look like this:
![Updated Graph Structure](https://github.com/madelinegannon/example-mediapipe-udp/blob/master/assets/mediapipe_graph_view.png)

## 2. Making a Custom Calculator
_Adding UDP, Detections, Landmarks, and Hand Rectangles to the PassThrough Calculator_

1. Copy the file _src/mediapipe/my_pass_though_calculator.cc_ to your Mediapipe calculators directory _mediapipe/calculators/core_.

2. The main differences between `my_pass_though_calculator.cc` and the original `pass_though_calculator.cc` are that it adds UDP streaming, and uses Landmark, Rect, and Detection protobufs in the `::mediapipe::Status Process()` function. Next we need to modify the graph file to declare the Tag of the calculators input and stream. 

## 3. Modifying Calculators BUILD file

1. Add the following in _mediapipe/calculators/core/BUILD_ to include `my_pass_through_calculator` dependencies:

```
cc_library(
    name = "my_pass_through_calculator",
    srcs = ["my_pass_through_calculator.cc"],
    visibility = [
        "//visibility:public",
    ],
    deps = [
        "//mediapipe/framework:calculator_framework",
        "//mediapipe/framework/port:status",
        "//mediapipe/framework/formats:landmark_cc_proto",
        "//mediapipe/framework/formats:rect_cc_proto",
        "//mediapipe/framework/formats:detection_cc_proto",
        "//mediapipe/framework/formats:wrapper_hand_tracking_cc_proto",
    ],
    alwayslink = 1,
)
```

## 4. Modify the Graphs BUILD file

1. Lastly, you need to modify the _mediapipe/graphs/hand_tracking/BUILD_ file to add the new calculator as a dependency:
```
cc_library(
    name = "desktop_offline_calculators",
    deps = [
        "//mediapipe/calculators/core:flow_limiter_calculator",
        "//mediapipe/calculators/core:gate_calculator",
        "//mediapipe/calculators/core:immediate_mux_calculator",
        "//mediapipe/calculators/core:packet_inner_join_calculator",
        "//mediapipe/calculators/core:previous_loopback_calculator",
        "//mediapipe/calculators/video:opencv_video_decoder_calculator",
        "//mediapipe/calculators/video:opencv_video_encoder_calculator",
        "//mediapipe/calculators/core:my_pass_through_calculator"
    ],
)
```

## 5. Add the wrapper proto

1.  Copy the `wrapper_hand_tracker.proto` in this repo's `/mediapipe` [directory](https://github.com/madelinegannon/example-mediapipe-udp/tree/master/mediapipe) into `mediapipe/framework/formats` directory.

2. Add it and its dependencies to `mediapipe/framework/formats/BUILD`:

```
mediapipe_proto_library(
    name = "wrapper_hand_tracking_proto",
    srcs = ["wrapper_hand_tracking.proto"],
    visibility = ["//visibility:public"],
    deps = [
      "//mediapipe/framework/formats:landmark_proto",
      "//mediapipe/framework/formats:detection_proto",
      "//mediapipe/framework/formats:rect_proto",
    ],
)
```


You should be able to build and run the mediapipe hand_tracking_desktop_live example now without any errors. In your mediapipe root directory, run:

```bash
bazel build -c opt --define MEDIAPIPE_DISABLE_GPU=1     mediapipe/examples/desktop/hand_tracking:hand_tracking_cpu
```
```
GLOG_logtostderr=1 bazel-bin/mediapipe/examples/desktop/hand_tracking/hand_tracking_cpu     --calculator_graph_config_file=mediapipe/graphs/hand_tracking/hand_tracking_desktop_live.pbtxt
```


## Sending Multiple Protobuf Messages over UDP
There's no good way to send multiple messages (like a LandmarkList, Rect, and DetectionList) in proto2. Therefore, you have to wrap those messages into a new protobuf.

I called mine `wrapper_hand_tracking.proto` and added it and its dependencies to `mediapipe/framework/formats/BUILD`.

Here's a roadblock I hit:

- MediaPipe uses `protobuf-3.11.4` and is built with Bazel, but I'm using `protobuf-3.6.1` for openFrameworks, and it builds with CMake.
- So if I link the `wrapper_hand_tracking.proto` generated with Bazel to the openFrameworks project, there are compatibility issues with the protobuf static library I've already built and linked.

So the hacky workaround I found was to build `wrapper_hand_tracking.proto` with the MediaPipe build system, and then rebuild it with my system-wide protobuf installation. Here's what I did ... in the top-level `/mediapipe` directory:

1. First build with the .proto with bazel.
    - Most of `wrapper_hand_tracking.proto` file is commented out, just leaving the import calls to link to dependent protos.
2. Second, uncomment the .proto and rebuild with `protoc`.
    - For openFrameworks, I've altread built a separate static protobuf library for openFrameworks using [my previous tutorial](https://github.com/madelinegannon/protobuf_tutorial). I need to generate the `.pb.h` and `.pb.cc` files from `wrapper_hand_tracking.proto` using this library.
    - In `wrapper_hand_tracking.proto`, comment out the import calls at the top of the file, and uncomment the body of the proto (`protoc` doesn't link dependencies, so I just copied all the necessary protobufs into this one file).
    - Go into the `mediapipes\framework\formats` directory and run `protoc --cpp_out=. wrapper_hand_tracking.proto`
    - Move those `wrapper_hand_tracking.pb.h` and `wrapper_hand_tracking.pb.cc` files over to your openFrameworks src folder.
    - Drag and drop these two files into your openFrameworks project (be sure to check "Add to Project"). 
     - There shouldn't be any errors now when you add `#include "wrapper_hand_tracking.pb.h"` to `ofApp.h`
     
## Running the Example

When you run the MediaPipe example _hand_tracking_desktop_live_, it broadcasts any hand landmarks and rectangles on port `localhost:8080`. The openFrameworks example _example-protobuf-udp_ is listening for those protobufs on port 8080.

#### Run the MediaPipe Example
1. From your MediaPipe root directory, run _hand_tracking_desktop_live_:

```
GLOG_logtostderr=1 bazel-bin/mediapipe/examples/desktop/hand_tracking/hand_tracking_cpu     --calculator_graph_config_file=mediapipe/graphs/hand_tracking/hand_tracking_desktop_live.pbtxt
```
You should see a video feed of yourself, with hand landmarks and bounding rectangle overlaid.

#### Run the openFrameworks Example
1. Build _example-protobuf-udp_ in openFramework's Project Generator

2. Drag and drop the `/libs` directory into the Xcode Project Pane and select _Add to Target_.

3. Drag and drop your newly generated `wrapper_hand_tracking.pb.cc` and `wrapper_hand_tracking.pb.h` files into the `/src` directory in the Project Pane and select _Add to Target_.

4. Link the `libprotobuf.a` static library from `/libs/protobuf/lib/osx` in _Project Settings > General > Linked Frameworks and Libraries_.

5. In _Project Settings > Build Settings_, add the following to _Header Search Paths_:
```
$(PROJECT_DIR)/libs/protobuf/include
$(PROJECT_DIR)/libs/protobuf/include/google
$(PROJECT_DIR)/libs/protobuf/include/google/compiler
$(PROJECT_DIR)/libs/protobuf/include/google/compiler/cpp
$(PROJECT_DIR)/libs/protobuf/include/google/io
$(PROJECT_DIR)/libs/protobuf/include/google/stubs
$(PROJECT_DIR)/libs/protobuf/include/google/util
```
6. Hit ⌘-r to Run.

You should see the numbered landmarks and bounding rectangle on a white screen. 

Press 'SPACE' to use your hand to swat around some particles.

> Note: The example runs on the cpu, so it's a little slow. But the framerate improves a bit once a hand is detected.
