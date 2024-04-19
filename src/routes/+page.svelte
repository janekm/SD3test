<!-- src/routes/index.svelte -->
<script>
  import { onMount } from 'svelte';

  let apiKey = '';
  let prompt = 'dog wearing black glasses';
  let generatedImages = [];
  let selectedImage = '';
  let isGenerating = false;
  let errorMessage = '';

  onMount(() => {
    apiKey = localStorage.getItem('APIKey') || '';
  });

  function saveApiKey() {
    localStorage.setItem('APIKey', apiKey);
  }

  async function generateImage() {
    isGenerating = true;
    errorMessage = null;

    const url = 'https://api.stability.ai/v2beta/stable-image/generate/sd3';
    const formData = new FormData();
    formData.append('prompt', prompt);
    formData.append('output_format', 'jpeg');

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
        throw new Error(`Error ${response.status}`);
      }
    } catch (error) {
      errorMessage = error.message;
    } finally {
      isGenerating = false;
    }
  }

  function selectImage(imageUrl) {
    selectedImage = imageUrl;
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
    <button on:click={generateImage} disabled={isGenerating || !apiKey}>
      Generate Image
    </button>
    {#if errorMessage}
      <div class="error-message">{errorMessage}</div>
    {/if}
    <div class="image-list">
      {#each generatedImages as image}
        <div class="image-item" on:click={() => selectImage(image.url)}>
          <img src={image.thumbnail} alt={image.name} />
          <span>{image.name}</span>
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
  }

  .generated-image {
    max-width: 100%;
    max-height: 100%;
  }

  .no-image-selected {
    font-size: 18px;
    color: #888;
  }
</style>