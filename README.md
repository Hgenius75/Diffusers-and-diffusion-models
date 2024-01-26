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
