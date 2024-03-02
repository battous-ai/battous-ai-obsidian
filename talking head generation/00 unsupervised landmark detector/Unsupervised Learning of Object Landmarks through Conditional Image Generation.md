# Keyword

learned landmarks, GAN
# Overview

| ?            | !                                                                              |
| ------------ | ------------------------------------------------------------------------------ |
| project page | https://www.robots.ox.ac.uk/~vgg/research/unsupervised_landmarks/              |
| github       | https://github.com/tomasjakab/imm                                              |
| presentation | https://www.robots.ox.ac.uk/~tomj/data/unsupervised_landmarks-compressed.pdf https://www.youtube.com/watch?v=PMpeMsM_5ak&ab_channel=UCFCRCV
| paper        | Unsupervised Learning of Object Landmarks through Conditional Image Generation |

# Architecture

![[image-20230917230915327.png]]

# Notes

- learn landmark detector → generate image conditioned on both appearance and geometry
	- appearance from the 1st image
	- geometry from the 2nd image (as a set of landmarks)
- two images differ by a viewpoint change and/or an object deformation
- heatmap → a probability distribution (via softmax) → landmark point (expected value) → Gaussian-like function  
	![[image-20230918084808415.png]]
- perceptual loss
	- compare a set of the activations extracted from multiple layers of a deep network for both the reference and the generated images

