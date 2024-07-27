## Project1: To build a model that can generate a descriptive caption for an image Without Attention :
- Some Result of Fiest Proect
---

!<img src="image.png" alt="Image" width="400" height="400" /> !<img src="image-1.png" alt="Image" width="400" height="400" />

---
- More results in Notebook

## Project2: To build a model that can generate a descriptive caption for an image:

- In the interest of keeping things simple, let's implement the [_Show, Attend, and Tell_](https://arxiv.org/abs/1502.03044) paper. This is by no means the current state-of-the-art, but is still pretty darn amazing.
- This model learns _where_ to look. As you generate a caption, word by word, you can see the model's gaze shifting across the image. This is possible because of its _Attention_ mechanism, which allows it to focus on the part of the image most relevant to the word it is going to utter next.
- Here are some captions generated on _test_ images not seen during training or validation:
![alt text](<Screenshot 2024-07-27 125501.png>)
---

![](./img/plane.png)

---

![](./img/boats.png)

---

![](./img/bikefence.png)

---

![](./img/sheep.png)

---

There are more examples at the [end of the tutorial](https://github.com/sgrvinod/a-PyTorch-Tutorial-to-Image-Captioning#some-more-examples).

---

## Concepts Explaination behind this project

* **Image captioning**. duh.

* **Encoder-Decoder architecture**. Typically, a model that generates sequences will use an Encoder to encode the input into a fixed form and a Decoder to decode it, word by word, into a sequence.

* **Attention**. The use of Attention networks is widespread in deep learning, and with good reason. This is a way for a model to choose only those parts of the encoding that it thinks is relevant to the task at hand. The same mechanism you see employed here can be used in any model where the Encoder's output has multiple points in space or time. In image captioning, you consider some pixels more important than others. In sequence to sequence tasks like machine translation, you consider some words more important than others.

* **Transfer Learning**. This is when you borrow from an existing model by using parts of it in a new model. This is almost always better than training a new model from scratch (i.e., knowing nothing). As you will see, you can always fine-tune this second-hand knowledge to the specific task at hand. Using pretrained word embeddings is a dumb but valid example. For our image captioning problem, we will use a pretrained Encoder, and then fine-tune it as needed.

* **Beam Search**. This is where you don't let your Decoder be lazy and simply choose the words with the _best_ score at each decode-step. Beam Search is useful for any language modeling problem because it finds the most optimal sequence.


### Encoder:

- The Encoder **encodes the input image with 3 color channels into a smaller image with "learned" channels**. This smaller encoded image is a summary representation of all that's useful in the original image. Since we want to encode images, we use Convolutional Neural Networks (CNNs). We don't need to train an encoder from scratch. Why? Because there are already CNNs trained to represent images.
- For years, people have been building models that are extraordinarily good at classifying an image into one of a thousand categories. It stands to reason that these models capture the essence of an image very well.
- I have chosen to use the **101 layered Residual Network trained on the ImageNet classification task**, already available in PyTorch. As stated earlier, this is an example of Transfer Learning. You have the option of fine-tuning it to improve performance.
![ResNet Encoder](./img/encoder.png)

- These models progressively create smaller and smaller representations of the original image, and each subsequent representation is more "learned", with a greater number of channels. The final encoding produced by our ResNet-101 encoder has a size of 14x14 with 2048 channels, i.e., a `2048, 14, 14` size tensor.

- I encourage you to experiment with other pre-trained architectures. The paper uses a VGGnet, also pretrained on ImageNet, but without fine-tuning. Either way, modifications are necessary. Since the last layer or two of these models are linear layers coupled with softmax activation for classification, we strip them away.

### Decoder (Main diffrence between both projects)

- The Decoder's job is to **look at the encoded image and generate a caption word by word**.Since it's generating a sequence, it would need to be a **Recurrent Neural Network (RNN). We will use an LSTM**.

- **In a typical setting without Attention**, you could simply average the encoded image across all pixels. You could then feed this, with or without a linear transformation, into the Decoder as its first hidden state and generate the caption. Each predicted word is used to generate the next word.

![Decoder without Attention](./img/decoder_no_att.png)

- **In a setting _with_ Attention**, we want the Decoder to be able to **look at different parts of the image at different points in the sequence**. For example, while generating the word `football` in `a man holds a football`, the Decoder would know to focus on – you guessed it – the football!

![Decoding with Attention](./img/decoder_att.png)

- **Instead of the simple average, we use the _weighted_ average across all pixels**, with the weights of the important pixels being greater. This weighted representation of the image can be concatenated with the previously generated word at each step to generate the next word.

### Attention:
- The Attention network **computes these weights**.Intuitively, how would you estimate the importance of a certain part of an image? You would need to be aware of the sequence you have generated _so far_, so you can look at the image and decide what needs describing next. For example, after you mention `a man`, it is logical to declare that he is `holding a football`.This is exactly what the Attention mechanism does – it considers the sequence generated thus far, and _attends_ to the part of the image that needs describing next.

![Attention](./img/att.png)

- We will use _soft_ Attention, where the weights of the pixels add up to 1. If there are `P` pixels in our encoded image, then at each timestep `t` –

<p align="center">
<img src="./img/weights.png">
</p>

- You could interpret this entire process as computing the **probability that a pixel is _the_ place to look to generate the next word**.

### Putting it all together:

- It might be clear by now what our combined network looks like.

![Putting it all together](./img/model.png)

- Once the Encoder generates the encoded image, we transform the encoding to create the initial hidden state `h` (and cell state `C`) for the LSTM Decoder.
- At each decode step,
  - the encoded image and the previous hidden state is used to generate weights for each pixel in the Attention network.
  - the previously generated word and the weighted average of the encoding are fed to the LSTM Decoder to generate the next word.

### Beam Search:

- We use a linear layer to transform the Decoder's output into a score for each word in the vocabulary. The straightforward – and greedy – option would be to choose the word with the highest score and use it to predict the next word. But this is not optimal because the rest of the sequence hinges on that first word you choose. If that choice isn't the best, everything that follows is sub-optimal. And it's not just the first word – each word in the sequence has consequences for the ones that succeed it.
- It might very well happen that if you'd chosen the _third_ best word at that first step, and the _second_ best word at the second step, and so on... _that_ would be the best sequence you could generate. It would be best if we could somehow choose the sequence that has the highest _overall_ score from a basket of candidate sequences.

- Beam Search does exactly this.
 - At the first decode step, consider the top `k` candidates.
 - Generate `k` second words for each of these `k` first words.
 - Choose the top `k` [first word, second word] combinations considering additive scores.
 - For each of these `k` second words, choose `k` third words, choose the top `k` [first word, second word, third word] combinations.
 - Repeat at each decode step.
 - After `k` sequences terminate, choose the sequence with the best overall score.

![Beam Search example](./img/beam_search.png)

- As you can see, some sequences (striked out) may fail early, as they don't make it to the top `k` at the next step. Once `k` sequences (underlined) generate the `<end>` token, we choose the one with the highest score.

### Implementation:
#### Dataset:
- I'm using the Flicker8k Dataset and download this images dataset from [kaggle](https://www.kaggle.com/datasets/adityajn105/flickr8k) . We will use [Andrej Karpathy's training, validation, and test splits](http://cs.stanford.edu/people/karpathy/deepimagesent/caption_datasets.zip). This zip file contain the captions of Flicker8k. You will also find splits and captions for the MSCOCO and Flicker30k datasets.
- So if any want to use other data like MSCOCO and Flicker30k the feel free to use this but need to download images for these dataset.

#### Inputs to model:

- Images: Since we're using a pretrained Encoder, we would need to process the images into the form this pretrained Encoder is accustomed to. Pretrained ImageNet models available as part of PyTorch's `torchvision` module. [This page](https://pytorch.org/docs/master/torchvision/models.html) details the preprocessing or transformation we need to perform – pixel values must be in the range [0,1] and we must then normalize the image by the mean and standard deviation of the ImageNet images' RGB channels. Also, PyTorch follows the NCHW convention, which means the channels dimension (C) must precede the size dimensions. We will resize all Flicker8k images to 256x256 for uniformity. Therefore, **images fed to the model must be a `Float` tensor of dimension `N, 3, 256, 256`**, and must be normalized by the aforesaid mean and standard deviation. `N` is the batch size.

- Captions: Captions are both the target and the inputs of the Decoder as each word is used to generate the next word. To generate the first word, however, we need a *zeroth* word, `<start>`. At the last word, we should predict `<end>` the Decoder must learn to predict the end of a caption. This is necessary because we need to know when to stop decoding during inference.`<start> a man holds a football <end>`Since we pass the captions around as fixed size Tensors, we need to pad captions (which are naturally of varying length) to the same length with `<pad>` tokens.`<start> a man holds a football <end> <pad> <pad> <pad>....`Furthermore, we create a `word_map` which is an index mapping for each word in the corpus, including the `<start>`,`<end>`, and `<pad>` tokens. PyTorch, like other libraries, needs words encoded as indices to look up embeddings for them or to identify their place in the predicted word scores.

- Caption Lengths: Since the captions are padded, we would need to keep track of the lengths of each caption. This is the actual length + 2 (for the `<start>` and `<end>` tokens). Caption lengths are also important because you can build dynamic graphs with PyTorch. We only process a sequence upto its length and don't waste compute on the `<pad>`s. Therefore, **caption lengths fed to the model must be an `Int` tensor of dimension `N`**.


#### Encoder:

See `Encoder` in `models.py`, We use a pretrained ResNet-101 already available in PyTorch's `torchvision` module. Discard the last two layers (pooling and linear layers), since we only need to encode the image, and not classify it. We do add an `AdaptiveAvgPool2d()` layer to **resize the encoding to a fixed size**. This makes it possible to feed images of variable size to the Encoder. (We did, however, resize our input images to `256, 256` because we had to store them together as a single tensor.) Since we may want to fine-tune the Encoder, we add a `fine_tune()` method which enables or disables the calculation of gradients for the Encoder's parameters. We **only fine-tune convolutional blocks 2 through 4 in the ResNet**, because the first convolutional block would have usually learned something very fundamental to image processing, such as detecting lines, edges, curves, etc. We don't mess with the foundations.

#### Attention:

See `Attention` in `models.py`, the Attention network is simple, it's composed of only linear layers and a couple of activations.Separate linear layers **transform both the encoded image (flattened to `N, 14 * 14, 2048`) and the hidden state (output) from the Decoder to the same dimension**, viz. the Attention size. They are then added and ReLU activated. A third linear layer **transforms this result to a dimension of 1**, whereupon we **apply the softmax to generate the weights** `alpha`.

#### Decoder:

- See `DecoderWithAttention` in `models.py`, The output of the Encoder is received here and flattened to dimensions `N, 14 * 14, 2048`. This is just convenient and prevents having to reshape the tensor multiple times. We **initialize the hidden and cell state of the LSTM** using the encoded image with the `init_hidden_state()` method, which uses two separate linear layers. At the very outset, we **sort the `N` images and captions by decreasing caption lengths**. This is so that we can process only _valid_ timesteps, i.e., not process the `<pad>`s.
![](./img/sorted.jpg)

- We can iterate over each timestep, processing only the colored regions, which are the **_effective_ batch size** `N_t` at that timestep. The sorting allows the top `N_t` at any timestep to align with the outputs from the previous step. At the third timestep, for example, we process only the top 5 images, using the top 5 outputs from the previous step.

- This **iteration is performed _manually_ in a `for` loop** with a PyTorch [`LSTMCell`](https://pytorch.org/docs/master/nn.html#torch.nn.LSTM) instead of iterating automatically without a loop with a PyTorch [`LSTM`](https://pytorch.org/docs/master/nn.html#torch.nn.LSTM). This is because we need to execute the Attention mechanism between each decode step. An `LSTMCell` is a single timestep operation, whereas an `LSTM` would iterate over multiple timesteps continously and provide all outputs at once. We **compute the weights and attention-weighted encoding** at each timestep with the Attention network. In section `4.2.1` of the paper, they recommend **passing the attention-weighted encoding through a filter or gate**. This gate is a sigmoid activated linear transform of the Decoder's previous hidden state. The authors state that this helps the Attention network put more emphasis on the objects in the image. We **concatenate this filtered attention-weighted encoding with the embedding of the previous word** (`<start>` to begin), and run the `LSTMCell` to **generate the new hidden state (or output)**. A linear layer **transforms this new hidden state into scores for each word in the vocabulary**, which is stored.We also store the weights returned by the Attention network at each timestep. 

### Training:
- Before you begin, make sure to save the required data files for training, validation, and testing.See `train.py`. To **train your model from scratch**, simply run this file – `python train.py`. To **resume training at a checkpoint**, point to the corresponding file with the `checkpoint` parameter at the beginning of the code. Note that we perform validation at the end of every training epoch.

- Since we're generating a sequence of words, we use `CrossEntropyLoss`. You only need to submit the raw scores from the final layer in the Decoder, and the loss function will perform the softmax and log operations.

- **Early stopping with BLEU:** To evaluate the model's performance on the validation set, we will use the automated [BiLingual Evaluation Understudy (BLEU)](http://www.aclweb.org/anthology/P02-1040.pdf) evaluation metric. This evaluates a generated caption against reference caption(s). For each generated caption, we will use all `N_c` captions available for that image as the reference captions.

- The authors of the _Show, Attend and Tell_ paper observe that correlation between the loss and the BLEU score breaks down after a point, so they recommend to stop training early on when the BLEU score begins to degrade, even if the loss continues to decrease. I used the BLEU tool [available in the NLTK module](https://www.nltk.org/_modules/nltk/translate/bleu_score.html). Note that there is considerable criticism of the BLEU score because it doesn't always correlate well with human judgment. The authors also report the METEOR scores for this reason, but I haven't implemented this metric.

- the BLEU score measured above on the resulting captions _does not_ reflect real performance. In fact, the BLEU score is a metric designed for comparing naturally generated captions to ground-truth captions of differing length. Once batched inference is implemented, i.e. no Teacher Forcing(means ground truth caption gives when last token not generated this proces make training fast), early-stopping with the BLEU score will be truly 'proper'.

- With this in mind, I used [`eval.py`](https://github.com/sgrvinod/a-PyTorch-Tutorial-to-Image-Captioning/blob/master/eval.py) to compute the correct BLEU-4 scores of this model checkpoint on the validation and test sets _without_ Teacher Forcing, at different beam sizes –

Beam Size | Validation BLEU-4 | Test BLEU-4 |
:---: | :---: | :---: |
1 | 29.98 | 30.28 |
3 | 32.95 | 33.06 |
5 | 33.17 | 33.29 |

- The BLUE test score is higher than the result in the paper, and could be because of how our BLEU calculators are parameterized, the fact that I used a ResNet encoder, and actually fine-tuned the encoder – even if just a little. Also, remember – when fine-tuning during Transfer Learning, it's always better to use a learning rate considerably smaller than what was originally used to train the borrowed model. This is because the model is already quite optimized, and we don't want to change anything too quickly. I used `Adam()` for the Encoder as well, but with a learning rate of `1e-4`, which is a tenth of the default value for this optimizer.

- See `caption.py`.During inference, we _cannot_ directly use the `forward()` method in the Decoder because it uses Teacher Forcing. Rather, we would actually need to **feed the previously generated word to the LSTM at each timestep**. `caption_image_beam_search()` reads an image, encodes it, and applies the layers in the Decoder in the correct order, while using the previously generated word as the input to the LSTM at each timestep. It also incorporates Beam Search. `visualize_att()` can be used to visualize the generated caption along with the weights at each timestep as seen in the examples.

- To **caption an image** from the command line, point to the image, model checkpoint, word map (and optionally, the beam size) as follows –

`python caption.py --img='path/to/image.jpeg' --model='path/to/BEST_checkpoint_coco_5_cap_per_img_5_min_word_freq.pth.tar' --word_map='path/to/WORDMAP_coco_5_cap_per_img_5_min_word_freq.json' --beam_size=5`

### Some more examples

---

![](./img/birds.png)

---

![](./img/salad.png)

---

![](./img/manbike.png)

---

![](./img/catbanana.png)

---

![](./img/firehydrant.png)

---

![](./img/tommy.png)

---
