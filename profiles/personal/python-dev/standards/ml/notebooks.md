## Jupyter Notebook Best Practices

### Notebook Philosophy
- **Interactive Development**: Use notebooks for exploration, experimentation, and visualization
- **Production Code Separate**: Move production code to `.py` modules
- **Document Workflow**: Notebooks should tell a story with code, text, and visualizations
- **Version Control Friendly**: Keep notebooks clean and focused

### Notebook Structure
- **Linear Flow**: Organize cells in logical, top-to-bottom order
- **Clear Sections**: Use markdown headers to organize notebook into sections
- **Cell Purpose**: Each cell should have a single, clear purpose
- **Restart & Run All**: Notebooks should work when run from top to bottom

```markdown
# Project Title

## Setup
- Import libraries
- Load configuration
- Initialize connections

## Data Loading
- Load data sources
- Initial exploration

## Analysis
- Data transformation
- Model training
- Evaluation

## Results
- Visualizations
- Conclusions
```

### Imports and Setup
- **Top of Notebook**: All imports at the top in first code cell
- **Grouped Imports**: Group standard library, third-party, and local imports
- **Configuration Cell**: Dedicated cell for configuration and constants

```python
# Standard library
import os
from pathlib import Path
from typing import List, Dict

# Third-party libraries
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
from tqdm import tqdm

# LLM and ML libraries
import openai
from anthropic import Anthropic
from langchain.chains import LLMChain
from sentence_transformers import SentenceTransformer

# Local imports
from src.utils import load_data, save_results

# Configuration
%load_ext autoreload
%autoreload 2

# Display settings
pd.set_option('display.max_columns', None)
plt.style.use('seaborn-v0_8-darkgrid')

# Constants
DATA_DIR = Path("data")
MODELS_DIR = Path("models")
OUTPUT_DIR = Path("output")
```

### Environment Configuration
- **dotenv**: Use python-dotenv for secrets and config
- **Inline Config**: Keep simple config in notebook for visibility
- **No Hardcoded Secrets**: Never commit API keys

```python
from dotenv import load_dotenv
import os

# Load environment variables
load_dotenv()

# API Configuration
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY")
ANTHROPIC_API_KEY = os.getenv("ANTHROPIC_API_KEY")

# Model Configuration
LLM_MODEL = "gpt-4"
TEMPERATURE = 0.7
MAX_TOKENS = 1000
```

### Data Loading and Exploration
- **Load Early**: Load data early in notebook
- **Display Samples**: Show data samples after loading
- **Check Shape**: Always check data shape and types
- **Quick EDA**: Perform quick exploratory data analysis

```python
# Load data
df = pd.read_csv(DATA_DIR / "dataset.csv")

# Display info
print(f"Dataset shape: {df.shape}")
print(f"Columns: {df.columns.tolist()}")

# Display sample
display(df.head())

# Quick statistics
display(df.describe())

# Check for missing values
print("\nMissing values:")
print(df.isnull().sum())
```

### Progress Tracking
- **tqdm for Loops**: Use tqdm for progress bars
- **Informative Messages**: Print clear status messages
- **Intermediate Results**: Save and display intermediate results

```python
from tqdm.notebook import tqdm

results = []
for item in tqdm(data, desc="Processing items"):
    result = process_item(item)
    results.append(result)

print(f"Processed {len(results)} items successfully")
```

### LLM Calls in Notebooks
- **Async Support**: Use async functions when possible
- **Error Handling**: Handle API errors gracefully
- **Token Tracking**: Track token usage
- **Cache Results**: Cache expensive LLM calls

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI(api_key=OPENAI_API_KEY)

async def generate_completion(prompt: str) -> str:
    """Generate completion with error handling."""
    try:
        response = await client.chat.completions.create(
            model="gpt-4",
            messages=[{"role": "user", "content": prompt}],
            max_tokens=1000
        )
        return response.choices[0].message.content
    except Exception as e:
        print(f"Error: {e}")
        return None

# Run async function in notebook
result = await generate_completion("Explain quantum computing")
print(result)

# For batch processing with progress
prompts = ["prompt 1", "prompt 2", "prompt 3"]
results = []

for prompt in tqdm(prompts, desc="Generating completions"):
    result = await generate_completion(prompt)
    results.append(result)
```

### Visualization Best Practices
- **Inline Plots**: Use %matplotlib inline or %matplotlib widget
- **Clear Labels**: Always label axes and add titles
- **Consistent Style**: Use consistent plotting style
- **Save Figures**: Save important figures to files

```python
import matplotlib.pyplot as plt
import plotly.express as px

# Matplotlib configuration
%matplotlib inline
plt.rcParams['figure.figsize'] = (12, 6)
plt.rcParams['font.size'] = 12

# Example plot
plt.figure(figsize=(10, 6))
plt.plot(x_data, y_data, linewidth=2, label='Data')
plt.xlabel('X Axis Label')
plt.ylabel('Y Axis Label')
plt.title('Clear Descriptive Title')
plt.legend()
plt.grid(True, alpha=0.3)
plt.tight_layout()

# Save figure
plt.savefig(OUTPUT_DIR / 'plot.png', dpi=300, bbox_inches='tight')
plt.show()

# Interactive plots with Plotly
fig = px.scatter(df, x='column1', y='column2', color='category',
                 title='Interactive Scatter Plot')
fig.show()
```

### Interactive Widgets
- **ipywidgets**: Use widgets for interactive parameters
- **Gradio in Notebooks**: Embed Gradio interfaces
- **Real-time Updates**: Create interactive dashboards

```python
import ipywidgets as widgets
from IPython.display import display

# Slider widget
temperature_slider = widgets.FloatSlider(
    value=0.7,
    min=0.0,
    max=2.0,
    step=0.1,
    description='Temperature:',
    style={'description_width': 'initial'}
)

# Dropdown widget
model_dropdown = widgets.Dropdown(
    options=['gpt-4', 'gpt-3.5-turbo', 'claude-3-opus'],
    value='gpt-4',
    description='Model:',
)

# Interactive function
def generate_text(model, temperature):
    print(f"Generating with {model} at temperature {temperature}")
    # Your generation logic here

# Connect widgets
widgets.interactive(generate_text, model=model_dropdown, temperature=temperature_slider)
```

### Gradio in Notebooks
- **Quick Interfaces**: Create quick UIs for testing
- **share=True**: Share demos temporarily
- **Inline Display**: Display Gradio in notebook

```python
import gradio as gr

def predict(text, model_name, temperature):
    """Prediction function for Gradio interface."""
    # Your LLM logic here
    response = llm_client.generate(
        prompt=text,
        model=model_name,
        temperature=temperature
    )
    return response

# Create Gradio interface
demo = gr.Interface(
    fn=predict,
    inputs=[
        gr.Textbox(lines=5, label="Input Text"),
        gr.Dropdown(["gpt-4", "claude-3-opus"], label="Model"),
        gr.Slider(0, 2, value=0.7, label="Temperature")
    ],
    outputs=gr.Textbox(label="Output"),
    title="LLM Demo",
    description="Test LLM models interactively"
)

# Launch in notebook
demo.launch(inline=True, share=False)
```

### Cell Output Management
- **Clear Output**: Clear unnecessary output before committing
- **Suppress Output**: Use semicolon to suppress output
- **Display Control**: Use IPython display functions

```python
# Suppress output with semicolon
large_dataframe.head();

# Clear display
from IPython.display import clear_output

for i in range(100):
    clear_output(wait=True)
    print(f"Processing: {i}%")

# Formatted display
from IPython.display import display, HTML, Markdown

display(Markdown("## Results"))
display(HTML(df.to_html()))
```

### Model Training in Notebooks
- **Checkpoints**: Save model checkpoints regularly
- **Metrics Tracking**: Track and display training metrics
- **Early Stopping**: Implement early stopping for long training

```python
from transformers import Trainer, TrainingArguments

# Training configuration
training_args = TrainingArguments(
    output_dir=str(MODELS_DIR / "checkpoints"),
    num_train_epochs=3,
    per_device_train_batch_size=8,
    save_steps=500,
    save_total_limit=2,
    logging_steps=100,
    evaluation_strategy="steps",
    eval_steps=500,
)

# Train model
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=eval_dataset,
)

# Train with progress
trainer.train()

# Save final model
trainer.save_model(str(MODELS_DIR / "final_model"))
```

### Working with Embeddings
- **Batch Processing**: Process embeddings in batches
- **Dimension Reduction**: Visualize embeddings with UMAP/t-SNE
- **Storage**: Save embeddings for reuse

```python
from sentence_transformers import SentenceTransformer
import numpy as np

# Load embedding model
embedding_model = SentenceTransformer('all-MiniLM-L6-v2')

# Generate embeddings in batches
texts = df['text'].tolist()
batch_size = 32

embeddings = embedding_model.encode(
    texts,
    batch_size=batch_size,
    show_progress_bar=True,
    convert_to_numpy=True
)

# Save embeddings
np.save(OUTPUT_DIR / 'embeddings.npy', embeddings)

# Load embeddings
loaded_embeddings = np.load(OUTPUT_DIR / 'embeddings.npy')

# Visualize with UMAP
import umap
import plotly.express as px

reducer = umap.UMAP(n_components=2)
embedding_2d = reducer.fit_transform(embeddings)

fig = px.scatter(
    x=embedding_2d[:, 0],
    y=embedding_2d[:, 1],
    color=df['category'],
    hover_data=[df['text'].str[:50]],
    title='Text Embeddings Visualization'
)
fig.show()
```

### Saving and Loading Results
- **Pickle for Python Objects**: Use pickle for Python objects
- **CSV/Parquet for DataFrames**: Use appropriate format for data
- **JSON for Metadata**: Use JSON for configuration and metadata

```python
import pickle
import json

# Save Python objects
with open(OUTPUT_DIR / 'results.pkl', 'wb') as f:
    pickle.dump(results, f)

# Load Python objects
with open(OUTPUT_DIR / 'results.pkl', 'rb') as f:
    loaded_results = pickle.load(f)

# Save DataFrame
df.to_parquet(OUTPUT_DIR / 'processed_data.parquet')
df.to_csv(OUTPUT_DIR / 'processed_data.csv', index=False)

# Save metadata
metadata = {
    'model': 'gpt-4',
    'temperature': 0.7,
    'timestamp': str(datetime.now()),
    'num_samples': len(results)
}

with open(OUTPUT_DIR / 'metadata.json', 'w') as f:
    json.dump(metadata, f, indent=2)
```

### Version Control Best Practices
- **Clear Before Commit**: Clear output before committing
- **.gitignore**: Ignore checkpoint files and large outputs
- **nbstripout**: Use nbstripout to auto-clear outputs
- **Meaningful Names**: Use descriptive notebook names

```bash
# Install nbstripout
pip install nbstripout

# Configure git to auto-clear outputs
nbstripout --install

# .gitignore for notebooks
*.ipynb_checkpoints
output/
models/checkpoints/
*.pkl
*.npy
```

### Google Colab Compatibility
- **GPU Access**: Enable GPU in Colab settings
- **Install Dependencies**: Install required packages at top
- **Mount Drive**: Mount Google Drive for data access
- **Save to Drive**: Save outputs to Drive

```python
# Check if running in Colab
try:
    import google.colab
    IN_COLAB = True
except:
    IN_COLAB = False

if IN_COLAB:
    # Mount Google Drive
    from google.colab import drive
    drive.mount('/content/drive')

    # Install dependencies
    !pip install -q openai anthropic langchain chromadb

    # Set data paths
    DATA_DIR = Path('/content/drive/MyDrive/data')
    OUTPUT_DIR = Path('/content/drive/MyDrive/output')

    # Check GPU
    !nvidia-smi
else:
    # Local paths
    DATA_DIR = Path('data')
    OUTPUT_DIR = Path('output')
```

### Testing Notebooks
- **pytest-nbval**: Test notebook execution
- **papermill**: Parameterize and execute notebooks
- **Smoke Tests**: Quick tests to ensure notebook runs

```python
# Using papermill to execute notebook programmatically
import papermill as pm

pm.execute_notebook(
    'analysis.ipynb',
    'output/analysis_executed.ipynb',
    parameters=dict(
        model_name='gpt-4',
        temperature=0.7,
        max_tokens=1000
    )
)
```

### Documentation
- **Markdown Cells**: Use markdown for explanations
- **Code Comments**: Comment complex logic
- **Results Interpretation**: Explain results and findings
- **Next Steps**: Document next steps and TODOs

```markdown
## Analysis Results

The model achieved the following metrics:
- Accuracy: 94.5%
- F1 Score: 0.92
- Inference Time: 150ms per sample

### Key Findings
1. The model performs best on shorter texts (< 500 tokens)
2. Adding context improves accuracy by 8%
3. Temperature of 0.7 provides good balance between creativity and coherence

### Next Steps
- [ ] Test on larger dataset
- [ ] Implement caching for frequently used prompts
- [ ] Deploy as FastAPI endpoint
```

### Performance Optimization
- **Profile Code**: Use %%time and %% timeit
- **Memory Management**: Monitor memory usage
- **Batch Operations**: Process data in batches

```python
# Time cell execution
%%time
result = expensive_operation()

# Time single line
%timeit fast_operation()

# Memory profiling
%load_ext memory_profiler
%memit large_array_operation()

# Monitor GPU memory (if using PyTorch)
import torch
print(f"GPU Memory: {torch.cuda.memory_allocated() / 1024**3:.2f} GB")
```
