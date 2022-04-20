---
layout: post
title:  "Easy and flexible events managing for behavioral studies in Unity"
date:   2022-04-19 16:00:00 +0700
categories: unity
tags: [behavioral experiments, unity, uxf]

[//]: # (tweet: tweet id)
---

How to create a script to run a behavioral experiment in Unity? Your first intention might be to connect all elements of the study (e.g., instructions, stimuli) to a script that is often called “MainController”. Then you would be tempted to run an experiment in one huge coroutine. I believe this is acceptable only for a small and easy task. But programming will turn into a nightmare in a larger study. For instance, experiments can include several instructions before specific trials; practice trials that are different from the main trials; several types of tasks in each trial; specific feedback at the end of some trials, you name it! 👹🙀💀 Here I will describe how one can simplify and decouple the code of the experiment. 💪🙌 I will use the `Event Bus` programming pattern combined with the [Unity Experiment Framework (UXF)](https://github.com/immersivecognition/unity-experiment-framework) to create a fairly complex behavioral experiment.

#### tl;dr
* The example project is published [here]( https://github.com/vagechirkov/TemplateEventBusUXF). You can test it [here]( https://vagechirkov.github.io/TemplateEventBusUXF/).
* [ExpEventBus.cs](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/ExpEventBus.cs) and [ExperimentEventTypes.cs](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/ExperimentEventTypes.cs) contain Event Bus related scripts.
* [ExperimentManager.cs](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/ExperimentManager.cs) contains the experiment loop that iterates through all the trials in the experiment.
* [GenerateExperiment.cs](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/GenerateExperiment.cs) is a script to generate UXF session content.
* When the global event (e.g. “ActionBegin”) is published, all the subscribed functions in any script in the project are triggered. This allows to decouple and simplify the code.

#### Experiment 🦅
I will use a mock experiment in which participants are instructed to steer a bird. The task is to keep the bird (the avatar) in the center of a flock. The experiment consists of a practice trial, where the avatar is steering the flock without the flock around, and several main trials in which the avatar is among the flock. The trial consists of the instruction, the bird steering part, and the evaluation of performance. You can look at the actual experiment [here](https://vagechirkov.github.io/TemplateEventBusUXF/). I believe that many behavioral experiments have a similar structure. To simulate the flocking birds, I used a [free unity asset](https://assetstore.unity.com/packages/3d/characters/animals/simple-boids-flocks-of-birds-fish-and-insects-164188) developed by [Nicholas Veselov]( https://assetstore.unity.com/publishers/30428).

<figure>
<img src="/event-bus-with-uxf/unity-editor.png" alt="the experiment scene in Unity Editor">
<figcaption>The experiment scene in Unity Editor</figcaption>
</figure>


#### Event Bus 🚌
Event Bus is a simple programming pattern that allows the creation of **globally broadcasted events**. One can **subscribe** functions from any part of the project to these events, so that when the event is **published** the necessary function will be triggered. Events can be published from any script in the project and are independent of the subscribed functions. The main benefit is that one can subscribe as many functions in the separate scripts to a specific event as one needs. As soon as this event is published, all these functions will be triggered **simultaneously**.

To learn more about the Event Bus, check chapter 6 in Game Development Patterns with Unity 2021 (code is available [here](https://github.com/PacktPublishing/Game-Development-Patterns-with-Unity-2021-Second-Edition/tree/main/Assets/Chapters/Chapter06) with the [video demonstration](https://youtu.be/rGaXu0bnoOQ)).


#### Unity Experiment Framework (UXF)🧪

UXF is a nice and robust framework for developing behavioral experiments in Unity. It allows to create an experiment structure using **Trials** and **Blocks**, store **settings** for each trial, block, or the whole session, and record participant’s behavior. It has its own event system that is connected to the beginning and the end of the trial. In this post, I will present how to extend UXF event system using the Event Bus programming pattern. To get more information check [UXF GitHub repo]( https://github.com/immersivecognition/unity-experiment-framework).


#### Publish and Subscribe📢

Let’s look closer at how to use the Event Bus programming pattern. One particularly nice example in the project is the [`BackgroundController`](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/BackgroundController.cs). Let’s say you want to have a white background throughout the whole experiment and turn it off only when there is an action or practice task. The only thing you will need is to switch off the canvas when `ActionBegin` or `PracticeBegin` are published and turn it on again at the beginning of the performance evaluation (`EvaluationBegin `). Here is the code:

```csharp
using UnityEngine;

public class BackgroundController : MonoBehaviour
{
    [SerializeField] Canvas canvas;

    void OnEnable()
    {
        ExpEventBus.Subscribe(ExpEvents.ActionBegin, () => canvas.enabled = false);
        ExpEventBus.Subscribe(ExpEvents.PracticeBegin, () => canvas.enabled = false);
        ExpEventBus.Subscribe(ExpEvents.EvaluationBegin, () => canvas.enabled = true);
    }
}
```

Note that the background controller is completely isolated from other scripts. If you don’t need it anymore, you can easily delete it without breaking an experiment.

Now let’s look at how one can publish an event. This is illustrated in the [performance evaluation script](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/Evaluation.cs).

```csharp
using System.Collections;
using UnityEngine;
using UnityEngine.UI;

public class Evaluation : MonoBehaviour
{
    [SerializeField] GameObject evaluationPanel;
    [SerializeField] Slider slider;
    void OnEnable()
    {
        ExpEventBus.Subscribe(ExpEvents.EvaluationBegin, () => StartCoroutine(EvaluatePerformance()));
    }

    IEnumerator EvaluatePerformance()
    {
        evaluationPanel.SetActive(true);
        yield return new WaitUntil(() => !evaluationPanel.activeSelf);
        ExpEventBus.Publish(ExpEvents.EvaluationEnd);
        slider.value = 0.5f;
    }
}
```

The `EvaluatePerformance` coroutine begins when the `EvaluationBegin` event is published. After the evaluation is finished, `EvaluationEnd` is published.

🔥🔥🔥 **So, one needs one line of code to publish or subscribe to an event!!** 🔥🔥🔥

#### Experiment Loop

After subscribing all the functions to the corresponding events, we can set up an experiment loop that will be started at the beginning of the study. You can find it in the [Experiment Manager](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/ExperimentManager.cs) script. For each trial we will do the following:

```csharp
// start UXF trial
trial.Begin();

// start instruction at the beginning of the trial
ExpEventBus.Publish(ExpEvents.InstStartBegin);

// wait for instruction to finish
yield return new WaitUntil(() => _isInstStartFinished);

// start action event
ExpEventBus.Publish(trial.number == 1 ? ExpEvents.PracticeBegin : ExpEvents.ActionBegin);

// wait for action to finish
yield return new WaitForSeconds(10f);
ExpEventBus.Publish(trial.number == 1 ? ExpEvents.PracticeEnd : ExpEvents.ActionEnd);

// start evaluation at the end of the trial
ExpEventBus.Publish(ExpEvents.EvaluationBegin);

// wait for evaluation to finish
yield return new WaitUntil(() => _isEvaluationFinished);

_isInstStartFinished = _isEvaluationFinished = false;

// end UXF trial
trial.End();
yield return null;
```

Note that the `EvaluationEnd` and `InstStartEnd` events are published in the other scripts. Once they are published, the boolean variables`_isInstStartFinished` and `_isEvaluationFinished` variables become `true` and the loop continues. Importantly, there are **no direct connections** between the [evaluation](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/Evaluation.cs)/[instruction](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/Instructions.cs) and the [experiment manager](https://github.com/vagechirkov/TemplateEventBusUXF/blob/main/Assets/Scripts/ExperimentManager.cs) scripts.

<figure>
<img src="/event-bus-with-uxf/game_sample.gif" alt="the experiment trial sample">
</figure>

### Materials and recourses

* Baron, D. (2021). Game Development Patterns with Unity 2021: Explore practical game development using software design patterns and best practices in Unity and C#, 2nd Edition (2nd ed. edition). Packt Publishing. [link]( https://www.packtpub.com/product/game-development-patterns-with-unity-2021-second-edition/9781800200814)
* Brookes, J., Warburton, M., Alghadier, M., Mon-Williams, M., & Mushtaq, F. (2020). Studying human behavior with virtual reality: The Unity Experiment Framework. Behavior Research Methods, 52(2), 455–463. https://doi.org/10.3758/s13428-019-01242-0; See on [GitHub](https://github.com/immersivecognition/unity-experiment-framework)
* [Events or UnityEvents?????????](https://youtu.be/oc3sQamIh-Q) by
  Jason Weimann
* Simple Boids (Flocks of Birds, Fish and Insects) by [Nicholas Veselov]( https://assetstore.unity.com/publishers/30428); See in [Unity asset store](https://assetstore.unity.com/packages/3d/characters/animals/simple-boids-flocks-of-birds-fish-and-insects-164188).

