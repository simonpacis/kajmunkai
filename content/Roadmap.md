# Roadmap

For reasons outlined below, the plan is to try our hand first with Meta's LLaMa2. [Click to experiment with the model in its non-fine-tuned state](https://www.llama2.ai).

## Fine-tuning model to Danish
The first step on the road to a functioning Kaj Munk AI is to ensure that we can chat with it in Danish. There are currently no up-to-date models that have been trained to do so. They might have been trained on Danish material, so that they understand and can translate to and from Danish, but they are trained to converse in English. As such, we need to fine-tune it to expect Danish.

For the [LLaMa2](https://llama.meta.com) model this can be done without manually needing to create training data, by way of UnderstandLingBV's project [`LLaMa2lang`](https://github.com/understandlingbv/llama2lang). Here we use AI to translate the [OASST2 dataset (Open Assistant Conversations Dataset Release 2)](https://huggingface.co/datasets/OpenAssistant/oasst2) into Danish, and then we fine-tune the model using the translated dataset. According to UnderstandLingBV, the performance of the model for intelligibly conversing in Danish should increase substantially by doing this. This has not been attempted for Danish yet, but has been proven successful for Dutch. This will be our first step.

## Fine-tuning model to Kaj Munk
This is the end goal. I cannot at this time outline the concrete steps to take to get there. We are aiming for an LLM which can function as an "expert" on all matters Kaj Munk, but potentially also be used for text classification to generate metadata for the digital archive.