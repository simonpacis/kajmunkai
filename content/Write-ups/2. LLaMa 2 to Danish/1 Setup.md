---
title: Setup
---
To be able to communicate satisfactorily with LLaMa 2 in Danish, we need to train it to be conversational in Danish. While LLaMa 2 and other such models have been trained on Danish data, they have not been trained to be conversational in it.

In fact, of LLaMa 2's dataset, [only 0.2% of it is in Danish](https://heidloff.net/article/llm-languages-german/#:~:text=LLaMA%202%20has%20been%20trained,as%20out%2Dof%2Dscope.). This is most likely not sufficient for us to get consistently useful responses in Danish. As such, any fine-tuning of a model to operate on a Danish dataset must necessarily first include a fine-tuning of the model to be conversational in Danish.

We might naturally presume that this requires us to write a dataset in Danish which corresponds to the English dataset used. This would be an insurmountably large task for our small team. Instead, we will try to leverage the translation proficiency of LLM's to translate the dataset for us, before we fine-tune LLaMa 2 on it. This approach should in theory be applicable to other models, but has been proven to work for LLaMa 2, which is why we will start here.

Here is an outline of the steps we will be taking:

1. Translate an English dataset to Danish
	- Dataset used will be [OASST2](https://huggingface.co/datasets/OpenAssistant/oasst2), as it is a dataset specifically made for conversations.
	- We will be using Google's [MADLAD](https://huggingface.co/google/madlad400-10b-mt) translation model to perform the translation.
2. Combine checkpoints into a single dataset
	- The output of the translation will give us multiple JSON-files containing the translated dataset. We simply need to combine them into a single file on which the dataset can be fine-tuned.
3. Fine-tune LLaMa 2
	1. This is a two-step process. First, we will fine-tune it using (Q)LoRA and PEFT. Both of these approaches emphasize efficiency in fine-tuning, and may not necessarily deliver the best results. A good starting point.
		- We will run benchmarks here.
	2. Subsequently, we will fine-tune using DPO. DPO, or Decision Process Optimization, should specifically help the model in its ability to generate more coherent and contextually appropriate responses.
		- We will again run benchmarks here.
1. Profit!
	- We should be able to chat with the model in its final form here. Running benchmarks on it will give us an idea of improvement of the model, and whether further fine-tuning is required as pertains to conversing in Danish.