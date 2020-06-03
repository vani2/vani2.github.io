---
tags: [avsession, ios, photo, video]
date: "2017-12-09T07:12:00Z"
title: Anti-fraud with AVSession
---

![anti-fraud sign](/assets/2017-12-09/anti-fraud.jpeg)

In one of our apps, I should do something called “hidden photos”. I’ll explain it. This is not spying for our users for sure. This feature was implemented for an enterprise application, users as employees use a camera in the main process of their job. So our client wants to be sure they don’t trick and make real photos of real objects. The real scenario is to make hidden photos every 10 seconds.

<!--more-->

# The problem

If you just capture an image the same way the user press the photo button, UI will freeze for a moment in a random moment. This is obvious for a user that something strange happened. The next step is to try to launch video and make captions from it, but you can’t capture still images and record video from a camera using the same instance of AVSession.

# The solution

## Step 1

After some experiments, reading documentation, and thinking, I realized how to do that easily and elegantly.
Add video output to the existing session.

```swift
let videoOutput = AVCaptureVideoDataOutput()
videoOutput.videoSettings = [
    kCVPixelBufferPixelFormatTypeKey as String:
    kCVPixelFormatType_32BGRA
]
videoOutput.setSampleBufferDelegate(self, queue: DispatchQueue.global(qos: .background))
captureSession.addOutput(videoOutput) // your session previously configured for capturing still image
```

## Step 2

Implement delegate to obtain sample buffer.

```swift
class CameraManager {
    // Storage for last sample buffer from session
    private var lastCapturedBuffer: CMSampleBuffer?
    
    //...
}

extension CameraManager: AVCaptureVideoDataOutputSampleBufferDelegate {

    public func captureOutput(_ output: AVCaptureOutput, didOutput sampleBuffer: CMSampleBuffer, from connection: AVCaptureConnection) {
        lastCapturedBuffer = nil
        CMSampleBufferCreateCopy(kCFAllocatorDefault, sampleBuffer, &lastCapturedBuffer)
    }
  
}
```

Note that you need to copy buffer, because if you change it later, you can crash because the pointer is no longer valid.

Also we should manually set `lastCapturedBuffer` to nil to release the previous pointer. Otherwise `kCMSampleBufferDroppedFrameReason_OutOfBuffers` error occurred. You can debug such situations by printing `kCMSampleBufferAttachmentKey_DroppedFrameReason` from `sampleBuffer`
when `captureOutput(_ output: didDrop sampleBuffer: from connection:)` is called.

## Step 3

Convert sample buffer to `UIImage` by demand.

```swift
class CameraManager {
 
    // last captured photo
    var lastHiddenPhoto: UIImage? {
        if let buffer = lastCapturedBuffer {
            return CameraBufferConverter.image(from: buffer)
        } else {
            return nil
        }
    } 
    
    //...
}
```

Every 10 seconds I get `lastHiddenPhoto` from `CameraManager` and save it on disk.

## Step 4

To convert image buffer to `UIImage` I use Apple`s sample code with modifications.

```objective-c
@implementation CameraBufferConverter

+ (UIImage *)imageFromBuffer:(CMSampleBufferRef)imageSampleBuffer {
    // Get a CMSampleBuffer's Core Video image buffer for the media data
    CVImageBufferRef imageBuffer = CMSampleBufferGetImageBuffer(imageSampleBuffer);
    
    // Lock the base address of the pixel buffer
    CVPixelBufferLockBaseAddress(imageBuffer, kCVPixelBufferLock_ReadOnly);
    
    void *baseAddress = CVPixelBufferGetBaseAddress(imageBuffer);
    
    // Get the number of bytes per row for the pixel buffer
    size_t bytesPerRow = CVPixelBufferGetBytesPerRow(imageBuffer);
    // Get the pixel buffer width and height
    size_t width = CVPixelBufferGetWidth(imageBuffer);
    
    size_t height = CVPixelBufferGetHeight(imageBuffer);
    
    // Create a device-dependent RGB color space
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    
    // Create a bitmap graphics context with the sample buffer data
    CGContextRef context = CGBitmapContextCreate(
     baseAddress,
     width,
     height,
     8,
     bytesPerRow,
     colorSpace,
     kCGBitmapByteOrder32Little | kCGImageAlphaPremultipliedFirst);
    
    // Create a Quartz image from the pixel data in the bitmap graphics context
    CGImageRef quartzImage = CGBitmapContextCreateImage(context);
    
    // Free up the context and color space
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    
    // Create an image object from the Quartz image
    UIImage *image = [UIImage imageWithCGImage:quartzImage
                                        scale:1
                                  orientation:UIImageOrientationRight];
    
    // Release the Quartz image
    CGImageRelease(quartzImage);
    
    // Unlock the pixel buffer
    CVPixelBufferUnlockBaseAddress(imageBuffer, kCVPixelBufferLock_ReadOnly);
    
    return image;
}

@end
```

## The result

Capturing hidden photos is smooth now and that’s hidden. It’s not complex code and I didn’t mention any bugs or performance issues. If you need to customize quality and other options of hidden photos look closely at video settings of video output (step 1).
