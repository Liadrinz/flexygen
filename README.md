# FlexyGen

English | [中文版](README-zh.md)

FlexyGen is an easy-to-use **controllable generation tool for LLMs**. Through the idea of ​​inversion of control (IoC), developers can inject a series of **triggers** into the model to control the model generation process. In the trigger, the generated content can be modified according to the current generation state of the model (currently only supports splicing new content after the current generated sentence).

## Installation

```shell
pip install flexygen
```

## Getting Started

Demo Code: [examples/emoji.py](examples/emoji.py)

### 0. Import Dependencies

```python
import random

from transformers import AutoTokenizer, AutoModelForCausalLM
from flexygen import FlexyGen, GenerationState
```

### 1. Load HF Tokenizer and Model

```python
tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen2.5-0.5B-Instruct")
model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen2.5-0.5B-Instruct")
```

### 2. Wrap the Model with FlexyGen Interface

```python
model = FlexyGen.wrap(model, tokenizer)
```

### 3. Inject a Splicer Trigger

Inject a splicer trigger named `emoji` into the model.

The trigger will be called after each token is generated by the model.

If the current sentence ends with `",", ".", "!", "?"`, the trigger returns a random emoji character and the emoji will be attached after the current sentence. Else the trigger returns `None` and the current sentence will not be modified.

`state` stores the current generation state. The current sentence can be accessed through `state.input_ids`.

```python
@model.splicer("emoji")
def emoji_trigger(state: GenerationState) -> bool:
    def random_emoji():
        ranges = [
            (0x1F600, 0x1F64F),
            (0x1F300, 0x1F5FF),
            (0x1F680, 0x1F6FF),
            (0x2700, 0x27BF),
        ]
        start, end = random.choice(ranges)
        code_point = random.randint(start, end)
        return chr(code_point)
    sentence = tokenizer.batch_decode(state.input_ids)[0].strip()
    if sentence.endswith((",", ".", "!", "?")):
        return random_emoji()  # Returns a random emoji character
```

### 4. Generate

```python
input_text = tokenizer.apply_chat_template([
    {"role": "user", "content": "Why the sky is blue?"},
], tokenize=False, add_generation_prompt=True)
inputs = tokenizer(input_text, return_tensors="pt").to(model.device)
outputs = model.generate(**inputs, do_sample=True, max_new_tokens=128)
# Emoji attached after each sentence.
print(tokenizer.batch_decode(outputs)[0])
```

## Access other contents in `state`

In addition to `state.input_ids`, `state` can also access other contents:

- `state.model_kwargs`: model keyword arguments, including `attention_mask`, `past_key_values` and other keyword args passed to the model
- `state.current_length`: current sentence length (number of tokens)
- `state.next_tokens`: token currently generated by the model, equivalent to `state.input_ids[:, -1]`
- `state.next_token_logits`: logits of the current token
- `state.next_token_scores`: The scores of the current token (logits processed by logit_processor)

If `return_dict_in_generate=True` and `output_scores=True` are specified in the `generate()` method, the `state.scores`, that is, the scores of all tokens, can be accessed in the trigger.

If `return_dict_in_generate=True` and `output_logits=True` are specified in the `generate()` method, the `state.raw_logits`, that is, the attention weights of the model, can be accessed in the trigger.

If `return_dict_in_generate=True` and `output_attentions=True` are specified in the `generate()` method, `state.decoder_attentions` and `state.cross_attentions` can be accessed in the trigger, which are the attention weights and cross attention weights of the decoder (only for encoder-decoder architecture), respectively

If `return_dict_in_generate=True` and `output_hidden_states=True` are specified in the `generate()` method, `state.decoder_hidden_states` can be accessed in the trigger, which represents the hidden state vector output by each Transformer layer of the decoder.

## Using `SentenceLevelFlexyGen`

In some applications, it is necessary to trigger certain calls based on the probability of a sentence or the probability of certain tokens in a sentence. For example, adaptive **RAG** ​​will determine the probability of a token in a sentence to decide whether to trigger a retrieval.

At this time, you can use `SentenceLevelFlexyGen`:

```python

# ... Omitting model definitions

from flexygen import SentenceLevelFlexyGen, SentenceLevelGenerationState


model = SentenceLevelFlexyGen.wrap(model, tokenizer)


@model.splicer("prob")
def prob_trigger(state: SentenceLevelGenerationState):
    if state.end_of_sentences[0]:
        if min(state.sentence_token_probs[0]) < 0.1:
            # Returns reflection words when the minimum 
            # token probability of a sentence is lower than 0.1
            return " ... Wait, I'm not sure. "


# ... Omitting generation
```

When using `SentenceLevelFlexyGen`, `state` in the trigger becomes a `SentenceLevelGenerationState` object, which has more content than `GenerationState`:

- `state.end_of_sentences: List[bool]`: a list of Boolean variables indicating whether each output in a batch generates a complete sentence (by default, ending with the six characters `.?!.?!` indicates that a sentence has been generated)
- `state.sentence_tokens: List[List[int]]`: the current sentence
- `state.sentence_token_probs: List[List[int]]`: the probability of each token in the current sentence
