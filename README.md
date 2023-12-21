# LLaMa2lang
This repository contains convenience scripts to finetune LLaMa2-7b for chat towards any language (that isn't English). The rationale behind this is that LLaMa2 is trained on primarily English data and while it works to some extent for other languages, its performance is poor compared to English.

# What it does
The process we follow to tune LLaMa2 for a specific language is as follows:

1. We use the [Open Assistant dataset](https://huggingface.co/datasets/OpenAssistant/oasst1) from Huggingface as our base instruct data.
2. The dataset is fully translated into a specific target language using [Helsinki-NLP's OPUS translation models](https://huggingface.co/Helsinki-NLP). This follows a two-step process:

    2.1 Try to translate from the source language specified in OASST1 to the desired target language using a model for that language pair.

    2.2 If there is no such model, try to translate from the source language to English first and then from English to the target language.
3. Load the translated OASST1 dataset and extract threads by recursively selecting prompts with their respective answers with the highest rank only, through to subsequent prompts, etc.
4. Turn the threads into texts using [LLaMa's prompt format](https://huggingface.co/blog/llama2#how-to-prompt-llama-2).
5. Use QLoRA and PEFT to finetune LLaMa2-chat on this dataset.

## Cost and runtime

The above process can be fully run on a free Google Colab T4 GPU. The last step however, can only be successfully run with short enough context windows and a batch of at most 2. In addition, the translation in step 2 takes about 36 hours in total for any given language so should be run in multiple steps if you want to stick with a free Google Colab GPU.

Our fine-tuned models for step 5 were performed using an A40 on [vast.ai](https://vast.ai/) and cost us less than a dollar for each model, completing in about 1.5 hours.

# Usage
1. Make sure pytorch is installed and working for your environment (use of CUDA preferable): https://pytorch.org/get-started/locally/

2. Clone the repo and install the requirements.

`pip install -r requirements.txt`

2. Translate the OASST1 dataset into your target language. This script writes out intermediate results to a `checkpoint_location` because its runtime is quite lengthy (about 30-40 hours on a T4 Google Colab GPU).

`python translate_oasst.py [TARGET_LANG] [CHECKPOINT_FOLDER] [CHECKPOINT_N]`

Parameters:

- `TARGET_LANG` The target language, use ISO language codes as used in the [Helsinki-NLP's OPUS translation models](https://huggingface.co/Helsinki-NLP).
- `CHECKPOINT_FOLDER` The folder the script will write (JSONized) checkpoint files to. Folder will be created if it doesn't exist.
- `CHECKPOINT_N` An integer representing how often a checkpoint file will be written out. OASST1 contains 84.4k records in train and another 4.4k records in validation. We found `200` to be a reasonable number for this parameter.

3. Combine the JSON arrays from the checkpoints' files into a Huggingface Dataset and then either write it to disk or publish it to Huggingface. The script will try to write to disk by default and fall back to publishing to Huggingface if the folder doesn't exist on disk. For publishing to Huggingface, make sure you have your `HF_TOKEN` environment variable set up as per [the documentation](https://huggingface.co/docs/huggingface_hub/package_reference/environment_variables#hftoken).

`python combine_checkpoints.py [CHECKPOINT_FOLDER] [OUTPUT_LOCATION]`

Parameters:

- `[CHECKPOINT_FOLDER]` The same checkpoint location as used in translation but now with the `[TARGET_LANG]` added to it. Example: checkpoint location used to write: `./checkpoints` for target language `nl` results in this script requiring `./checkpoints/nl`.
- [OUTPUT_LOCATION] Where to write the Huggingface Dataset to. Can either be a location on disk or a Hugginface Dataset repository if the location on disk does not exist. Be sure to set up `HF_TOKEN`.

4. Turn the translated dataset into threads in LLaMa2-chat format. We do this by always using the highest ranking answer following a given input prompt.

`python create_thread_prompts.py [INPUT_DATASET] [INSTRUCTION_PROMPT] [OUTPUT_DATASET]`

Parameters:

* `[INPUT_DATASET]` The input dataset, loaded from Huggingface datasets. This should be the result of the previous setp.
* `[INSTRUCTION_PROMPT]` An instruction message added to every prompt given to the chatbot to force it to answer in the target language. Should be something like this:
    * EN: You are a generic chatbot that always answers in English.
    * ES: Eres un chatbot genérico que siempre responde en español.
    * FR: Tu es un chatbot générique qui répond toujours en français.
    * NL: Je bent een generieke chatbot die altijd in het Nederlands antwoord geeft.
* `[OUTPUT_DATASET]` Where to write the Huggingface Dataset to. Can either be a location on disk or a Hugginface Dataset repository if the location on disk does not exist. Be sure to set up `HF_TOKEN`.

5. Fine-tune LLaMa2-7B-chat (or another base model) using LoRA and PEFT.

`python finetune.py [BASE_MODEL] [TUNED_MODEL] [DATASET_NAME]`

Parameters:

* `[BASE_MODEL]` The base foundation model. If you don't know which to use, we recommend [https://huggingface.co/NousResearch/Llama-2-7b-chat-hf](https://huggingface.co/NousResearch/Llama-2-7b-chat-hf).
* `[TUNED_MODEL]` The name of the resulting tuned model. This will be pushed to Huggingface directly. Make sure you have `HF_TOKEN` set as an environment variable.
* `[DATASET_NAME]` The name of the dataset to use for finetuning.

# Datasets and models

We have created and will continue to create numerous datasets and models already.

## Translated oasst1 datasets

- [UnderstandLing/oasst1_nl](https://huggingface.co/datasets/UnderstandLing/oasst1_nl) The oasst1 dataset translated to Dutch.
- [UnderstandLing/oasst1_es](https://huggingface.co/datasets/UnderstandLing/oasst1_es) The oasst1 dataset translated to Spanish.
- [UnderstandLing/oasst1_fr](https://huggingface.co/datasets/UnderstandLing/oasst1_fr) The oasst1 dataset translated to French.
- [UnderstandLing/oasst1_de](https://huggingface.co/datasets/UnderstandLing/oasst1_de) The oasst1 dataset translated to German.

## Translated LLaMa2 thread chat prompt datasets

- [UnderstandLing/oasst1_nl_threads](https://huggingface.co/datasets/UnderstandLing/oasst1_nl_threads) The LLaMa2 chat prompts with history from threads in oasst1 for Dutch.
- [UnderstandLing/oasst1_es_threads](https://huggingface.co/datasets/UnderstandLing/oasst1_es_threads) The LLaMa2 chat prompts with history from threads in oasst1 for Spanish.
- [UnderstandLing/oasst1_fr_threads](https://huggingface.co/datasets/UnderstandLing/oasst1_fr_threads) The LLaMa2 chat prompts with history from threads in oasst1 for French.
- [UnderstandLing/oasst1_de_threads](https://huggingface.co/datasets/UnderstandLing/oasst1_de_threads) The LLaMa2 chat prompts with history from threads in oasst1 for German.

## Language-specific LLaMa2-7B chat model adapters

- [UnderstandLing/llama-2-7b-chat-nl](https://huggingface.co/UnderstandLing/llama-2-7b-chat-nl) QLoRA adapter for LLaMa2-7b-chat in Dutch.
- [UnderstandLing/llama-2-7b-chat-es](https://huggingface.co/UnderstandLing/llama-2-7b-chat-es) QLoRA adapter for LLaMa2-7b-chat in Spanish.
- [UnderstandLing/llama-2-7b-chat-fr](https://huggingface.co/UnderstandLing/llama-2-7b-chat-fr) QLoRA adapter for LLaMa2-7b-chat in French.
- [UnderstandLing/llama-2-7b-chat-de](https://huggingface.co/UnderstandLing/llama-2-7b-chat-de) QLoRA adapter for LLaMa2-7b-chat in German.