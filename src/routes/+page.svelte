<!-- src/routes/index.svelte -->
<script>
  import { onMount } from 'svelte';

  let apiKey = '';
  let prompt = 'dog wearing black glasses';
  let generatedImages = [];
  let selectedImage = null;
  let isGenerating = false;
  let errorMessage = null;
  let model = 'sd3';
  let aspectRatio = '1:1';
  let styles = [];
  let selectedStyle = null;
  let inputImage = null;
  let strength = 0.5;

  const aspectRatioOptions = [
    '16:9',
    '1:1',
    '21:9',
    '2:3',
    '3:2',
    '4:5',
    '5:4',
    '9:16',
    '9:21',
  ];

  onMount(async () => {
    apiKey = localStorage.getItem('APIKey') || '';
    await fetchStyles();
  });

  async function fetchStyles() {
    try {
      const response = await fetch('/sdxl_styles_sai.json');
      styles = await response.json();
      selectedStyle = styles[0];
    } catch (error) {
      console.error('Error fetching styles:', error);
    }
  }

  function saveApiKey() {
    localStorage.setItem('APIKey', apiKey);
  }

  function handleImageUpload(event) {
    inputImage = event.target.files[0];
  }

  async function generateImage() {
    isGenerating = true;
    errorMessage = null;

    const url = 'https://api.stability.ai/v2beta/stable-image/generate/sd3';
    const formData = new FormData();
    const styledPrompt = selectedStyle.prompt.replace('{prompt}', prompt);
    formData.append('prompt', styledPrompt);
    if (model === 'sd3') {
      formData.append('negative_prompt', selectedStyle.negative_prompt);
    }
    formData.append('output_format', 'jpeg');
    formData.append('model', model);
    if (inputImage) {
      formData.append('image', inputImage);
      formData.append('strength', strength.toString());
      formData.append('mode', 'image-to-image');
    }else {
      formData.append('aspect_ratio', aspectRatio);
    }


    try {
      const response = await fetch(url, {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${apiKey}`,
          Accept: 'image/*',
        },
        body: formData,
      });

      if (response.ok) {
        const blob = await response.blob();
        const imageUrl = URL.createObjectURL(blob);
        const imageName = `image_${Date.now()}.jpeg`;
        generatedImages = [{ name: imageName, url: imageUrl, thumbnail: imageUrl }, ...generatedImages];
        selectedImage = imageUrl;
      } else {
        console.log(await response.json());
        throw new Error(`Error ${response.status}`);
      }
    } catch (error) {
      console.error('Error generating image:', error);
      
      errorMessage = error.message;
    } finally {
      isGenerating = false;
    }
  }

  function selectImage(imageUrl) {
    selectedImage = imageUrl;
  }

  function downloadImage(imageUrl, imageName) {
    const link = document.createElement('a');
    link.href = imageUrl;
    link.download = imageName;
    link.click();
  }
</script>

<div class="app">
  <div class="sidebar">
    <div class="form-group">
      <label for="api-key">API Key:</label>
      <input type="text" id="api-key" bind:value={apiKey} on:input={saveApiKey} />
    </div>
    <div class="form-group">
      <label for="prompt">Prompt:</label>
      <textarea id="prompt" bind:value={prompt} rows="4"></textarea>
    </div>
    <div class="form-group">
      <label for="model">Model:</label>
      <select id="model" bind:value={model}>
        <option value="sd3">sd3</option>
        <option value="sd3-turbo">sd3-turbo</option>
      </select>
    </div>
    <div class="form-group">
      <label for="aspect-ratio">Aspect Ratio:</label>
      <select id="aspect-ratio" bind:value={aspectRatio}>
        {#each aspectRatioOptions as option}
          <option value={option}>{option}</option>
        {/each}
      </select>
    </div>
    <div class="form-group">
      <label for="style">Style:</label>
      <select id="style" bind:value={selectedStyle}>
        {#each styles as style}
          <option value={style}>{style.name}</option>
        {/each}
      </select>
    </div>
    <div class="form-group">
      <label for="image-upload">Upload Image:</label>
      <input type="file" id="image-upload" accept="image/jpeg, image/png, image/webp" on:change={handleImageUpload} />
    </div>
    {#if inputImage}
      <div class="form-group">
        <label for="strength">Strength:</label>
        <input type="range" id="strength" min="0" max="1" step="0.1" bind:value={strength} />
        <span>{strength.toFixed(1)}</span>
      </div>
    {/if}
    <button on:click={generateImage} disabled={isGenerating || !apiKey}>
      Generate Image
    </button>
    {#if errorMessage}
      <div class="error-message">{errorMessage}</div>
    {/if}
    <div class="image-list">
      {#each generatedImages as image}
        <div class="image-item">
          <div class="image-wrapper" on:click={() => selectImage(image.url)}>
            <img src={image.thumbnail} alt={image.name} />
            <div class="download-button" on:click|stopPropagation={() => downloadImage(image.url, image.name)}>
              <img src="/download-icon.svg" alt="Download" />
            </div>
          </div>
        </div>
      {/each}
    </div>
  </div>
  <div class="main-content">
    {#if selectedImage}
      <img src={selectedImage} alt="Generated Image" class="generated-image" />
    {:else}
      <div class="no-image-selected">No image selected</div>
    {/if}
  </div>
</div>

<style>
	.app {
    display: flex;
  }

  .sidebar {
    width: 300px;
    padding: 20px;
  }

  .form-group {
    margin-bottom: 20px;
  }

  label {
    display: block;
    font-size: 14px;
    margin-bottom: 5px;
  }

  input[type='text'],
  textarea {
    width: 100%;
    padding: 8px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }

  button {
    padding: 10px 20px;
    background-color: #007bff;
    color: #fff;
    border: none;
    border-radius: 4px;
	  cursor: pointer;
	}
  
	button:disabled {
    background-color: #ccc;
    cursor: not-allowed;
  }

  .error-message {
    color: red;
    margin-top: 10px;
  }

  .image-list {
    margin-top: 20px;
  }

  .image-item {
    display: flex;
    align-items: center;
    margin-bottom: 10px;
    cursor: pointer;
  }

  .image-item img {
    width: 50px;
    height: 50px;
    margin-right: 10px;
  }

  .main-content {
    flex: 1;
    padding: 20px;
	  display: flex;
	  justify-content: center;
	  align-items: center;
	  cursor: pointer;
	}
  
	.generated-image {
    max-width: 100%;
    max-height: 100%;
  }

  .no-image-selected {
    font-size: 18px;
    color: #888;
	}

  .image-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
    grid-gap: 10px;
    margin-top: 20px;
  }

  .image-item {
    position: relative;
    cursor: pointer;
  }

  .image-wrapper {
    position: relative;
    display: inline-block;
  }

  .image-wrapper img {
    width: 100%;
    height: auto;
    border-radius: 4px;
  }

  .download-button {
    position: absolute;
    top: 5px;
    right: 5px;
    width: 16px;
    height: 16px;
    border-radius: 50%;
    background-color: rgba(255, 255, 255, 0.0);
    display: flex;
    justify-content: center;
    align-items: center;
    cursor: pointer;
  }

  .download-button img {
    left: 16px;
    width: 16px;
    height: 16px;
    justify-content: center;
	  align-items: center;
  }
  .form-group {
    margin-bottom: 20px;
  }

  label {
    display: block;
    font-size: 14px;
    margin-bottom: 5px;
  }

  select {
    width: 100%;
    padding: 8px;
    border: 1px solid #ccc;
    border-radius: 4px;
  }
  input[type='range'] {
    width: 100%;
  }
  </style>