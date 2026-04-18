# LLM Text Steganography

This project implements text steganography based on large language models (LLM), using the GPT-2 model to hide information in generated text. The project contains two encoding methods: Huffman Coding and Fixed Length Coding (FLC).

## Features

- Implements steganography in GPT-2 text generation
- Supports two encoding methods:
  - Huffman Coding
  - Fixed Length Coding (FLC)
- Generates steganographic text and normal text for comparison

## Environment Requirements

- Python 3.8+
- PyTorch
- Transformers
- CUDA (optional, for GPU acceleration)

## Installation Steps

Install required packages:

```bash
pip install torch # If using GPU, make sure to specify the GPU version of torch
pip install transformers
pip install jupyter
```

## Usage

1. Open the Jupyter notebook:
```bash
jupyter notebook llm_stega.ipynb
```

2. The notebook contains three main sections:
   - Huffman Encoding implementation
   - Fixed Length Coding (FLC) implementation
   - Text generation and steganography

3. Generate steganographic text:
   - Choose an encoding method (Huffman Coding or FLC)
   - Set parameters (k value for encoding)
   - Run the main function with custom prompts

Example code:
```python
# Initialize steganography handler
k = 2
handle = Huffman(k=2**k, bits=bits)
# Or use FLC
# handle = FLC(k=k, bits=bits)

# Generate text with steganography
prompt = "Hello, I'm"
stega_output, normal_output = generate_text_with_steganography(
    model, tokenizer, prompt, handle
)
```

## Output Files

The program generates two text files:
- `outputs-gpt2-stega.txt`: Steganographic text
- `outputs-gpt2-normal.txt`: Normal generated text without steganographic constraints

## Notes

- The GPT-2 model will be automatically downloaded on first run. In China, you can also pre-download the GPT-2 model via https://hf-mirror.com, or use other large models.
- Ensure sufficient disk space (model is approximately 500MB)
- It is recommended to use a GPU with CUDA support for better performance
- The length of hidden information will affect the length of generated text

## How It Works

1. Steganography process:
   - Convert secret information to a binary sequence
   - Use GPT-2 model to generate text
   - Embed information bits into generated text through the encoding method

2. Decoding process:
   - Use the same GPT-2 model and context
   - Extract hidden information bits from text through the decoding method
   - Convert the binary sequence back to the original information
