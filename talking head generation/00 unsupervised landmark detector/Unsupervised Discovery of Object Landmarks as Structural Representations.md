# Keyword

learned landmarks, auto-encoder
# Overview

| ?            | !                                                                                                                                |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------- |
| project page | https://www.ytzhang.net/projects/lmdis-rep/                                                                                      |
| github       | https://github.com/YutingZhang/lmdis-rep                                                                                         |
| presentation | [files.ytzhang.net/lmdis-rep/cvpr2018-oral-presentation.mp4](https://files.ytzhang.net/lmdis-rep/cvpr2018-oral-presentation.mp4) |
| paper        | Unsupervised Discovery of Object Landmarks as Structural Representations                                                         |

# Architecture

![[image-20230917171735755.png]]
# Notes

- unsupervised learning of a human-perceptible structural representation of images
- landmark discovery as an intermediate step for image autoencoding
	- many soft contraints to enforce a desirable properties for landmarks
		- equivariance: equivariant to image transoformation, including object and camera motions
			- ==thin plate spline (TPS)==
- encoder
	- landmark
		- channel-wise softmax → heatmaps
		- weighted mean inside a single channel
	- local latend descriptor (because landmarks are insufficient to represent all visual content)
		- not too much information to overwhelm the image structures reflected by landmarks
		- low dimensional  local descriptor for each landmark
- controllable image decoding
	- by varying the input landmark (and fixing the latent descriptor), the decoder part can be use to do image manipulation and image generation
