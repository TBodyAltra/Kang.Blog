---
date: '2024-08-05T20:29:20+08:00'
draft: false
tags: ["LLM"]
ShowToc: true
title: 'An Application of LLMs as HomeKit'
---

I have a camera installed in the baby’s room and I have it streamed to my monitor in my room so that I can keep on eye on her while I am not by her side. However, I cannot be 24 x7 awake, so I need a daemon to monitor the streams and alert me if the baby is in any danger.

I have been an AI engineer in the YOLO era, so I well understand that it would be too difficult (if not even impossible) to implement an algorithm using old-school CNNs or OpenCV. So I turned to LLMs without a doubt. After some research, I installed ollama in my WSL2 environment, and I wrote a simple python script to let LLM monitor snapshots from my camera streaming. To be detailed, I configured VLC player to constantly take a snapshot at a rate of 1/500 frames, and the python script would always fetch the latest snapshot, encode it with base64 and then feed it to the LLM I have chosen (for example, llava). At each iteration a new snapshot is feed to the LLM with a question such as “Is the baby suffocating?“ or “Is the baby in any danger?” and I instruct the LLM to answer with a codec if it considers the answer to my question is positive. After receiving the codec, the python script would alert me by SMS or simply a long last beep.

Turned out the system can work, but I have to turn it off, for now. Because the LLMs simply cannot make a good judge of the condition of the baby. I tried bakllava, llava-13b and a bunch of others. The effect is not satisfying. When the baby is not in the crib, LLMs would claim she is in the crib. When the babysitter is sleeping on the bed next to the crib, LLMs would alert the risk of suffocating (only make sense if the babysitter and baby are sleeping close on the same bed). On the other hand, the LLM did alert me when the baby is sleeping in the prone position, it makes sense since the baby could potentially get suffocated by the mattress (but we would make the baby sleep in prone position sometimes to make her comfortable and somebody would be watching over her). In a nutshell, this system is too sensitive that it would be alerting most of the time!

Maybe I should tweak the prompts or install a new camera just up to the face of my baby so that LLMs would ‘see’ more clearly.
