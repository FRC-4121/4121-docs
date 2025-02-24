# Glossary of Design Patterns

When you're programming, there are certain design patterns that can make your code clearer. While you shouldn't _overuse_ them, an understanding of them is important to write good code, and you'll often find yourself drawn to them in the first place.

## Polymorphism

_also present in `vision.md`_

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

### Example 1: Shared behavior

```java
// 4121-Vision-Java;camera/CameraBase.java
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
// 4121-Vision-Java;camera/FrameCamera.java
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

### Example 2: Incremental Behavior

This is an example of when polymorphism is used to incrementally build upon behavior. Let's look at our vision processing code:

```java
// 4121-Vision-Java;process/VisionProcessor.java
abstract class VisionProcessor {
  // ...
  public abstract void process(Mat img, CameraBase handle /* other arguments*/);
  public abstract void toNetworkTable(NetworkTable table, CameraBase handle);
  public abstract void drawOnImage(Mat img, CameraBase handle);
}
```

This defines what a vision processor needs to be able to do: process an image, send it to the network table, and draw on the image. There's a lot of ways that this can be done, but we're going to focus on the main chain of inheritance.

```java
// 4121-Vision-Java;process/InstancedVisionProcessor.java
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
// 4121-Vision-Java;process/ObjectVisionProcessor.java
abstract class ObjectVisionProcessor extends InstancedVisionProcessor<Collection<VisionObject>> {
  protected abstract Collection<VisionObject> processObjects(Mat img /* other arguments */);
  // processStateful calls processObjects and saves the seen objects, toNetworkTableStateful sends some information about the objects, and drawOnImageStateful draws rectangles.
}
```

At this level, state management is taken care of, and it allows any derived class to only focus on finding objects in the image, not what's being done with them. From there, `RectVisionProcessor` and `AprilTagVisionProcessor` implement this and have the convenience of only focusing on finding objects in the frame.

## Factories

_also present in `vision.md`_

Both the cameras and the vision processors can be specified in configuration files, but they have additional information on how to create them. While a constructor defines how to create an object of a known type, a factory describes the construction of an object of an unknown type. As an example, let's look at how cameras are created:

```java
// 4121-Vision-Java;load/CameraFactory.java
abstract class CameraFactory {
  public abstract String typeName();
  public class<? extends CameraConfig> configType() { /* default config type*/ }
  public void modifyBuilder(GsonBuilder builder) {}
  public abstract CameraBase create(String name, CameraConfig ccfg, LocalDateTime date) throws IOException;
}
```

The focus here is the `create` method. We can't just create a `CameraBase` directly because it's an abstract class, but we don't know ahead of time which derived class to create. This is where the factory pattern comes in. Here's the factory for building a `VideoCaptureCamera`:

```java
// 4121-Vision-Java;camera/VideoCaptureCamera.java
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

### An Alternative to Factories: Functions

There is an alternative to the factory pattern, but it's a bit messier. The key thing to note is that it's essentially doing the same thing, but the data is expressed as fields on a record (or class) rather than the function table for a class:

```java
public record CameraParams(String name, CameraConfig cfg, LocalDateTime date) {}
public record CameraFactory(String typeName, Class<? extends Config> config, Function<CameraParams, CameraBase> creator) {}
```

Different instances of the `CameraFactory` could be used, with no polymorphism, and this is sometimes what you have to do in languages without polymorphism. Doing it this way loses the specificity of what exceptions the builder can throw and doesn't allow a type-level name.

## Beans, Builders, and Setters

Sometimes you have a lot of configuration for a type. Storing it all in fields is great if it stays within the type, but if a lot needs to be passed in, things can get messy fast.

Consider this command in the robot code:

```java
// 4121-Robot;commands/AutoDrive.java, as of 2025-02-24
public class AutoDrive extends AutoCommand {
  private final SwerveDrive drivetrain;
  private double targetDriveDistance; // inches
  private double targetAngle;
  private double targetRotation;
  private double targetSpeed;
  private double currentGyroAngle = 0;
  private double gyroOffset;
  private double frontAngle;

  // ...

  public AutoDrive(SwerveDrive drive, double speed, double dis, double ang, double heading, double rotation, double time) {
    // ...
  }
  // ...
}
```

This is a lot of parameters to deal with, and it can be difficult to tell at a glance what they all do. This is better with inline hints offered by most IDEs, but good code should be legible without an IDE. Let's look at a few ways we can make this better:

### Beans and Records

A "bean" is a class with all public fields and few or no methods. It looks like this:

```java
public class AutoDrive extends AutoCommand {
  private final SwerveDrive drivetrain;

  public static class Config {
    public double driveDistance; // inches
    public double angle;
    public double rotation;
    public double speed;
  }

  private Config config;

  public AutoDrive(SwerveDrive drive, Config config, double time) {
    // ...
  }
}
```

Now that all of our variables in the config have names, it could look like this:

```java
var autoDrive = new AutoDrive(
  drive,
  new Config() {
    {
      driveDistance = 10;
      angle = 0.2;
      speed = 0.5;
    }
  },
  10
);
```

This syntax technically creates an instance of an anonymous class that extends `Config` with an initializer block, but it's essentially the same as creating an instance with these values.

#### Records

The bean pattern is so common, Java introduced a new language construct for it: the record. Rather than our static class, we could do this:

```java
public static record Config(double driveDistance, double angle, double rotation, double speed) {}
```

This is actually syntax sugar for, meaning it's equivalent to, this:

```java
public static final class Config {
  final double driveDistance;
  final double angle;
  final double rotation;
  final double speed;

  public Config(double driveDistance, double angle, double rotation, double speed) {
    this.driveDistance = driveDistance;
    this.angle = angle;
    this.rotation = rotation;
    this.speed = speed;
  }

  public final double driveDistance() {
    return driveDistance;
  }
  public final double angle() {
    return angle;
  }
  public final double rotation() {
    return rotation;
  }
  public final double speed() {
    return speed;
  }
}
```

In addition, you can provide your own constructors and methods within the braces, if you need to do more. One downside of records is that you can't initialize them like beans, since they can't be inherited from, and it would also require more work in the default constructor, because it's not the "canonical" constructor that was generated by the sugar.

### Builders

Another way of defining configuration is with the builder pattern, which sets the fields through method calls. Finally, when all of the fields are set, the actual type can be constructed. The builder pattern would look like this:

```java
public class AutoDrive extends AutoCommand {
  public static class Builder {
    private double driveDistance;
    private double angle;
    private double rotation;
    private double speed;
    private double time;

    public Builder withDistance(double driveDistance) {
      this.driveDistance = driveDistance;
      return this;
    }
    public Builder withAngle(double angle) {
      this.angle = angle;
      return this;
    }
    // implement the rest of the setters

    // sometimes there are parameters in the final step, other times there aren't, it's more a matter of preference
    // the swerve drive doesn't *feel* like a configuration parameter, so I put it as a parameter for `build` rather
    // than a field, but either way is acceptable
    public AutoDrive build(SwerveDrive drivetrain) {
      // rather than call a messy constructor (which could be kept private), this could handle the initialization
      //  logic itself, especially any validation that needs to be done.
      return new AutoDrive(drivetrain, /* everything else */);
    }
  }
  // original constructor
}
```

To use it, it would look like this:

```java
var driveCommand = new AutoDrive.Builder()
  .withDistance(10)
  .withAngle(0.2)
  .withSpeed(0.5)
  .build(drive);
```

Now, rather than a constructor, every argument is associated with a method call. This is especially helpful for cases in which you want to create a lot of similar objects. Just create your initial builder, then modify only the parameters you need that to be different.

### Chain Setters

The builder pattern uses setters to set each field, with each returning `this` so they can be chained. Sometimes, in cases where there's no validation or processing of the values, it can be worthwhile to just put those setters on the object itself, like this:

```java
public class AutoDrive extends AutoCommand {
  // all of the initial fields

  // still take things that we *need* in the constructor
  public AutoDrive(SwerveDrive drivetrain, double time) {
    // ...
  }

  public AutoDrive withDistance(double distance) {
    targetDriveDistance = distance;
    return this;
  }
  // repeat for all of the other parameters
}
```

With this solution, your initialization looks like this:

```java
var driveCommand = new AutoDrive(drive, 10)
  .withDistance(10)
  .withAngle(0.2)
  .withSpeed(0.5);
```

### Hybrid Solutions

You don't need to just use one of these patterns. One common hybrid solution is a bean with chained setters. CTRE has this in their APIs for configuration, like this:

```java
// blame Mr. Fuller for this mess, my code is far more beautiful

var driveConfigs = new TalonFXConfiguration(); // this class is a config bean

var driveOutputConfigs = driveConfigs.MotorOutput;
driveOutputConfigs.Inverted = InvertedValue.Clockwise_Positive;
driveOutputConfigs.NeutralMode = NeutralModeValue.Brake;
driveOutputConfigs.withDutyCycleNeutralDeadband(DRIVE_DEADBAND); // here's an example of setters (these should ideally be chained)

var driveLimitConfig = driveConfigs.CurrentLimits;
driveLimitConfig.StatorCurrentLimitEnable = true;
driveLimitConfig.StatorCurrentLimit = 100;

var driveSensorConfig = driveConfigs.Feedback;
driveSensorConfig.withFeedbackSensorSource(FeedbackSensorSourceValue.RotorSensor);

// Set drive motor PID constants
var slot0Configs = driveConfigs.Slot0;
slot0Configs.kG = drive_kG;
slot0Configs.kS = drive_kS;
slot0Configs.kV = drive_kV;
slot0Configs.kA = drive_kA;
slot0Configs.kP = drive_kP;
slot0Configs.kI = drive_kI;
slot0Configs.kD = drive_kD;

// Apply drive motor configuration and initialize position to 0
StatusCode driveStatus = swerveDriveMotor.getConfigurator().apply(driveConfigs, 0.050); // not the builder pattern per se, but this is the application of configs
```

In our `AutoDrive` example, it could look like this:

```java
// same as the bean example, but with setter methods on our config
var autoDrive = new AutoDrive(
  drive,
  new Config()
    .withDistance(10)
    .withAngle(0.2)
    .withSpeed(0.5),
  10
);
```

All of these solutions are equally valid, and there aren't any real trade-offs of note beside their clarity. It's really a matter of preference for the most part, but even more important than that is to have a consistent style so someone else writing code with your API (or even you in a few weeks) knows what to expect for their initialization.
