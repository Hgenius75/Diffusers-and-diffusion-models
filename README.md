<h1 align="center">Introduction to the Diffusers library and diffusion models</h1>

Diffusers is a library by Hugging Face that enables working with hundreds of pre-trained Stable Diffusion models for generating images and audio.

Tasks that can be addressed using Stable Diffusion (from the Diffusers library):

## Text-To-Image (txt2img)

Most common usage: you describe in words what you want to generate, and the neural network provides you with an image.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/e8d/706/441/e8d7064413c3d8d5ed8c642f998eb47a.png">
</p>

## Image-To-Image (img2img)


Generating new images based on existing ones. You provide an input image as a starting point (template) and describe in text what you want to do with it. The diffusion model will generate a new image with the same color and composition as the input image but with the described modifications.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/79b/b6f/110/79bb6f110d81ee4c0c7e6f11fadc79f1.png">
</p>

## Inpaint


Input consists of an image, a mask, and text. The neural network replaces the image inside the mask with what is described in the request.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/85a/11e/d43/85a11ed438ea8a6615971268b0ade897.png">
</p>

## Depth-to-image (depth2img)


This model is also used for generating new images from existing ones. However, it takes into account the shape and depth of objects in the original image, specified using a depth map.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/126/57e/f33/12657ef3354f7be2b9b31e7125cce771.png">
</p>

> In addition to Stable Diffusion, there are two other models comparable in quality: Dall-E 2 and MidJerney. However, they are closed-source, their weights are not publicly available, and access to them can only be obtained for a fee.

## Theory
Before diving into practice, let's briefly explore the theory...


### What is Diffusion
Stable Diffusion belongs to the class of diffusion models. The idea behind them is to blend an image with random noise and then train a neural network to recover the originals from the noisy images.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/9a1/f47/49e/9a1f4749e51b6f19f9441f33a36e62a7.png">
</p>

The diffusion process consists of two parts: forward and inverse. The forward process corrupts the image, while the inverse process restores it.

The forward process consists of multiple steps (e.g., 100). It starts with the original image. At each step, the image from the previous step is taken, and a bit of noise (1%) is added to it. This continues until the last step, where the original image turns into random noise.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/8d2/1b4/9f9/8d21b49f978085075b5b3825898d0265.png">
</p>

Now, for each of the 100 steps, we have a pair of images: the image with noise and the noise itself. We will train the model to predict this noise by inputting the image with noise.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/377/d52/85b/377d5285b3a6dd64b2f5fe8c954ec75d.png">
</p>

> It is possible to control how much noise is added to the image at each step, allowing for the distribution of the process over 10 or 100 steps.

The inverse process is also iterative: at each step, we ask the model to predict the noise and subtract it from the image. This continues until we go through all the steps and remove all the noise from the image.

> Since the image is reconstructed from random noise each time, a new image will be generated each time.

## Model Contents
Stable Diffusion is not a single model but rather a whole system consisting of several components and models. During the inference stage, i.e., when the model is already trained, the following workflow is observed:
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/542/df4/b28/542df4b28a7900f1604667f5a85cfd6c.png">
</p>
### CLIP
CLIP (the textual encoder of the model) is a specialized language model of the Transformer type. CLIP takes text as input and outputs a vector of numbers describing each word in the text.

The CLIP model was trained on images and their descriptions. Its main feature is that during training, it generates two vectors: one for the image and one for the text. The model learns to minimize the cosine distance between these vectors. Thus, the embeddings it produces simultaneously describe both the text and the image.

### UNet + Scheduler
UNet is the workhorse under the hood of Stable Diffusion. UNet specifically predicts noise. The process is controlled by Scheduler, an algorithm that does not contain trainable parameters. It is responsible for how we add noise to the image. Together, they progressively clean the image from noise.

Generating a random image from noise is not enough. We also need the model to pay attention to the textual description we provide. To achieve this, UNet uses CrossAttention blocks. These blocks allow the model to look into CLIP during the cleaning process, ensuring that the generated result aligns with the textual description.

### Decoder
During the operation of UNet and Scheduler, it is not the image itself that is cleaned from noise but the so-called latent space. This is a kind of compressed representation of the image. Before starting the work, noise is generated in this latent space, and it is directly cleaned. Once the latent space is cleared of noise, it is expanded back to a normal image using the decoder.

The decoder is part of another model, a Variational Autoencoder (VAE). An autoencoder is a neural network with two sub-networks. The encoder compresses the image, and the decoder learns to decompress it to its original state with maximum accuracy. In the trained Stable Diffusion model, only the decoder is present. Both parts are used during training.
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/013/33a/950/01333a9501344c967b42e090eac59e76.png">
</p>

> A color image with a resolution of 512x512 consists of 262,144 pixels in each of the 3 channels. Applying the process directly to it would require a lot of time and memory. However, we want to generate images quickly and on relatively simple graphics cards. Therefore, the autoencoder used in Stable Diffusion has a reduction coefficient of 8. This means that an image of shape (3, 512, 512) becomes (3, 64, 64) in the latent space, requiring 64 times less memory.

## Installation
To begin, let's install the libraries:
```
pip install --upgrade diffusers accelerate transformers
```

After that, we need to find a model. Where exactly to find them will be discussed below. For now, let's use the [Stable Diffusion XL](https://huggingface.co/stabilityai/stable-diffusion-xl-base-1.0) model from the Hugging Face platform.

> The installed and imported "accelerate" library significantly speeds up model loading into memory.

## Usage
The easiest way to use a pre-trained diffusion model for inference is to use the DiffusionPipeline class. It contains everything needed for the diffusion model (tokenizer, UNet, decoder, etc.).

For experiments, let's learn to convert text into an image.
```
from diffusers import DiffusionPipeline

pipeline = DiffusionPipeline.from_pretrained("stabilityai/stable-diffusion-xl-base-1.0")

promt = "An image of a squirrel in Picasso style"
image = pipeline(promt=promt).images[0]

image.save("image_of_squirrel_painting.png")
```
Here we:

1. Created an instance of DiffusionPipeline and loaded a pre-trained model from the Hugging Face Hub.

2. Formulated a text query and passed it to the pipeline. Received an image as the output from the pipeline.

3. Saved the image.

Things to note here...

## Cache and Local Usage
DiffusionPipeline loads and caches all components included in the pipeline: models, tokenizers, schedulers, etc.

You can find them locally at the following path:
```
C:\Users\<user_name>\.cache\huggingface\hub
```
Alternatively, you can directly download the model you need locally:
```
!git lfs install
!git clone https://huggingface.co/runwayml/stable-diffusion-v1-5
```
And then specify the path to it in the pipeline:
```
pipeline = DiffusionPipeline.from_pretrained("./stable-diffusion-v1-5")
```
## Pipeline Contents
As mentioned before, Stable Diffusion is not a monolithic model but a complex modular structure. You can see what specifically is included in it by simply printing the pipeline.
```
StableDiffusionPipeline {
  "_class_name": "StableDiffusionPipeline",
  "_diffusers_version": "0.19.1",
  "_name_or_path": "/root/work/text_to_image/stable_diffusion_v1_5/snapshots/c9ab35ff5f2c362e9e22fbafe278077e196057f0",
  "feature_extractor": [
    "transformers",
    "CLIPImageProcessor"
  ],
  "requires_safety_checker": true,
  "safety_checker": [
    "stable_diffusion",
    "StableDiffusionSafetyChecker"
  ],
  "scheduler": [
    "diffusers",
    "PNDMScheduler"
  ],
  "text_encoder": [
    "transformers",
    "CLIPTextModel"
  ],
  "tokenizer": [
    "transformers",
    "CLIPTokenizer"
  ],
  "unet": [
    "diffusers",
    "UNet2DConditionModel"
  ],
  "vae": [
    "diffusers",
    "AutoencoderKL"
  ]
}
```
## Reproducibility
Stable Diffusion generates images from random noise. Due to this, you will see a new image with each run. This might be unnecessary in itself, and it can also hinder your ability to improve the image by adjusting parameters (discussed below). Therefore, you can fix the generated image using a seed:
```
import torch

generator = torch.Generator("cuda").manual_seed(0)
image = pipeline(prompt, generator=generator).images[0]
image
```
More information about reproducibility can be found [here](https://huggingface.co/docs/diffusers/using-diffusers/reproducibility).

## Speed and Memory
By default, DiffusionPipeline performs calculations with float32 precision. You can speed up the inference process (and reduce memory consumption) by switching to lower precision - float16.
```
pipeline = DiffusionPipeline.from_pretrained(model_name, torch_dtype=torch.float16)
```
## Main Parameters
Let's consider some pipeline parameters...

### num_inference_steps
One of the most crucial parameters. The number of denoising steps. Increasing num_inference_steps usually leads to a higher quality image but comes at the cost of longer computation. Default value = 50.
```
image = pipeline(prompt, num_inference_steps=10).images[0]
```

> The relationship between image quality and the number of steps heavily depends on the scheduler used. Some modern schedulers can produce high-quality images in a relatively small number of steps.

<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/90d/678/c23/90d678c23f9022bcd0bf4c1d84202a8b.png">
</p>

### height and width
Set the dimensions of the output image (in pixels). However, keep in mind that the model was trained on images of a specific size and is optimized to produce results of the same size. Changing these dimensions will likely degrade the quality of the generated image.
```
image = pipeline(prompt, height=256, width=256).images[0]
```
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/ea1/4da/9b2/ea14da9b205bded451f5b0cacc9d785d.png">
</p>

### guidance_scale
Determines how accurately the generated image corresponds to the input query. A higher value encourages the model to generate images closely tied to the textual input, at the expense of lower image quality. The default value is 7.5.
```
image = pipeline(prompt, guidance_scale=10).images[0]
```
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/331/39a/e54/33139ae546b948cecd4eb676ebb9e923.png">
</p>

### Mass Generation
Generating images one by one can be quite time-consuming. Therefore, mass generation can be employed.

For example, you can generate images for different queries at once:
```
# Functiont output several imgs
def image_grid(imgs, rows=2, cols=2):
    w, h = imgs[0].size
    grid = Image.new("RGB", size=(cols * w, rows * h))

    for i, img in enumerate(imgs):
        grid.paste(img, box=(i % cols * w, i // cols * h))

    return grid 

# Requests 
prompts = [
    'red car at night on a racing track',
    'yellow car at night on a racing track',
    'blue car at night on a racing track'
] 

# Form pipeline
pipeline = DiffusionPipeline.from_pretrained("stabilityai/stable-diffusion-xl-base-1.0")
images = pipeline(prompts).images

# Save pics
#for i in range(len(images)):
#    images[i].save(f'{i}.png')

image_grid(images, rows=1, cols=3)
```
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/1c6/efb/c66/1c6efbc66b75e205df7a51dc0a420650.png">
</p>
Or you can generate the same query with different seeds:

```
prompts = 3 * ['red car at night on a racing track'] 

pipeline = DiffusionPipeline.from_pretrained("stabilityai/stable-diffusion-xl-base-1.0")
images = pipeline(prompts).images

image_grid(images, rows=1, cols=3)
```
<p align="center">
  <img src="https://habrastorage.org/r/w1560/getpro/habr/upload_files/8b0/092/7d7/8b00927d73ea6ac9027258f5dab0b552.png">
</p>
The number of images that can fit into memory at once depends on the model used and your GPU. Try increasing the quantity until you encounter an OutOfMemoryError.
