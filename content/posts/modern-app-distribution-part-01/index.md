---
title: "Modern Android app distribution with GitHub Actions Part — I"
summary: Modern app distribution
date: 2020-08-31
weight: 1
tags: ["CI/CD", "Github Actions", "Android", "Internal test track", "Automation"]
cover:
  image: images/header.jpg 
  
  caption: "Photo by [Bernd 📷 Dittrich](https://unsplash.com/@hdbernd?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/white-and-green-bus-on-road-during-daytime-0vAAngcguI0?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)"
  hiddenInList: false
---

Hey folks 👋 I hope everyone is safe. Now let me ask you a couple of questions about your build/distribution cycle:

- Do you manually create a build for your testing?
- Still distributing your android app old school - like sharing over mail or through google drive?

If you answered yes to any of those questions, you need to stop what you’re doing right now and automate your build/distribution cycle.

This is a 2 part series on Modern Android Distribution where I am going to help you automate your app distribution through GitHub Actions. So let’s get started.

> *This blog assumes that you are well versed with [GitHub Actions](https://github.com/features/actions). If not then I highly recommend exploring it and come back.*

In this part, we will be looking at how to automate internal app distribution through Internal App Sharing.

## Automating internal app distribution through Internal App Sharing

[Internal App Sharing](https://support.google.com/googleplay/android-developer/answer/9303479?hl=en) is an excellent tool on Play console which allows you to share aab/apk internally with your QA team and other stakeholders. At BookMyShow, we are using it for sharing feature builds and integrated builds with the QA team. The whole experience is as good as installing an app from the play store except it is restricted to whitelisted emails. No more side-loading needed, and if that wasn’t good enough, you could even integrate it into your workflow to share it on Slack once you upload apk/aab to Internal App Sharing(IAS).

At a high level, this is what we are looking at:

*Build —> Upload aab/apk to IAS —> Share link from the previous step on Slack.*

Let’s build our workflow.

You will need to decide on an event that triggers your workflow. It could be a push event to a specific branch or [some other event](https://docs.github.com/en/actions/configuring-and-managing-workflows/configuring-a-workflow#triggering-a-workflow-with-events). In most of the cases, it is a push event to a branch from where you are sharing an internal build with the QA team. Let’s configure that first.

```yml
name: Build, Upload to IAS and share on slack.
on:
  push:
    branches:
      # Specify branch from where you are sharing build
      - [branch_name]
```

Now let’s see how to configure the build step we discussed earlier.

```yml
jobs:
  build:
    name: Building app
    runs-on: ubuntu-latest
    steps:
      - name: Checking out branch
        uses: actions/checkout@v2
          with:
            # Specify branch from where you are sharing build        
            ref: '[branch_name]'
      - name: Setting up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      - name: Running build command
        # Run your own gradle command to generate build.
        run: ./gradlew bundleRelease
      - name: Uploading build
        uses: actions/upload-artifact@v2
        with:
          name: bundle
          path: app/build/outputs/bundle/release/app-release.aab
```

As you can see, the build job has four tasks. This job, as we have specified, runs on an instance of a Linux machine. Just like you checkout a branch on your new machine when you setup any project, here you will need to checkout the branch from where you would like to create a build. 

- Step 1: **checking out branch** — does that using a custom action `actions/checkout@v2`. 

- Step 2: **setting up JDK** — will setup JDK so that we can compile our project using a custom action *`actions/setup-java@v1`*.

- Step 3: **running build command** — executes the Gradle command to create a bundle (you can build an apk too if you are not using bundle). 

- Step 4 uploads our generated bundle so that we access it later in our next job — note that we have given a name to our artifact called ‘bundle’ so that we can refer to it later. Also, we have mentioned the file path of our bundle to be uploaded. If you want to share a file across jobs this is how you can achieve it. It uses custom action *`actions/upload-artifact@v2`*.

Alright so we are halfway through our workflow and so far we have managed to build and store our bundle on artifact so that we can use it in our next job.

Let’s move on to our next job but before that…

{{< iframe src="https://giphy.com/embed/26hitpANolPjPmoCc" width="480" height="480" frameBorder="0" class="giphy-embed" >}}

Ready? Let’s move on then!

Since the last job is quite complex we will break it down into different steps.

Each job runs on a different instance of a Linux machine. Just how we configured our previous job, this job too runs on an instance of Linux.

Note the special *‘needs’* tag at line number 3 below. That tells GitHub that this job has a dependency on the name specified. In our case, it is build job.

Our final job has 3 steps. First, we download the build that we stored in the last step of the previous job. We are referring to an artifact that we want to download through the name that we have given *‘bundle’*.

```yml
upload_to_internal_app_sharing:
    name: Uploading build to IAS
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Downloading build
        uses: actions/download-artifact@v2
        with:
          name: bundle
```

Once we have bundle it’s time to upload it to Internal App Sharing.

My fellow engineers at BookMyShow and I have written a [custom GitHub action](https://github.com/sagar-viradiya/internal-app-sharing-action) which allows you to upload your build to Internal App Sharing.

This action requires 3 things to upload your build to IAS.

1. Your service account JSON in text format (we recommend to set it in your GitHub secret)
2. Your application’s package name
3. And the path to your build

For the first two of those, you need to replace with your service account JSON as GitHub secret. The last one would simply be a file name since the downloaded artifact would be at the root level unless you specify a custom path. In case you specify a custom path while downloading, please make sure you use the same path here.

This action spits out 3 outputs

1. **`downloadUrl`** - The download URL generated for the uploaded artifact
2. **`certificateFingerprint`** - The SHA256 fingerprint of the certificate used to sign the generated artifact
3. **`sha256`** - The SHA-256 hash of the artifact

Here is the step for uploading to Internal App Sharing.

```yml
      - name: Uplaoding to IAS
        id: ias
        uses: sagar-viradiya/internal-app-sharing-action@v1.1.0
        with:
          # Your service account JSON GitHub secret
          serviceAccountJsonPlainText: ${{ secrets.[your-github-service-acc-json-secret] }}
          # Your package name 
          packageName: '[your-package-name]'
          aabFilePath: 'app-release.aab'
```

In our next and final step, we notify **`downloadUrl`** output from above step on slack. Again it is using custom action **`rtCamp/action-slack-notify@v2.1.0`**. You need to mention the following :

1. Your slack webhook
2. Channel where you want to notify
3. Message — In our case **`downloadUrl`** output from previous step
4. Title (Default would be ‘Message’)
5. Slack user name (Default would be ‘rtBot’) — The name of the sender of the message. Does not need to be a “real” username

```yml
      - name: Sharing on slack
        uses: rtCamp/action-slack-notify@v2.1.0
        env:
          # Your slack webhook GitHub secret
          SLACK_WEBHOOK: ${{ secrets.[your-slack-webhook] }}
          # Slack channel where you want to notify
          SLACK_CHANNEL: [your-channel]
          SLACK_USERNAME: "JARVIS"
          SLACK_TITLE: "Internal testing build"
          SLACK_MESSAGE: ${{ steps.ias.outputs.downloadUrl }}
```

Voila! Your workflow for automating build and distribution through Internal App Sharing is ready!

Here is gist of the full workflow 👇

{{< gist sagar-viradiya de5f448e811f587b2c599b0a9a7cdb9f internal-app-sharing-workflow.yml >}}

In the next one, we will be exploring how to automate your production release, so stay tuned!

See you in part 2 👋

Stay safe!

*Thanks [Adnan A M](https://medium.com/u/6e51ff92a37b) for proofreading this.*

> *This was originally posted on [DEV Community](https://dev.to/sagarviradiya/modern-android-app-distribution-with-github-actions-part-i-5pp)*