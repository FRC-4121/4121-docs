# 4121 Vision Overview

The vision system is based around producer/consumer systems and makes heavy use of polymorphism to achieve its modularity. This document is an overview of all of them, along with some implementation details.

## Design Patterns

I prioritize readibility and understanding in my code, not just making something that "works". I advise you to do the same, because it'll save you time in the long run. Also, if I come back in four years and the codebases are a mess, you _will not_ hear the end of it.

But everyone has to learn somewhere, right? I won't go in great depth here as I would have to for this to be your _only_ resource, but hopefully it should help you not be too lost as you read this.

### Polymorphism

This is more a language feature than a design pattern itself, but it's a useful start and ubiquitous throughout my code. Polymorphism allows a derived class to specify some behavior for its base class. Now, if you've been reading Java tutorials, you might have seen some code like this:

```java
class Animal {
  public abstract String makeSound();
}
class Dog extends Animal {
  @Override
  public String makeSound() {
    return "bark!";
  }
}
```

Sure, that's a neat little trick, but what's it actually useful for? Here are some examples of how it's used in our code:

#### Example 1: Shared behavior

```java
// camera/CameraBase.java
abstract class CameraBase {
  // ...
  protected abstract Mat readFrameRaw() throws Exception;
  // ...
  public Mat readFrame() throws Exception {
    // locking and stuff to make sure we don't overwrite the frame when a consumer is using it
    Mat frame = readFrameRaw();
    // framerate throttling, null checks, and an optional crop
    // also handle and log exceptions
  }
  // ...
}
// camera/FrameCamera.java
// this is overridden in VideoCaptureCamera too, but this implementation is simpler.
class FrameCamera {
  // ...
  @Override
  protected Mat readFrameRaw() {
    if (frame == null) {
      return thisFrame.clone();
    }
    thisFrame.copyTo(frame);
    return frame;
  }
  // ...
}
```

In order to read a frame, we have a lot of common behavior between camera implementations. We want to catch and log any exceptions that occur, we want to throttle our framerate based on configuration, and we want to not overwrite a frame while something else is using it. That's a lot of shared behavior, and we don't really want a derived class to do something different. By using polymorphism here, we can now define cameras that vary only in how they get the frame, but still handle it the same way.

#### Example 2: Incremental Behavior

This is an example of when polymorphism is used to incrementally build upon behavior. Let's look at our vision processing code:

```java
// process/VisionProcessor.java
abstract class VisionProcessor {
  // ...
  public abstract void process(Mat img, CameraBase handle /* other arguments*/);
  public abstract void toNetworkTable(NetworkTable table, CameraBase handle);
  public abstract void drawOnImage(Mat img, CameraBase handle);
}
```

This defines what a vision processor needs to be able to do: process an image, send it to the network table, and draw on the image. There's a lot of ways that this can be done, but we're going to focus on the main chain of inheritance.

```java
abstract class InstancedVisionProcessor<S> extends VisionProcessor {
  // ...
  protected abstract void processStateful(Mat img, CameraBase cfg, /* other arguments*/ Ref state);
  protected abstract void toNetworkTableStateful(NetworkTable table, Ref state);
  protected abstract void drawOnImageStateful(Mat img, Ref state);
  // process, toNetworkTable, and drawOnImage call their stateful versions, providing a managed state.
}
```

This instanced vision processor manages separate state for each camera. Every processor needs to "remember" information from the `process` call to use in `toNetworkTable` and `drawOnImage`. The `InstancedVisionProcessor` manages that state and provides an interface that implementations can easily use to store per-camera state, but because it's a derived class, it's not exposed to consumers of the `VisionProcessor` class—and it shouldn't, that's not the callers' concern.

```java
abstract class ObjectVisionProcessor extends InstancedVisionProcessor<Collection<VisionObject>> {
  protected abstract Collection<VisionObject> processObjects(Mat img /* other arguments */);
  // processStateful calls processObjects and saves the seen objects, toNetworkTableStateful sends some information about the objects, and drawOnImageStateful draws rectangles.
}
```

At this level, state management is taken care of, and it allows any derived class to only focus on finding objects in the image, not what's being done with them. From there, `RectVisionProcessor` and `AprilTagVisionProcessor` implement this and have the convenience of only focusing on finding objects in the frame.

### Factories

Both the cameras and the vision processors can be specified in configuration files, but they have additional information on how to create them. While a constructor defines how to create an object of a known type, a factory describes the construction of an object of an unknown type. As an example, let's look at how cameras are created:

```java
// load/CameraFactory.java
abstract class CameraFactory {
  public abstract String typeName();
  public class<? extends CameraConfig> configType() { /* default config type*/ }
  public void modifyBuilder(GsonBuilder builder) {}
  public abstract CameraBase create(String name, CameraConfig ccfg, LocalDateTime date) throws IOException;
}
```

The focus here is the `create` method. We can't just create a `CameraBase` directly because it's an abstract class, but we don't know ahead of time which derived class to create. This is where the factory pattern comes in. Here's the factory for building a `VideoCaptureCamera`:

```java
// camera/VideoCaptureCamera.java
class VideoCaptureCamera {
  // ...
  public static class Factory extends CameraFactory {
    @Override
    public Class<Config> configType() {
      return Config.class;
    }
    @Override
    public String typeName() {
      return "cv";
    }
    @Override
    public void modifyBuilder(GsonBuilder builder) {
      // stuff for fancier deserialization, not relevant here
    }
    @Override
    public VideoCaptureCamera create(String name, CameraConfig cfg, LocalDateTime date) throws IOException {
      return new VideoCaptureCamera(name, (Config)cfg, date);
    }
  }
}
```

The `create` method here just calls a concrete constructor of the `VideoCaptureCamera`. Since it's a factory, though, we can have multiple factories, with each kind calling a different constructor. That means we can have different implementations that abstract away the details of creating a camera—we just put in the parameters and a camera comes out (or possibly an `IOException`).

#### An Alternative to Factories: Functions

There is an alternative to the factory pattern, but it's a bit messier. The key thing to note is that it's essentially doing the same thing, but the data is expressed as fields on a record (or class) rather than the function table for a class:

```java
public record CameraParams(String name, CameraConfig cfg, LocalDateTime date) {}
public record CameraFactory(String typeName, Class<? extends Config> config, Function<CameraParams, CameraBase> creator) {}
```

Different instances of the `CameraFactory` could be used, with no polymorphism, and this is sometimes what you have to do in languages without polymorphism. Doing it this way loses the specificity of what exceptions the builder can throw and doesn't allow a type-level name.

## Cameras

For our purposes, a camera is an instance of an class that extends `frc.vision.camera.CameraBase`. The main part of its API is `readFrame()`, which returns a `Mat`. If the frame fails to capture, it returns the last frame. If this was the first frame, that returns `null` and probably causes a NPE (as of 2025-02-21). To implement the frame capturing, `readFrameRaw()` should be overridden instead. This method can throw any exception or return `null` to signify an error.

The camera also has a lock associated with it that can be used to delay a read for a duration. Right now, this is just a `ReentrantLock` (as of 2025-02-21), but it should probably be a `ReentrantReadWriteLock`. Also, it's not really used anywhere, so that's not a big deal. If someone wants to try some fun new synchronization stuff, it's there!

Finally, there's a `PrintWriter` associated with it. This writer is set up to write to `<log directory>/log_<camera name>_<timestamp>.txt`, and is also used to report errors for vision.

### Loading

In order to be loaded from the configuration file (defaulting to `config/cameras.json`), a camera type needs to have a factory that extends `CameraFactory` registered with the `CameraLoader`. The configuration type from the factory needs to extend `CameraConfig`, which has:

- width
- height
- FOV
- a list of vision processors to use (called `vlibs` as a relic of when processors were called "libraries")
  - note that if this isn't specified, it'll use all available processors!
- framerate throttling
- a timeout in milliseconds for waiting to lock

### `VideoCaptureCamera`

This is the camera that corresponds to OpenCV's `VideoCapture`, which is probably how you'll be grabbing frames. It's flexible enough that it could also probably read from a video file, but that's untested.  
It also has:

- port-based lookup (on the Pi 4 and Pi 5)
- automatic reloading on frame failure, with exponential backoff

The config has additional fields:

- an optional OpenCV index (which attempts to open `/dev/videoN` where N is the index)
- an optional OpenCV path (passed directly to OpenCV for it to figure out)
- a port, specified as:
  - a number, see the "Camera Ports" section below
  - a name (unimplemented)
  - an array of port specifiers
- the camera's brightness
- whether configuration should be skipped entirely (some cameras didn't play nice with OpenCV and GStreamer)
- the fourcc string that specifies the video codec. Common values are "YUYV" and "MJPG".

#### Camera Ports

For port indexing, a number is associated with each physical port. Using OpenCV indices is non-deterministic; even if there's only one camera plugged in, it occasionally picks index 2 instead of 0. While indices are great for testing, for the actual robot, a port number should be used.

Ports 0 and 1 correspond to the USB 3 ports on both Pis. Port 0 is the bottom one, and port 1 is the top one.
Ports 2 and 3 are the USB 2 ports, with 2 being on the bottom and 3 on the top.

### `FrameCamera`

The frame camera just returns the same frame every frame. It has the optional `filePath` configuration parameter, which can specify the path of an image to use, with a black image being used if none is specified or found. This image will be resized to the configured size.

## Vision Processors

## Combined Pipeline

### Camera Threads

### `VisionLibsGroup`

### Miscellaneous Utilities
