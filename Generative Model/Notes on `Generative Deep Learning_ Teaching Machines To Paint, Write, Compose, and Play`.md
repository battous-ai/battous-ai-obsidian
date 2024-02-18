# Chapter 3 Variational Autoencoders
- autoencoder
	- autoencoder can be used as generative model if we choose randomly from the latent space and run it through decoder 
	- autoencoders can be used to clean noisy images
	- problems
		- different groups cover uneven large part of the latent space
		- the latent space is not countinuous, no idea how to choose a random point
		- increasing the dimensionality of the lantent space leads to problems if we want to use autoencoder as a generative model
			- huge gaps between groups of similar points
## Variational Autoencoders
generate的时候，默认每一个dim在lantent space都是normal distribution