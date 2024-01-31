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
