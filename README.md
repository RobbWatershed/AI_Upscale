Android library to upscale a given picture by a 2x factor using the [Real-CUGAN](https://github.com/nihui/realcugan-ncnn-vulkan) AI model.

Sources are adapted from [RealSR-NCNN-Android](https://github.com/tumuyan/RealSR-NCNN-Android) v1.8.3

Used in the [Hentoid](https://github.com/avluis/Hentoid) app.

# Setup

1. Import the AI_Upscale project to Android Studio
2. Build it
3. Get the resulting AAR file and copy it into your main project (e.g. into a "ai-upscale" folder next to the "app" folder)
4. Add it as a dependency

```
implementation files('../ai-upscale/ai-upscale.aar')
```

# Usage

1. Call `init` to load the model once

```kt
val upscaler = AiUpscaler()
upscaler.init(
    applicationContext.resources.assets,
    "realsr/models-nose/up2x-no-denoise.param",
    "realsr/models-nose/up2x-no-denoise.bin"
)
```

2. Call `upscale` to run the upscale process

Sample call
```kt
private fun upscale(rawData: ByteArray): ByteArray {
      val cacheDir = getOrCreateCacheFolder(applicationContext, "upscale") ?: return rawData
      val outputFile = File(cacheDir, "upscale.png") // File upscaler will write to
      val progress = ByteBuffer.allocateDirect(1) // Poll to get progress (0..100)
      val killSwitch = ByteBuffer.allocateDirect(1) // Set to 1 to stop the process
      val dataIn = ByteBuffer.allocateDirect(rawData.size)
      dataIn.put(rawData)

      upscaler?.let {
          try {
              killSwitch.put(0, 0)
              CoroutineScope(Dispatchers.Default).launch {
                  val res = withContext(Dispatchers.Default) {
                      it.upscale(
                          dataIn, outputFile.absolutePath, progress, killSwitch
                      )
                  }
                  // Fail => exit immediately
                  if (res != 0) progress.put(0, 100)
              }

              // Poll for progress while processing
              val intervalSeconds = 3
              var iterations = 0
              while (iterations < 180 / intervalSeconds) { // max 3 minutes
                  pause(intervalSeconds * 1000)

                  // isStopped can be any boolean you set to true when the user hits "cancel" to stop upscaling
                  if (isStopped) { 
                      Timber.d("Kill order sent")
                      killSwitch.put(0, 1)
                      return rawData
                  }

                  val p = progress.get(0)
                  Timber.i("Progress % : "+p)

                  iterations++
                  if (p >= 100) break
              }
          } finally {
              // can't recycle ByteBuffer dataIn
          }
      }

      getInputStream(applicationContext, outputFile.toUri()).use { input ->
          return input.readBytes()
      }
  }
```

3. Be certain to free resources when you're done

```kt
upscaler.cleanup()
```

Working example : https://github.com/avluis/Hentoid/blob/master/app/src/main/java/me/devsaki/hentoid/workers/TransformWorker.kt
