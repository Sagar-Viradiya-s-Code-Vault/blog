---
title: "Modern Android app distribution with GitHub Actions Part — II"
summary: Modern app distribution
date: 2020-11-18
weight: 1
tags: ["CI/CD", "Github Actions", "Android", "Internal test track", "Automation"]
cover:
  image: images/header.jpg 
  
  caption: "Photo by [Bernd 📷 Dittrich](https://unsplash.com/@hdbernd?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/white-and-green-bus-on-road-during-daytime-0vAAngcguI0?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)"
  hiddenInList: false
---

Hola folks 👋 , I'm back with part 2 of Modern Android app Distribution. In the first part, we saw how to automate the internal distribution using GitHub Actions and Internal App sharing. If you missed the first part you can check it 👇

[Modern Android app distribution with GitHub Actions Part - I]({{< ref "/content/posts/modern-app-distribution-part-01/index.md" >}})

In this part, we will see how to automate your production release through GitHub Actions and test track.

## Automating app release through the test track

In the first part, we saw Internal app distribution which allows you to quickly share development builds. At [BookMyShow](https://in.bookmyshow.com/) we distribute our feature level builds for QA through [Internal App Sharing](https://support.google.com/googleplay/android-developer/answer/9844679?hl=en&visit_id=637411981827269812-1035070845&rd=1). But what about a formal internal launch of the integrated build before you go live with the release? Test track allows you to distribute your build to a limited set of audience to test it out and get valuable feedback. Play console offers three tracks to distribute your build.

1. Internal (Limited to 100 testers)
2. Closed (Closed group which is invite-only)
3. Open (Anyone can join the testing program)

Once you are happy with your build you can promote it to closed, open, or production track depending on your current track. Read more about test tracks [here](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en).

Let's see how to automate this process. At a high level, we are looking at this.

*Trigger for release build --> Build --> Distribute to one of the track*

Let's build our workflow.

You need to decide on an event that will trigger your workflow. Since we are talking about pre-release distribution usually it is from the release branch and the event which will trigger the build has to be more specific. Tagging a release could be one of the events to trigger release build. Let's see how to achieve this.

```yml
name: Internal release
on:
  push:
    tags:
      - '[1-9]+.[0.9]+.[0.9]+'
```

As you can see you need to specify regex for your version name style and every time you tag your release it will trigger our workflow.

> Note: Please make sure to bump the version name and version code before tagging your release.

Now we need to configure the build job.

```yml
jobs:
  build:
    name: Building release app
    runs-on: ubuntu-latest
    steps:
      - name: Checking out tag
        uses: actions/checkout@v2
      - name: Settting up JDK 1.8
        uses: actions/setup-java@v1
        with:
          java-version: 1.8
      - name: Runing build command
        # Run your own gradle command to generate release build.
        run: ./gradlew bundleRelease
      - name: Uploading build
        uses: actions/upload-artifact@v2
        with:
          name: bundle
          path: app/build/outputs/bundle/release/app-release.aab
```

This is very similar to the one we saw in the previous part.

> Note: In case you are uploading apk you might need to upload mapping.txt as well which contains the obfuscated class, method, and field names mapped to the original names. This mapping file also contains information to map the line numbers back to the original source file line numbers. This file is required by Google Play to deobfuscate incoming stack traces from user-reported issues so you can review them in the Google Play Console.

The final job is to upload our build on the test track. Let's see how to achieve that.

First, we need to download our build that we uploaded in the previous job.

> Note: In case you uploaded apk file in the previous job you need to upload a mapping file as well and the same you need to download.

```yml
    upload_to_test_track:
        name: Uploading build to Internal test track
        needs: build
        runs-on: ubuntu-latest
        steps:
          - name: Downloading build
            uses: actions/download-artifact@v2
            with:
              name: bundle
```

After downloading it is just a matter of using a GitHub Action for uploading build on the test track. You can use [r0adkll/upload-google-play@v1](https://github.com/marketplace/actions/upload-android-release-to-play-store) action.

```yml
      - name: Uploading to test track
        uses: r0adkll/upload-google-play@v1
        with:
          # Your service account JSON GitHub secret
          serviceAccountJsonPlainText: ${{ secrets.[your-github-service-acc-json-secret] }}
          # Your package name
          packageName: 'your-package-name'
          releaseFiles: app-release.aab
          track: internal
```

The key things that you need to specify are

1. Your service account JSON file as a plain text (Or as a JSON file) through GitHub secret.
2. Your application package name.
3. Track you want to upload build to(internal/production/alpha/beta).
4. Mapping file (In case you upload apk file)

The action also allows you to specify a bunch of other things but all of the above are important parameters that are mandatory.

> Note: If you are uploading bundle you need to enroll in app signing by Google Play before uploading your app bundle on the Play Console. Read more about it [here](https://support.google.com/googleplay/android-developer/answer/9842756?hl=en).

Congratulations! Your workflow for automating internal as well as production release is ready 💪

Here is gist of the full workflow 👇

{{< gist sagar-viradiya 618829518708b3c8f51cc1b687c82eb3 internal-test-track-workflow.yml >}}

This brings to an end of the two-part series on Modern Android app Distribution with GitHub Actions. Hope you guys had fun learning automating your build and distribution process. As always feedbacks are welcome so please comment or reach out to me on [twitter](https://twitter.com/viradiya_sagar).

That's it for now folks. Until next time 👋. Happy coding :)

*Thanks to [Akshansh](https://twitter.com/AkshanshD) and [Nicola Corti](https://twitter.com/cortinico) for proofreading this.*

> *This was originally posted on [DEV Community](https://dev.to/sagarviradiya/modern-android-app-distribution-with-github-actions-part-ii-akk)*