---
title: "Automating Baseline Profile end-to-end on CI"
summary: Baseline profile CI automation
date: 2024-01-16
weight: 1
tags: ["Performance", "BaselineProfile", "Gradle"]
cover:
  image: images/header.jpeg 
  caption: "[Generated With AI](https://www.bing.com/images/create/code-cherry-pick-automation/1-65912e832c0a4e258deb4823f36e3831?id=jXFueAidvsqN5z6werFzpA%3D%3D&view=detailv2&idpp=genimg) on [Bing](https://www.bing.com/images/create?FORM=GENILP)"
  hiddenInList: false
---

Baseline profile allows you to pre-package a list of classes and methods with APK that are required during launch or in the critical user journey (Core flow of your app). Before the baseline profile, this list either had to be generated on the device by analyzing a few app launches or sourced from Play Store if other devices already figured out this list and uploaded it to Play Store. This list of classes and methods is then pre-compiled to machine code so that during app launch and critical user journey this code is ready to execute by CPU thereby making the entire flow faster.

Further reading of this blog requires context on the baseline profile and unfortunately, this is out of the scope for this blog. To start with I would recommend [this](https://youtu.be/yJm5On5Gp4c?si=yqfq8QtnQ5MJbnDq) talk from the Android Dev Summit to get some context on the baseline profile and then continue here.

## The problem at hand

Through this blog, I want to stress upon automation part of the baseline profile. As you make changes in the code you need to update the profile so that it can include new classes and methods required either during launch or during critical user journey.

Keeping the profile up to date is only half of the problem. You need to make sure the profile is working as expected. For that, you need to run macro-benchmark tests to verify profile is reducing launch time.

We will explore possible solutions to the above problems, so let’s get started without further ado!

## Automating profile generation

With [baseline profile gradle plugin](https://developer.android.com/topic/performance/baselineprofiles/configure-baselineprofiles) you can easily update profile as this plugin does the heavy lifting of building the required test APKs, running tests on the device, and copying the generated profile to the correct directory after test completes. Due to scope of this blog I won’t go into details about plugin setup. You would need a [test module setup](https://developer.android.com/topic/performance/baselineprofiles/create-baselineprofile) and [plugin setup](https://developer.android.com/topic/performance/baselineprofiles/configure-baselineprofiles) within the test module.

Assuming your plugin setup is correct, the CI setup is fairly simple. We will focus on GitHub action as our CI platform. Converting setup for other CI platforms shouldn’t be difficult. 

Here is the workflow that will update the profile on every pull request. After a successful run, the workflow should update PR with the modified profile.

```yml
name: Generate Baseline Profile  
  
on: pull_request
   
jobs:  
  build-benchmark-apks:  
    name: Generate baseline profile  
    runs-on: macos-latest  
    timeout-minutes: 20  
  
    steps:  
      - name: Checkout  
        uses: actions/checkout@v3  
  
      - name: Validate Gradle Wrapper  
        uses: gradle/wrapper-validation-action@v1  
  
      - name: Set up JDK 17  
        uses: actions/setup-java@v3  
        with:  
          distribution: 'zulu'  
          java-version: 17  
  
      - name: Install GMD image for baseline profile generation
        run: yes | "$ANDROID_HOME"/cmdline-tools/latest/bin/sdkmanager "system-images;android-33;aosp_atd;x86_64"

      - name: Accept Android licenses
        run: yes | "$ANDROID_HOME"/cmdline-tools/latest/bin/sdkmanager --licenses || true

      - name: Generate profile
        run: ./gradlew :sample:generateBaselineProfile
             -Pandroid.testoptions.manageddevices.emulator.gpu="swiftshader_indirect"

```

You can configure the baseline profile gradle plugin to generate profile through gradle managed device. The above workflow first installs a system image for gradle managed device. Then it accepts the license for the installed system image. Finally, it runs the generation part by executing `./gradlew :sample:generateBaselineProfile` command. Note `android.testoptions.manageddevices.emulator.gpu` test parameter we are passing. This is required to run gradle manged device on GitHub action running on macOS. Without this profile generation will fail throwing device setup error.

> *If you can not use baseline profile gradle plugin for some reason (required AGP 8.0 and above) then this can be tricky as there would be an additional step required to pull the profile from the device and store it in the source directory. [Chris Banes](https://chrisbanes.me/) wrote a [workflow](https://twitter.com/chrisbanes/status/1597605571930513408) for this that you can use as a reference.*

## Automating profile verification

When it comes to automating the verification of the profile, things are a little tricky. Profile verification requires running macro-benchmark tests on physical devices. This means you need to spin up a physical device to run verification and fetch benchmark results from the device. For the scope of this blog, we will focus on the Firebase test lab.

Running instrumented tests on the Firebase test lab is not challenging at all. Many of you are already doing this with your instrumented test setup on CI. Since macro benchmark tests are instrumented tests only, this should be fairly simple. What’s challenging though is to download benchmark JSON from the device and parse it on CI to analyze the result.

While contributing baseline profile CI setup on [COIL](https://github.com/coil-kt/coil) (Image loading library) I came across the same challenge. I explored all possible solutions to automate the verification part and ultimately decided to write a Gradle plugin for this.

[Auto-Benchmark](https://github.com/sagar-viradiya/auto-benchmark/tree/main) (Sorry for the shameless plug 😅) is a gradle plugin to automate this step as this is a bit complicated and not straightforward. The only limitation here is this plugin only supports Firebase Test Lab (FTL). It uploads benchmark APK and App APK to FTL and once tests are complete it downloads the result JSON. This JSON is then parsed and analyzed to determine the impact of the profile. The impact here is the percentage of improvement between no compilation median startup and baseline profile median startup time. Let’s see how you can utilize this plugin.

Following is the plugin configuration you need in your app-level build.gradle file.

```kotlin
autoBenchmark {
	  
    // A relative file path from the root to app apk with baseline  profile
    appApkFilePath.set("/sample/build/outputs/apk/benchmark/sample-benchmark.apk")
    
    // A relative file path from the root to benchmark apk
    benchmarkApkFilePath.set("/benchmark/build/outputs/apk/benchmark/benchmark-benchmark.apk")
	
    // A relative file path of service account JSON to authenticate Firebase Test Lab
    serviceAccountJsonFilePath.set("../../.config/gcloud/application_default_credentials.json")
    
    // Physical device configuration map to run benchmark
    physicalDevices.set(mapOf(
        "model" to "redfin", "version" to "30"
    ))
    
    // Tolerance percentage for improvement below which verification will fail
    tolerancePercentage.set(10f)
}
```

It takes three file paths. Two of them are relative paths to benchmark and app APKs. To authenticate Firebase Test Lab it requires your service account JSON file path. Next, it takes the device configuration on which you want to run your benchmark. In the above configuration, it uses Pixel 5 and API level 30. Lastly, It takes the `tolerancePercentage`. It is the percentage of improvement below which you want to fail your profile verification. So for example, if no compilation median startup is 233 ms and baseline profile median startup is 206 ms then startup time is improved by ~11%. If the provided tolerance percentage is 10 then the task will successfully complete.

To seed the tolerance percentage I would suggest running the benchmark on the initial profile locally many times (preferably more than 5 times) and taking the minimum percentage improvement between no compilation median startup and baseline profile median startup time.

>*Seeding tolerance percentage like this is something I came up with. I know this is controversial as some of you might find this not a proper way to decide the baseline on your improvement. I would love to know your thoughts in case you have better suggestions on seeding tolerance percentage.*

With the above simple configuration, it is just a matter of issuing one gradle command `./gradlew :app:runBenchmarkAndVerifyProfile` to verify profile. For a detailed setup of this plugin, I would suggest checking the [README guide](https://github.com/sagar-viradiya/auto-benchmark).

Now that we have seen how plugin works let’s see how you can integrate the plugin in CI pipeline. Once again we will see how you can set up this on GitHub action but it should be simple to migrate this on your choice of CI.

In the above configuration, we saw we need a service account JSON file path. We need to set up that first before we run profile verification on CI. For that first store base64 encoded service account JSON file as the GitHub environment variable. Then later in the workflow, we will decode this file and put it at the root level of the CI machine file system. To encode service account JSON run the following command.

```bash
base64 -i “$HOME/.config/gcloud/application_default_credentials.json” | pbcopy
```

Paste the encoded JSON file to the GitHub environment variable `GCLOUD_KEY`. To decode this and restore it in the file system of the CI machine, run the following bash commands as a workflow step.

```bash
GCLOUD_DIR="$HOME/.config/gcloud/"  
mkdir -p "$GCLOUD_DIR"  
echo "${{ vars.GCLOUD_KEY }}" | base64 --decode > "$GCLOUD_DIR/application_default_credentials.json"
```

The above commands decode the encoded service account JSON file from the GitHub environment variable at `$HOME/.config/gcloud/application_default_credentails.json`

Now let’s see the entire workflow.

```yml
name: Verify Baseline Profile  
  
on:  
  pull_request:  
    branches:  
      - release  
  
jobs:  
  build-benchmark-apks:  
    name: Build APKs and Run profile verification  
    runs-on: macos-latest  
    timeout-minutes: 20  
  
    steps:  
      - name: Checkout  
        uses: actions/checkout@v3  
  
      - name: Validate Gradle Wrapper  
        uses: gradle/wrapper-validation-action@v1  
  
      - name: Set up JDK 17  
        uses: actions/setup-java@v3  
        with:  
          distribution: 'zulu'  
          java-version: 17  
  
      - name: Build benchmark apk  
        run: ./gradlew :benchmark:assembleBenchmark  
  
      - name: Build app apk  
        run: ./gradlew :sample:assembleBenchmark  
  
      - name: Set up authentication  
        run: |  
          GCLOUD_DIR="$HOME/.config/gcloud/"
          mkdir -p "$GCLOUD_DIR"          
          echo "${{ vars.GCLOUD_KEY }}" | base64 --decode > "$GCLOUD_DIR/application_default_credentials.json"  
      - name: Verify baseline profile  
        run: ./gradlew :sample:runBenchmarkAndVerifyProfile
```


In the above workflow, `Set up authentication` step is exactly doing what we saw above to restore the service account JSON file on the CI machine for the GitHub environment variable. After setting up credentials it runs the gradle command from plugin to verify profile. 

This is all you need to detect regression in your baseline profile setup. You can check both workflows we saw in my [repo](https://github.com/sagar-viradiya/auto-benchmark/tree/release)for reference. 

One important thing that I want to highlight here is, this plugin is only to detect the performance dip below tolerance percentage and to fail fast on CI. You can go further and collect these benchmark run results to visualise on a dashboard. Monitoring macro-benchmark results on a dashboard would give you more insights into app performance over time. You can refer to [this](https://github.com/android/performance-samples/tree/main/MacrobenchmarkSample/ftl) performance sample repo guide to know how it can be done through GCloud Monitoring. Also, [Py ⚔](https://p-y.wtf/)published a [script](https://blog.p-y.wtf/a-script-to-compare-two-macrobenchmarks-runs) that Square uses to compare two benchmark runs. Perhaps these are good starting points to build a dashboard around your macro-benchmark runs.

I hope this guide will help you to automate baseline profile on CI. I would appreciate your feedback on the plugin so please do try it out and let me know what can be improved.

Until next time! ☮️ ✌️

Thanks to [Shreyas Patil](https://shreyaspatil.dev/) & [Ben Weiss](https://medium.com/u/65fe4f480b1c) for proofreading this.

> *This was originally posted on [Google Developer Expert Medium publication](https://medium.com/p/fc11f8389c0b)*