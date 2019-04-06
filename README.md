# MMD-GAN with Repulsive Loss Function
GAN: generative adversarial nets; MMD: maximum mean discrepancy; TF: TensorFlow

This repository contains codes for MMD-GAN and the repulsive loss proposed in ICLR paper [1]: \
Wei Wang, Yuan Sun, Saman Halgamuge. Improving MMD-GAN Training with Repulsive Loss Function. ICLR 2019. URL: https://openreview.net/forum?id=HygjqjR9Km.

## About the code
The code defines the neural network architecture as dictionaries and strings to ease test of different models. It also contains many other models I have tried, so sorry if you find it a little bit confusing.

The structure of code:
1. _DeepLearning/my_sngan/SNGan_ defines how a general GAN model is trained and evaluated. 
2. _GeneralTools_ contains various tools:
    1. _graph_func_ contains functions to run a model graph and metrics for evaluating generative models (Line 1595).
    2. _input_func_ contains functions to handle datasets and input pipeline.
    3. _layer_func_ contains functions to convert network architecture dictionary to operations
    4. _math_func_ defines various mathematical operations. You may find spectral normalization at Line 397, loss functions for GAN at Line 2088, repulsive loss at Line 2505, repulsive with bounded kernel (referred to as rmb) at Line 2530.
    5. _misc_fun_ contains FLAGs for the code.
3. *my_test_* contain the specific model architectures and hyperparameters. 

### Running the tests
1. Modify _GeneralTools/misc_func_ accordingly; 
2. Read _Data/ReadMe.md_; download and prepare the datasets;
3. Run *my_test_* with proper hyperparameters.

## About the algorithms
Here we introduce the algorithms and tricks. 

### Proposed Methods
The paper [1] proposed three methods:
1. Repulsive loss

![equation](https://latex.codecogs.com/gif.latex?\inline&space;L_G=\sum_{i\ne&space;j}k_D(x_i,x_j)-2\sum_{i\ne&space;j}k_D(x_i,y_j)&plus;\sum_{i\ne&space;j}k_D(y_i,y_j))

![equation](https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}=\sum_{i\ne&space;j}k_D(x_i,x_j)-\sum_{i\ne&space;j}k_D(y_i,y_j))

where ![equation](https://latex.codecogs.com/gif.latex?\inline&space;x_i,x_j) - real samples, ![equation](https://latex.codecogs.com/gif.latex?\inline&space;y_i,y_j) - generated samples, ![equation](https://latex.codecogs.com/gif.latex?\inline&space;k_D) - kernel formed by the discriminator ![equation](https://latex.codecogs.com/gif.latex?\inline&space;D) and kernel ![equation](https://latex.codecogs.com/gif.latex?\inline&space;k). The discriminator loss of previous MMD-GAN [2], or what we called attractive loss, is ![equation](https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{att}}=-L_G). 

Below is an illustration of the effects of MMD losses on free R(eal) and G(enerated) particles (code in _Figures_ folder). The particles stand for discriminator outputs of samples, but, for illustration purpose, we allow them to move freely. These GIFs extend the Figure 1 of paper [1].

| | |
| :---: | :---: |
|<img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{att}}" title="L_D^{\text{att}}"/> | <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}" title="L_D^{\text{rep}}"/> |
|<img src="Figures/0_mmd_d_att.gif" alt="mmd_d_att">  |  <img src="Figures/0_mmd_d_rep.gif" alt="mmd_d_rep"> |
| <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G" title="L_G"/> paired with <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{att}}" title="L_D^{\text{att}}"/> | <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G" title="L_G"/> paired with <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}" title="L_D^{\text{rep}}"/> |
| <img src="Figures/0_mmd_g_att.gif" alt="mmd_g_att">  |  <img src="Figures/0_mmd_g_rep.gif" alt="mmd_g_rep"> |

In the first row, we randomly initialized the particles, and applied <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{att}}" title="L_D^{\text{att}}"/> or <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}" title="L_D^{\text{rep}}"/> for 600 steps. The velocity of each particle is <img src="https://latex.codecogs.com/gif.latex?\inline&space;-0.01\nabla{L_D}" title="vD"/>. In the second row, we obtained the particle positions at the 450th step of the first row and applied <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G" title="L_G"/> for another 600 steps with velocity <img src="https://latex.codecogs.com/gif.latex?\inline&space;-0.01\nabla{L_G}" title="vG"/>. The blue and orange arrows stand for the gradients of attractive and repulsive components of MMD losses respectively. In summary, these GIFs indicate how MMD losses may move the free particles. Of course, the actual case of MMD-GAN is much more complex as we update the model parameters instead of output scores directly and both networks are updated at each step. 

We argue that <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{att}}" title="L_D^{\text{att}}"/> may cause opposite gradients from attractive and repulsive components of both <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D" title="L_D"/> and <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G" title="L_G"/> during training, and thus slow down the training process. Note this is different from the end-stage training when the gradients should be opposite and cancelled out to reach 0. Another way of interpretation is that, by minimizing <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{att}}" title="L_D^{\text{att}}"/>, the discriminator maximizes the similarity between the outputs of real samples, which results in D focusing on the similarities among real images and possibly ignoring the fine details that separate them. The repulsive loss <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}" title="L_D^{\text{rep}}"/> actively learns such fine details to make real sample outputs repel each other. 

2. Bounded kernel (used only in ![equation](https://latex.codecogs.com/gif.latex?L_D))

![equation](https://latex.codecogs.com/gif.latex?\inline&space;k_D^{b}(x_i,x_j)&space;=\exp(-\frac{1}{2\sigma^2}\min(\left&space;\|&space;D(x_i)-D(x_j)&space;\right&space;\|^2,&space;b_u)))

![equation](https://latex.codecogs.com/gif.latex?\inline&space;k_D^{b}(y_i,y_j)&space;=\exp(-\frac{1}{2\sigma^2}\max(\left&space;\|&space;D(y_i)-D(y_j)&space;\right&space;\|^2,&space;b_l)))

The gradient of Gaussian kernel is near 0 when the input distance is too small or large. The bounded kernel avoids kernel saturation by truncating the two tails of distance distribution, an idea inspired by the hinge loss. This prevents the discriminator from becoming too confident.

3. Power iteration for convolution (used in spectral normalization)

At last, we proposed a method to calculate the spectral norm of convolution kernel. At iteration t, for convolution kernel ![equation](https://latex.codecogs.com/gif.latex?\inline&space;W_c), do ![equation](https://latex.codecogs.com/gif.latex?\inline&space;u=\text{conv}(W_c,v^t)), ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\hat{v}=\text{transpose-conv}(W_c,u)), and ![equation](https://latex.codecogs.com/gif.latex?\inline&space;v^{t+1}=\hat{v}/\left&space;\|&space;\hat{v}&space;\right&space;\|). The spectral norm is estimated as ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\sigma_W=\left&space;\|&space;u&space;\right&space;\|).

### Practical Tricks and Issues
We recommend using the following tricks.
1. Spectral normalization, initially proposed in [3]. The idea is, at each layer, to use ![equation](https://latex.codecogs.com/gif.latex?\inline&space;\hat{W}_c=W_c\cdot&space;\frac{C}{\sigma_W}) for convolution/dense multiplication. Here we multiply the signal with a constant <img src="https://latex.codecogs.com/gif.latex?\inline&space;C>1" title="C>1"/> after each spectral normalization to compensate for the decrease of signal norm at each layer. In the main text of paper [1], we used <img src="https://latex.codecogs.com/gif.latex?\inline&space;C=1/0.55" title="C=1/0.55"/> empirically. In Appendix C.3 of paper [1], we tested a variety of <img src="https://latex.codecogs.com/gif.latex?\inline&space;C" title="C"/> values.
2. Two time-scale update rule (TTUR) [4]. The idea is to use different learning rates for the generator and discriminator.

Unlike the case of Wasserstein GAN, we do not encourage using the repulsive loss for discriminator <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}" title="L_D^{\text{rep}}"/> or the MMD loss for generator <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G}" title="LG"/> to indicate the progress of training. You may find that, during the training process,
- both <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}" title="L_D^{\text{rep}}"/> and <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G}" title="LG"/> may be close to 0 initially; this is because both G and D are weak.
- <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G}" title="L_G"/> may gradually increase during training; this is because it becomes harder for G to generate high quality samples and fool D (and G may not have the capacity to do so).

For balanced and capable G and D, we would expect both <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_D^{\text{rep}}" title="L_D^{\text{rep}}"/> and <img src="https://latex.codecogs.com/gif.latex?\inline&space;L_G}" title="L_G"/> to stay close to 0 during the whole training process and any kernel (i.e., <img src="https://latex.codecogs.com/gif.latex?\inline&space;k_D(x,x)}" title="k_D(x,x)"/>, <img src="https://latex.codecogs.com/gif.latex?\inline&space;k_D(y,y)}" title="k_D(y,y)"/> and <img src="https://latex.codecogs.com/gif.latex?\inline&space;k_D(x,y)}" title="k_D(x,y)"/>) to be close to 0.6 and away from 0 or 1.

In some cases, you may find training using the repulsive loss diverges. Do not panic. It may be that the learning rate is not suitable. Please try other learning rate or the bounded kernel. 

### Final Comments
Thank you for reading!

Please feel free to leave comments if things do not work or suddenly work, or if exploring my code ruins your day. :)

## Reference
[1] Wei Wang, Yuan Sun, Saman Halgamuge. Improving MMD-GAN Training with Repulsive Loss Function. ICLR 2019. URL: https://openreview.net/forum?id=HygjqjR9Km. \
[2] Chun-Liang Li, Wei-Cheng Chang, Yu Cheng, Yiming Yang, and Barnabas Poczos. MMD GAN: Towards deeper understanding of moment matching network. In NeurIPS, 2017.
[3] Takeru Miyato, Toshiki Kataoka, Masanori Koyama, and Yuichi Yoshida. Spectral normalization
for generative adversarial networks. In ICLR, 2018. \
[4] Martin Heusel, Hubert Ramsauer, Thomas Unterthiner, Bernhard Nessler, and Sepp Hochreiter.  GANs Trained by a Two Time-Scale Update Rule Converge to a Nash Equilibrium. In NeurIPS, 2017.
