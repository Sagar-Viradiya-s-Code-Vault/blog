---
title: "Vibe coding offline with local LLM and JetBrains AI"
summary: A guide to setup local LLM.
date: 2025-05-08
weight: 1
tags: ["AI", "LLM", "JetBarains AI", "Vibe Coding"]
cover:
  image: images/header.png
  caption: "Image generated with the help of AI (ChatGPT/DALL·E)"
  hiddenInList: false
---

With most popular IDEs now integrating AI tools, AI has become an integral part of the development process. Vibe coding helps you to iterate or fix something faster, making you more productive! While it is fun to vibe code, there is also a concern on privacy. Especially if you are sharing context (Proprietary code snippet or giving access to the entire codebase). For closed-source code, you have to be very careful how you are using AI and what you are sharing with third-party models. For open source code or a quick POC, you can go full brave mode and explore everything that the AI tool can offer.

Some AI tools, like Gemini integration within Android Studio, allow you to use them without sharing context. But the answers you get won’t be as clear and crisp as they would be if the model had full context.

Lately, JetBrains announced its AI integration to all IDEs. You can choose among many third-party models like GPT-4o, Gemini 2.5 pro, etc that will be used along with JetBrains proprietary model. Like Gemini in Android Studio, you can also disable context and use JetBrains AI like another chat assistant. What caught my eye, though, was the ability to plug in your local LLM.

Plugging local LLM means you can share context without worrying about privacy. This blog is my exploration of vibe coding offline by plugging a local LLM to JetBrains AI. I will show you how you can spin your local LLM and connect with JetBrains AI, and what the caveats are with local LLM.

## Running local LLM ✨

First things first, how can you run a local LLM? There are different ways you can run LLM on your machine. You can use GUI based tool like [LM Studio](https://lmstudio.ai), there is also Ollama, which is a CLI tool. IntelliJ AI offers connecting LLM through both [LM Studio](https://lmstudio.ai) and Ollama. Since I am new to AI game I chose the [LM studio](https://lmstudio.ai) as I found it easy and they have good documentation. Let's see how you can spin up LLM using [LM Studio](https://lmstudio.ai). 

Since each model has different capabilities and some are trained and optimized for a specific task, you need to decide which model is best for you. You can browse through different models available on [Hugging Face 🤗](https://huggingface.co).

After installing [LM Studio](https://lmstudio.ai), you can go to the Discover tab to browse through all the models available.

{{< figure src="images/lm_studio_home.png" >}}

{{< figure src="images/browsing_model.png" >}}

Once the model is downloaded, you need to load it.

{{< figure src="images/select_load_model.png" >}}

Then select the downloaded model. Once it is up and running, you can start chatting with LLM in the LM studio.

{{< figure src="images/lm_studio_running_llm.png" >}}

You just created an AI chat assistant backed by LLM running locally!

## Plugging local LLM to JetBrains AI 🔌

Now we have to connect LLM to JetBrains AI. Go to the Developer Tab and start a server on localhost. This will allow plugging LLM into the third-party tool. In our case, it is IntelliJ.

{{< figure src="images/lm_studio_developer_tab.png" >}}

{{< figure src="images/lm_studio_server_running.png" >}}

Once the server is up, it will show you on which IP and port it is reachable. You need this while connecting to the server within IntelliJ. In the above screenshot, you can see server is reachable at `http://127.0.0.1:1234` 

Let's switch to IntelliJ and see how we can connect LLM. 

{{< figure src="images/intelliJ_connect_local_llm.png" >}}

Enable LM Studio within third-party providers and enter the URL on which localhost is running. If everything is good, you should see a green tick mark after you enter the URL and click Test Connection.

Along with thrid-party AI provider, you also need to configure local models and select one running on your machine for Core features and Instant helpers. In the above screenshot, I have chosen the DeepSeek model. 

Finally, enable offline mode, which does not guarantee 100% offline mode, as you can see from the following warning.

> Prevents most remote calls, prioritizing local models. Despite these safeguards, rare instances of cloud usage may still occur.

For this, make sure you disconnect your machine from the internet if you are working on a proprietary project.

## Vibe coding offline 🦖

You are the T-REX now and ready to vibe code offline! Once you configure the local model, you should be able to select it from the chat assistant.

{{< figure src="images/intellij_select_local_llm.png" >}}

In the screenshot above, I selected the `deepseek-coder-v2-lite-instruct-mlx` model. I found this model to be fast and accurate, answering all Kotlin-related questions. But your mileage may vary based on your hardware and the type of questions you want to ask. I would say ~~do your research~~ ask AI which model is best for your use case 😛

Another thing to notice in the above screenshot is that I have turned off the automatic context attachment of Codebase. I found that for few queries it didn't pick the right files for context. You can manually specify files by dragging files to the chat input section for precise control over context.

Let's try the above setup and see if it is working. I was exploring KotlinConf official [KMP app](https://github.com/JetBrains/kotlinconf-app) codebase and asked few questions.

First, I wanted to know what all screens are there in the app, so I just dropped the `KotlinConfNavHost` file. 

<video controls autoplay loop muted style="max-width: 100%; height: auto;">
    <source src="/videos/local_llm_demo.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>

I would say the answer I got was quite satisfactory. This one was simple and not even AI worth question. I just wanted to test if it can answer easy questions.

Next, I wanted to know the logic behind local notification scheduling logic.

<video controls autoplay loop muted style="max-width: 100%; height: auto;">
    <source src="/videos/local_llm_demo_2.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>

Again, I was happy with the result. It broke down the function line by line, explaining the logic.

Lastly, I asked about the iOS implementation of the notification service logic.

<video controls autoplay loop muted style="max-width: 100%; height: auto;">
    <source src="/videos/local_llm_demo_3.mp4" type="video/mp4">
    Your browser does not support the video tag.
</video>

The answer was precise and crisp. After this, I asked a few basic questions related to kotlin in general, and it was able to answer them correctly. The local LLM was up and running, really pushing my Apple Silicon to its limits! However, local LLM has some downsides. Let's discuss them.

## Caveats with local LLM

### 1. Passing context

Auto inference of the context didn't work as expected for me. It was hit or miss. After turning on `Codebase` context in the chat input and asking a generic question, for example, "Could you please explain this file for me?", it didn't pick the open file but some other random files. Attaching the file manually worked best for me.

Another issue was, unlike third-party LLMs running on the cloud, local LLMs have limitations on the size of the context. So, if you want to attach the entire package containing related files, it is not possible.

### 2. Lack of generic inference

Due to the limitation on context auto-inference that I mentioned above, the answer to 
super generic question, for example, "Break down this project for me?"  won't be possible. Since you need to manually attach context, and there is a size limitation, it is super hard to get answer to such question.

Bottom line is, local LLM approach is best if you have some knowledge about the project and want to ask questions specific to files or code snippets.

## Parting thoughts

So far, my exploration to local LLM is very limited, but at least the initial result is promising. Local LLMs are great for closed-source projects where you need to delegate everyday tasks to AI. Also, imagine you have limited or no internet access, you can still get help from AI. As local models continue to improve and hardware gets better and better, running local LLM will become the norm.

JetBrains AI is in the initial phase, and I hope that support for local LLM and all the limitations will improve in the upcoming iterations. If you’re on the fence, I highly recommend giving it a try. If you’ve tried running local LLMs, I’d love to hear your approach and thoughts in the comments below.

Until next time! Stay curious ✌🏻