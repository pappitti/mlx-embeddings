# MLX-Embeddings

[![image](https://img.shields.io/pypi/v/mlx-embeddings.svg)](https://pypi.python.org/pypi/mlx-embeddings)

**MLX-Embeddings is a package for running Vision and Language Embedding models locally on your Mac using MLX.**

- Free software: GNU General Public License v3

## Features

- Generate embeddings for text and images using MLX models
- Support for single-item and batch processing
- Utilities for comparing text similarities

## Supported Models Archictectures
MLX-Embeddings supports a variety of model architectures for text embedding tasks. Here's a breakdown of the currently supported architectures:
- XLM-RoBERTa (Cross-lingual Language Model - Robustly Optimized BERT Approach)
- BERT (Bidirectional Encoder Representations from Transformers)
- ModernBERT (modernized bidirectional encoder-only Transformer model)

We're continuously working to expand our support for additional model architectures. Check our GitHub repository or documentation for the most up-to-date list of supported models and their specific versions.

## Installation

You can install mlx-embeddings using pip:

```bash
pip install mlx-embeddings
```

## Usage

### Single Item Embedding


#### Text Embedding
To generate an embedding for a single piece of text:

```python
from mlx_embeddings.utils import load

# Load the model and tokenizer
model_name = "mlx-community/all-MiniLM-L6-v2-4bit"
model, tokenizer = load(model_name)

# Prepare the text
text = "I like reading"

# Tokenize and generate embedding
input_ids = tokenizer.encode(text, return_tensors="mlx")
outputs = model(input_ids)
raw_embeds = outputs.last_hidden_state[:, 0, :] # CLS token
text_embeds = outputs.text_embeds # mean pooled and normalized embeddings
```

Note : text-embeds use mean pooling for bert and xlm-robert. For modernbert, pooling strategy is set through the config file, defaulting to mean

#### Masked Language Modeling

To generate embeddings for masked language modeling tasks:

```python
from mlx_embeddings.utils import load

# Load ModernBERT model and tokenizer
model, tokenizer = load("mlx-community/answerdotai-ModernBERT-base-4bit")

# Masked Language Modeling example
text = "The capital of France is [MASK]."
inputs = tokenizer.encode(text, return_tensors="mlx")
outputs = model(inputs)

# Get predictions for the masked token
masked_index = inputs.tolist()[0].index(tokenizer.mask_token_id)
predicted_token_id = mx.argmax(outputs.pooler_output[0, masked_index]).tolist()
predicted_token = tokenizer.decode(predicted_token_id)
print("Predicted token:", predicted_token)  # Should output: Paris
```

#### Sequence classification
```python
from mlx_embeddings.utils import load

# Load ModernBERT model and tokenizer
model, tokenizer = load(
    "NousResearch/Minos-v1",
)

id2label=model.config.id2label

# Masked Language Modeling example
text = "<|user|> Explain the theory of relativity in simple terms. <|assistant|> Imagine space and time are like a stretchy fabric. Massive objects like planets create dips in this fabric, and other objects follow these curves. That's gravity! Also, the faster you move, the slower time passes for you compared to someone standing still"
inputs = tokenizer.encode(text, return_tensors="mlx")
outputs = model(inputs)

# Get predictions for the masked token
predictions = outputs.pooler_output[0] # Shape: (num_label,)
print(text)

# Print results
print("\nTop predictions for classification:")
for idx, logit in enumerate(predictions.tolist()):
    label = id2label[str(idx)]
    print(f"{label}: {logit:.3f}")
```

### Batch Processing

#### Multiple Texts Comparison

To embed multiple texts and compare them using their embeddings:

```python
from sklearn.metrics.pairwise import cosine_similarity
import matplotlib.pyplot as plt
import seaborn as sns
import mlx.core as mx
from mlx_embeddings.utils import load

# Load the model and tokenizer
model_name ="mlx-community/all-MiniLM-L6-v2-4bit"
model, tokenizer = load(model_name)

def get_embedding(texts, model, tokenizer):
    max_position_embeddings = getattr(model.config,"max_position_embeddings",512)
    inputs = tokenizer.batch_encode_plus(
        texts, return_tensors="mlx", 
        padding=True, truncation=True, 
        max_length=max_position_embeddings)
    outputs = model(
        inputs["input_ids"],
        attention_mask=inputs["attention_mask"]
    )
    return outputs.text_embeds.astype(mx.float32) # np (used by sklearn, can't process bfloat16)

def compute_and_print_similarity(embeddings):
    B, _ = embeddings.shape
    similarity_matrix = cosine_similarity(embeddings)
    print("Similarity matrix between sequences:")
    print(similarity_matrix)
    print("\n")

    for i in range(B):
        for j in range(i+1, B):
            print(f"Similarity between sequence {i+1} and sequence {j+1}: {similarity_matrix[i][j]:.4f}")

    return similarity_matrix

# Visualize results
def plot_similarity_matrix(similarity_matrix, labels):
    plt.figure(figsize=(5, 4))
    sns.heatmap(similarity_matrix, annot=True, cmap='coolwarm', xticklabels=labels, yticklabels=labels)
    plt.title(f'Similarity Matrix Heatmap - {model_name}')
    plt.tight_layout()
    plt.show()

# Sample texts
texts = [
    "I like grapes",
    "I like fruits",
    "The slow green turtle crawls under the busy ant."
]

embeddings = get_embedding(texts, model, tokenizer)
similarity_matrix = compute_and_print_similarity(embeddings)

# Visualize results
labels = [f"Text {i+1}" for i in range(len(texts))]
plot_similarity_matrix(similarity_matrix, labels)
```

#### Masked Language Modeling

To get predictions for the masked token in multiple texts:

```python
import mlx.core as mx
from mlx_embeddings.utils import load

# Load the model and tokenizer
model, tokenizer = load("mlx-community/answerdotai-ModernBERT-base-4bit")

max_position_embeddings = getattr(model.config,"max_position_embeddings",512)

text = ["The capital of France is [MASK].", "The capital of Poland is [MASK]."]
inputs = tokenizer.batch_encode_plus(text, return_tensors="mlx", padding=True, truncation=True, max_length=max_position_embeddings )
outputs = model(**inputs)

# To get predictions for the mask:
# Find mask token indices for each sequence in the batch
# Find mask indices for all sequences in batch
mask_indices = mx.array([ids.tolist().index(tokenizer.mask_token_id) for ids in inputs["input_ids"]])

# Get predictions for all masked tokens at once
batch_indices = mx.arange(len(mask_indices))
predicted_token_ids = mx.argmax(outputs.pooler_output[batch_indices, mask_indices], axis=-1).tolist()

# Decode the predicted tokens
predicted_token = tokenizer.batch_decode(predicted_token_ids)

print("Predicted token:", predicted_token)
# Predicted token:  Paris, Warsaw
```


## Vision Transformer Models

MLX-Embeddings also supports vision models that can generate embeddings for images or image-text pairs.

### Single Image Processing

To evaluate how well an image matches different text descriptions:

```python
import mlx.core as mx
from mlx_embeddings.utils import load
import requests
from PIL import Image

# Load vision model and processor
model, processor = load("mlx-community/siglip-so400m-patch14-384")

# Load an image
url = "http://images.cocodataset.org/val2017/000000039769.jpg"
image = Image.open(requests.get(url, stream=True).raw)

# Create text descriptions to compare with the image
texts = ["a photo of 2 dogs", "a photo of 2 cats"]

# Process inputs
inputs = processor(text=texts, images=image, padding="max_length", return_tensors="pt")
pixel_values = mx.array(inputs.pixel_values).transpose(0, 2, 3, 1).astype(mx.float32)
input_ids = mx.array(inputs.input_ids)

# Generate embeddings and calculate similarity
outputs = model(pixel_values=pixel_values, input_ids=input_ids)
logits_per_image = outputs.logits_per_image
probs = mx.sigmoid(logits_per_image)  # probabilities of image matching each text

# Print results
print(f"{probs[0][0]:.1%} that image matches '{texts[0]}'")
print(f"{probs[0][1]:.1%} that image matches '{texts[1]}'")
```

### Batch Processing for Multiple Images comparison

Process multiple images and compare them with text descriptions:

```python
import mlx.core as mx
from mlx_embeddings.utils import load
import requests
from PIL import Image
import matplotlib.pyplot as plt
import seaborn as sns

# Load vision model and processor
model, processor = load("mlx-community/siglip-so400m-patch14-384")

# Load multiple images
image_urls = [
    "./images/cats.jpg",  # cats
    "./images/desktop_setup.png"   # desktop setup
]
images = [Image.open(requests.get(url, stream=True).raw) if url.startswith("http") else Image.open(url) for url in image_urls]

# Text descriptions
texts = ["a photo of cats", "a photo of a desktop setup", "a photo of a person"]

# Process all image-text pairs
all_probs = []


# Process all image-text pairs in batch
inputs = processor(text=texts, images=images, padding="max_length", return_tensors="pt")
pixel_values = mx.array(inputs.pixel_values).transpose(0, 2, 3, 1).astype(mx.float32)
input_ids = mx.array(inputs.input_ids)

# Generate embeddings and calculate similarity
outputs = model(pixel_values=pixel_values, input_ids=input_ids)
logits_per_image = outputs.logits_per_image
probs = mx.sigmoid(logits_per_image) # probabilities for this image
all_probs.append(probs.tolist())


# Print results for this image
for i, image in enumerate(images):
    print(f"Image {i+1}:")
    for j, text in enumerate(texts):
        print(f"  {probs[i][j]:.1%} match with '{text}'")
    print()

# Visualize results with a heatmap
def plot_similarity_matrix(probs_matrix, image_labels, text_labels):
    # Convert to 2D numpy array if needed
    import numpy as np
    probs_matrix = np.array(probs_matrix)

    # Ensure we have a 2D matrix for the heatmap
    if probs_matrix.ndim > 2:
        probs_matrix = probs_matrix.squeeze()

    plt.figure(figsize=(8, 5))
    sns.heatmap(probs_matrix, annot=True, cmap='viridis',
                xticklabels=text_labels, yticklabels=image_labels,
                fmt=".1%", vmin=0, vmax=1)
    plt.title('Image-Text Match Probability')
    plt.tight_layout()
    plt.show()

# Plot the images for reference
plt.figure(figsize=(8, 5))
for i, image in enumerate(images):
    plt.subplot(1, len(images), i+1)
    plt.imshow(image)
    plt.title(f"Image {i+1}")
    plt.axis('off')
plt.tight_layout()
plt.show()

image_labels = [f"Image {i+1}" for i in range(len(images))]
plot_similarity_matrix(all_probs, image_labels, texts)
```

### Late Interaction Multimodal Retrival Models (ColPali/ColQwen)

```python
import mlx.core as mx
from mlx_embeddings.utils import load
import requests
from PIL import Image
import torch

# Load vision model and processor
model, processor = load("qnguyen3/colqwen2.5-v0.2-mlx")

# Load an image

url_1 = "https://upload.wikimedia.org/wikipedia/commons/8/89/US-original-Declaration-1776.jpg"
image_1 = Image.open(url_1)

url_2 = "https://upload.wikimedia.org/wikipedia/commons/thumb/4/4c/Romeoandjuliet1597.jpg/500px-Romeoandjuliet1597.jpg"
image_2 = Image.open(url_2)

# Create text descriptions to compare with the image
texts = ["how many percent of data are books?", "evaluation results between models"]

# Process inputs - text and images need to be processed separately for ColQwen2.5
text_inputs = processor(text=texts, padding=True, return_tensors="pt")
image_inputs = processor(images=[image_1, image_2], padding=True, return_tensors="pt")

# Convert to MLX arrays
text_input_ids = mx.array(text_inputs.input_ids)

image_input_ids = mx.array(image_inputs.input_ids)
pixel_values = mx.array(image_inputs.pixel_values)
image_grid_thw = mx.array(image_inputs.image_grid_thw)

text_embeddings = model(input_ids=text_input_ids)
image_embeddings = model(
    input_ids=image_input_ids, 
    pixel_values=pixel_values, 
    image_grid_thw=image_grid_thw,
)

print(text_embeddings.text_embeds.shape)
print(image_embeddings.image_embeds.shape)

## convert to torch
import torch
text_embeddings = torch.tensor(text_embeddings.text_embeds)
image_embeddings = torch.tensor(image_embeddings.image_embeds)

scores = processor.score_retrieval(text_embeddings, image_embeddings)
print(scores)
```

## Contributing

Contributions to MLX-Embeddings are welcome! Please refer to our contribution guidelines for more information.

## License

This project is licensed under the GNU General Public License v3.

## Contact

For any questions or issues, please open an issue on the [GitHub repository](https://github.com/Blaizzy/mlx-embeddings).
