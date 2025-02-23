# Spectacular AI Python SDK for OAK-D

![SDK install demo](https://spectacularai.github.io/docs/gif/pip-install.gif)

## Prerequisites

 * An [OAK-D device](https://store.opencv.ai/products/oak-d)
 * Supported operating systems: Windows 10 or Linux (e.g., Ubuntu 20+)
 * Supported Python versions 3.6+
 * Linux only: set up [OAK-D udev rules](https://docs.luxonis.com/en/latest/pages/troubleshooting/#udev-rules-on-linux)

## Installation

Install the Python pacakge: `pip install spectacularAI`

## Examples

To install dependecies for all examples you can use: `pip install -r requirements.txt`. On Linux, you may also need to `sudo apt install python3-tk` to run the Matplotlib-based visualizations.

 * **Minimal example**. Prints 6-DoF poses as JSON text: [`python vio_jsonl.py`](vio_jsonl.py)
 * **Basic visualization**. Interactive 3D plot / draw in the air with the device: [`python vio_visu.py`](vio_visu.py)
 * **3D pen**. Draw in the air: cover the OAK-D color camera to activate the ink. [`python pen_3d.py`](pen_3d.py)
 * **3D mapping**. Build and visualize 3D point cloud of the environment in real-time.
    Also requires `open3d` to be installed, see [`mapping_visu.py`](mapping_visu.py) for details.
 * **3D mapping with Augmented Reality**. Show 3D mesh or point cloud on top of camera view, using OpenGL. [`mapping_ar.py`](mapping_ar.py)
 * **3D Mapping with ROS Integration**. Runs Spectacular AI VIO and publishes pose information and keyframes over ROS topics. [`mapping_ros.py`](mapping_ros.py)
 * **Advanced Spatial AI example**. Spectacular AI VIO + Tiny YOLO object detection.
    See [`depthai_combination.py`](depthai_combination.py) for additional dependencies that also need to be installed.
 * **Mixed reality**. In less than 130 lines of Python, with the good old OpenGL functions like `glTranslatef` used for rendering.
    Also requires `PyOpenGL_accelerate` to be installed, see [`mixed_reality.py`](mixed_reality.py) for details.
 * **Data recording** for, e.g., replay and troubleshooting: [`vio_record.py`](vio_record.py)
 * **GNSS-VIO** example, reads external GNSS from standard input [`vio_gnss.py`](vio_gnss.py) (see also [these instructions](https://spectacularai.github.io/docs/pdf/GNSS-VIO_OAK-D_Python.pdf))
 * **AprilTag integration**: https://spectacularai.github.io/docs/pdf/april_tag_instructions.pdf
 * **Remote visualization over SSH**. Can be achieved by combining the `vio_jsonl.py` and `vio_visu.py` scripts as follows:

        ssh user@example.org 'python -u /full/path/to/vio_jsonl.py' | python -u vio_visu.py --file=-

    Here `user@example.org` represents a machine (e.g., Raspberry Pi) that is connected to the OAK-D, but is not necessarily attached to a monitor.
    The above command can then be executed on a laptop/desktop machine, which then shows the trajectory of the OAK-D remotely (like in [this video](https://youtu.be/mBZ8bszNnwI?t=17)).

## API documentation

https://spectacularai.github.io/docs/sdk/python/latest

### Coordinate systems

The SDK uses the following coordinate conventions, which are also elaborated in the diagram below:
 * **World coordinate system**: Right-handed Z-is-up
 * **Camera coordinate system**: OpenCV convention (see [here](https://learnopencv.com/geometry-of-image-formation/) for a nice illustration), which is also right-handed

These conventions are _different_ from, e.g., Intel RealSense SDK (cf. [here](https://github.com/IntelRealSense/librealsense/blob/master/doc/t265.md#sensor-origin-and-coordinate-system)), ARCore, Unity and most OpenGL tutorials, most of which use an "Y-is-up" coordinate system, often different camera coordinates systems, and sometimes different pixel (or "NDC") coordinate conventions.

By default, the [pose](https://spectacularai.github.io/docs/sdk/python/latest/#spectacularAI.VioOutput.pose) object returned by the Spectacular AI SDK uses the left camera as the local reference frame. To get the pose for another camera, use either [getCameraPose](https://spectacularai.github.io/docs/sdk/python/latest/#spectacularAI.VioOutput.getCameraPose) OR [getRgbCameraPose](https://spectacularai.github.io/docs/sdk/python/latest/#spectacularAI.depthai.Session.getRgbCameraPose).

![SDK coordinate systems](https://spectacularai.github.io/docs/png/SpectacularAI-coordinate-systems-oak-d.png?v=2)

## Troubleshooting and customization

### Camera calibration

Rarely, the OAK-D device factory calibration may be inaccurate, which may cause the the VIO performance to be always very bad in all environments. If this is the case, the device can be recalibrated following [Luxonis' instructions](https://docs.luxonis.com/en/latest/pages/calibration/) (see also [our instructions for fisheye cameras](https://spectacularai.github.io/docs/pdf/oak_fisheye_calibration_instructions.pdf) for extra tips).

Also calibrate the camera according to [these instructions](https://spectacularai.github.io/docs/pdf/oak_fisheye_calibration_instructions.pdf), if you have changed the lenses or the device did not include a factory calibration.

### Frame rate

The camera frame rate can be controlled with the [Depth AI methods](https://docs.luxonis.com/projects/api/en/latest/components/nodes/mono_camera/)
```python
vio_pipeline = spectacularAI.depthai.Pipeline(pipeline)
changed_fps = 25 # new
vio_pipeline.monoLeft.setFps(changed_fps) # new
vio_pipeline.monoRight.setFps(changed_fps) # new
```
Reducing the frame rate (the default is 30) can be used to lower power consumption at the expense of accuracy. Depending on the use case (vehicular, hand-held, etc.), frame rates as low as 10 or 15 FPS may yield acceptable performance.

### Camera modes

By default, the SDK reads the following types of data from the OAK devices:

 * Depth map at full FPS (30 FPS)
 * Sparse image features at full FPS
 * Rectified monochrome images from two cameras at a reduced FPS (resolution 400p)
 * IMU data at full frequency (> 200Hz)

This can be controlled with the following configuration flags, which can be modified using the [Configuration](https://spectacularai.github.io/docs/sdk/python/latest/#spectacularAI.depthai.Configuration) object, for example
```python
config = spectacularAI.depthai.Configuration()
config.useStereo = False # example: enable monocular mode
vio_pipeline = spectacularAI.depthai.Pipeline(pipeline, config)
# ...
```
Arbitrary combinations of configuration changes are not supported, and non-default configurations are not guaranteed to be forward-compatible (may break between SDK releases). Changing the configuration is not expected to improve performance in general, but may help in specific use cases. Testing the following modes is encouraged:

 1. `useFeatureTracker = False`. Disabled accelerated feature tracking, which can improve accuracy, but also increases CPU consumption. Causes depth map and feature inputs to be replaced with full FPS monochrome image data.
 2. `useStereo = False`. Enable monocular mode. Reduces accuracy and robustness but also decreases CPU consumption.
 3. `useSlam = False`. Disable loop closures and other related processing. Can make the relative pose changes more predictable. Disables reduced FPS image data input.
 4. `useVioAutoExposure = True`. Enable custom auto-exposure to improve low-light scenarios and reduce motion blur (BETA).

### Unsupported OAK-D models

Some (less common) OAK models require setting certain parameters manually. Namely, the IMU-to-camera matrix may need to be changed if the device model was not recognized by the SDK. For example, For example:
```python
vio_pipeline = spectacularAI.depthai.Pipeline(pipeline)
# manual IMU-to-camera matrix configuration
vio_pipeline.imuToCameraLeft = [
    [0, 1, 0, 0],
    [1, 0, 0, 0],
    [0, 0,-1, 0],
    [0, 0, 0, 1]
]
```

## License

For more info, see the [main README.md](../../README.md).
