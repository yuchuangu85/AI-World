---
title: "Attention Is All You Need"
source: "https://en.wikipedia.org/wiki/Attention_Is_All_You_Need"
author:
  - "[[Contributors to Wikimedia projects]]"
published: 2023-12-04
created: 2026-04-12
description:
tags:
  - "attention"
---
" **Attention Is All You Need** " [^3] is a 2017 research paper in [machine learning](https://en.wikipedia.org/wiki/Machine_learning "Machine learning") authored by eight scientists working at [Google](https://en.wikipedia.org/wiki/Google "Google"). The paper introduced a new [deep learning](https://en.wikipedia.org/wiki/Deep_learning "Deep learning") architecture known as the [transformer](https://en.wikipedia.org/wiki/Transformer_\(machine_learning_model\) "Transformer (machine learning model)"), based on the [attention mechanism](https://en.wikipedia.org/wiki/Attention_mechanism "Attention mechanism") proposed in 2014 by Bahdanau *et al.*[^4] The transformer approach it describes has become the main architecture of a wide variety of artificial intelligence, including [large language models](https://en.wikipedia.org/wiki/Large_language_model "Large language model").[^5] [^6] At the time, the focus of the research was on improving [Seq2seq](https://en.wikipedia.org/wiki/Seq2seq "Seq2seq") techniques for [machine translation](https://en.wikipedia.org/wiki/Machine_translation "Machine translation"), but the authors go further in the paper, foreseeing the technique's potential for other tasks like [question answering](https://en.wikipedia.org/wiki/Question_answering "Question answering") and what is now known as [multimodal](https://en.wikipedia.org/wiki/Modality_\(human%E2%80%93computer_interaction\) "Modality (human–computer interaction)") [generative AI](https://en.wikipedia.org/wiki/Generative_AI "Generative AI").[^3]

Some early examples that the team tried their Transformer architecture on included English-to-German translation, generating Wikipedia articles on "The Transformer", and [parsing](https://en.wikipedia.org/wiki/Parsing "Parsing"). These convinced the team that the Transformer is a general-purpose language model, and not just good for translation.[^7]

As of 2025, the paper has been cited more than 173,000 times, placing it among the top ten most-cited papers of the 21st century.[^8] After the paper was published by Google, each of the authors left the company to join other companies or to found [startups](https://en.wikipedia.org/wiki/Startup_company "Startup company").

## Background

The authors of the paper are [Ashish Vaswani](https://en.wikipedia.org/wiki/Ashish_Vaswani "Ashish Vaswani"), [Noam Shazeer](https://en.wikipedia.org/wiki/Noam_Shazeer "Noam Shazeer"), Niki Parmar, Jakob Uszkoreit, Llion Jones, [Aidan Gomez](https://en.wikipedia.org/wiki/Aidan_Gomez "Aidan Gomez"), Łukasz Kaiser, and [Illia Polosukhin](https://en.wikipedia.org/wiki/Illia_Polosukhin "Illia Polosukhin"). All eight authors were "equal contributors" to the paper; the listed order was randomized (according to the paper itself). After the paper, each of the authors left Google to join other companies or to found [startups](https://en.wikipedia.org/wiki/Startup_company "Startup company").[^9] [^10]

The paper's title is a reference to the song " [All You Need Is Love](https://en.wikipedia.org/wiki/All_You_Need_Is_Love "All You Need Is Love") " by [the Beatles](https://en.wikipedia.org/wiki/The_Beatles "The Beatles").[^11] The name "Transformer" was picked because Jakob Uszkoreit, one of the paper's authors, liked the sound of that word.[^7] An early design document was titled "Transformers: Iterative Self-Attention and Processing for Various Tasks", and included an illustration of six characters from the *[Transformers](https://en.wikipedia.org/wiki/Transformers "Transformers")* franchise. The team was named Team Transformer.[^11]

## Methods discussed and introduced

The paper is best known for introducing the Transformer architecture, which underlies most modern [large language models](https://en.wikipedia.org/wiki/Large_language_model "Large language model") (LLMs). A key reason why the architecture is preferred by most modern LLMs is the parallelizability of the architecture over its predecessors. This ensures that the operations necessary for training can be accelerated on a GPU, allowing both faster training times and models of bigger sizes to be trained.

The paper introduced the following mechanisms as part of the development of the transformer architecture.

### Scaled dot-product attention and self-attention

The use of the scaled dot-product attention and self-attention mechanism instead of a [recurrent neural network](https://en.wikipedia.org/wiki/Recurrent_neural_network "Recurrent neural network") or [long short-term memory](https://en.wikipedia.org/wiki/Long_short-term_memory "Long short-term memory") (which rely on recurrence instead) allows for better performance as described in the following paragraph. The paper described the scaled dot-product attention as follows:

${\displaystyle {\rm {Attention}}(Q,K,V):={\rm {softmax}}\left({\frac {Q\times K^{T}}{\sqrt {d_{k}}}}\right)\times V}$

where ${\displaystyle Q}$, ${\displaystyle K}$, ${\displaystyle V}$ are respectively the query, key, value matrices, and ${\displaystyle d_{k}}$ is the dimension of the values.

Since the model relies on Query (*Q*), Key (*K*), and Value (*V*) matrices that come from the same source (i.e., the input sequence or context window), this eliminates the need for RNNs, completely ensuring parallelizability for the architecture. This differs from the original form of the Attention mechanism introduced in 2014. Additionally, the paper also discusses the use of an additional scaling factor that was found to be most effective with respect to the dimension of the key vectors (represented as ${\displaystyle d_{k}}$ and initially set to 64 within the paper) in the manner shown above.

In the specific context of translation, which the paper focused on, the Query and Key matrices are usually represented in embeddings corresponding to the source language, while the Value matrix corresponds to the target language.

### Multi-head attention

In the self-attention mechanism, queries (Q), keys (K), and values (V) are dynamically generated for each input sequence (typically limited by the size of the context window), allowing the model to focus on different parts of the input sequence at different steps. Multi-head attention enhances this process by introducing multiple parallel attention heads. Each attention head learns different linear projections of the Q, K, and V matrices. This allows the model to capture different aspects of the relationships between words in the sequence simultaneously, rather than focusing on a single aspect.

By doing this, multi-head attention ensures that the input embeddings are updated from a more varied and diverse set of perspectives. After the attention outputs from all heads are calculated, they are concatenated and passed through a final linear transformation to generate the output.

### Positional encoding

Since the Transformer does not rely on recurrence or convolution of the text in order to perform encoding and decoding, the paper relied on the use of sine and cosine wave functions to encode the position of the token into the embedding. The methods introduced in the paper are discussed below:

${\displaystyle PE_{({\rm {pos}},2i)}=\sin({\rm {pos}}/{10000}^{2i/d_{\rm {model}}})}$

${\displaystyle PE_{({\rm {pos}},2i+1)}=\cos({\rm {pos}}/{10000}^{2i/d_{\rm {model}}})}$

wherein ${\displaystyle {\rm {pos}}}$, ${\displaystyle i}$, ${\displaystyle {d_{\rm {model}}}}$ correspond to the position of the word, the current dimension index, and the dimension of the model, respectively. The sine function is used for even indices of the embedding while the cosine function is used for odd indices. The resultant ${\displaystyle PE}$ embedding is then added to the word at that corresponding position with respect to the current context window. The paper specifically comments on why this method was chosen describing:

"We chose the sinusoidal version because it may allow the model to extrapolate to sequence lengths longer than the ones encountered during training." [^3]

## Historical context

### Predecessors

For many years, sequence modelling and generation was done by using plain [recurrent neural networks](https://en.wikipedia.org/wiki/Recurrent_neural_network "Recurrent neural network") (RNNs). A well-cited early example was the [Elman network](https://en.wikipedia.org/wiki/Elman_network "Elman network") (1990). In theory, the information from one token can propagate arbitrarily far down the sequence, but in practice the [vanishing-gradient problem](https://en.wikipedia.org/wiki/Vanishing-gradient_problem "Vanishing-gradient problem") leaves the model's state at the end of a long sentence without precise, extractable information about preceding tokens.

A key breakthrough was [LSTM](https://en.wikipedia.org/wiki/Long_short-term_memory "Long short-term memory") (1995),[^1] an RNN which used various innovations to overcome the vanishing gradient problem, allowing efficient learning of long-sequence modelling. One key innovation was the use of an [attention mechanism](https://en.wikipedia.org/wiki/Attention_\(machine_learning\) "Attention (machine learning)") which used neurons that multiply the outputs of other neurons, so-called *multiplicative units*.[^12] Neural networks using multiplicative units were later called *sigma-pi networks* [^13] or *[higher-order networks](https://en.wikipedia.org/w/index.php?title=Higher-order_neural_network&action=edit&redlink=1 "Higher-order neural network (page does not exist)")*.[^14] LSTM became the standard architecture for long sequence modelling until the 2017 publication of transformers. However, LSTM still used sequential processing, like most other RNNs.[^2] Specifically, RNNs operate one token at a time from first to last; they cannot operate in parallel over all tokens in a sequence.

Modern transformers overcome this problem, but unlike RNNs, they require computation time that is [quadratic](https://en.wikipedia.org/wiki/Quadratic_function "Quadratic function") in the size of the context window. The linearly scaling [fast weight](https://en.wikipedia.org/w/index.php?title=Fast_weight&action=edit&redlink=1 "Fast weight (page does not exist)") controller (1992) learns to compute a weight matrix for further processing depending on the input.[^15] One of its two networks has "fast weights" or "dynamic links" (1981).[^16] [^17] [^18] A slow neural network learns by gradient descent to generate keys and values for computing the weight changes of the fast neural network which computes answers to queries.[^15] This was later shown to be equivalent to the unnormalized linear transformer.[^19] [^20]

### Attention with seq2seq

The idea of encoder–decoder sequence transduction had been developed in the early 2010s; commonly cited as the originators that produced seq2seq are two concurrently published papers from 2014.[^21] [^22]

A 380M-parameter model for machine translation uses two [long short-term memories](https://en.wikipedia.org/wiki/Long_short-term_memory "Long short-term memory") (LSTM).[^22] Its architecture consists of two parts. The *encoder* is an LSTM that takes in a sequence of tokens and turns it into a vector. The *decoder* is another LSTM that converts the vector into a sequence of tokens. Similarly, another 130M-parameter model used [gated recurrent units](https://en.wikipedia.org/wiki/Gated_recurrent_unit "Gated recurrent unit") (GRU) instead of LSTM.[^21] Later research showed that GRUs are neither better nor worse than LSTMs for seq2seq.[^23] [^24]

These early seq2seq models had no attention mechanism, and the state vector is accessible only after the *last* word of the source text was processed. Although in theory such a vector retains the information about the whole original sentence, in practice the information is poorly preserved. This is because the input is processed sequentially by one recurrent network into a *fixed* -size output vector, which is then processed by another recurrent network into an output. If the input is long, then the output vector would not be able to contain all relevant information, degrading the output. As evidence, reversing the input sentence improved seq2seq translation.[^25]

The *RNN search* model introduced an attention mechanism to seq2seq for machine translation to solve the bottleneck problem (of the *fixed-size* output vector), allowing the model to process long-distance dependencies more easily. The name is because it "emulates searching through a source sentence during decoding a translation".[^4]

The relative performances were compared between global (that of *RNN search*) and local (sliding window) attention model architectures for machine translation, finding that mixed attention had higher quality than global attention, while local attention reduced translation time.[^26]

In 2016, [Google Translate](https://en.wikipedia.org/wiki/Google_Translate "Google Translate") was revamped to [Google Neural Machine Translation](https://en.wikipedia.org/wiki/Google_Neural_Machine_Translation "Google Neural Machine Translation"), which replaced the previous model based on [statistical machine translation](https://en.wikipedia.org/wiki/Statistical_machine_translation "Statistical machine translation"). The new model was a seq2seq model where the encoder and the decoder were both 8 layers of bidirectional LSTM.[^27] It took nine months to develop, and it outperformed the statistical approach, which took ten years to develop.[^28]

### Parallelizing attention

Seq2seq models with attention (including self-attention) still suffered from the same issue with recurrent networks, which is that they are hard to [parallelize](https://en.wikipedia.org/wiki/Parallel_computing "Parallel computing"), which prevented them from being accelerated on GPUs. In 2016, *decomposable attention* applied a self-attention mechanism to [feedforward networks](https://en.wikipedia.org/wiki/Feedforward_neural_network "Feedforward neural network"), which are easy to parallelize, and achieved [SOTA](https://en.wikipedia.org/wiki/State_of_the_art "State of the art") result in [textual entailment](https://en.wikipedia.org/wiki/Textual_entailment "Textual entailment") with an order of magnitude fewer parameters than LSTMs.[^29] One of its authors, Jakob Uszkoreit, suspected that attention *without* recurrence would be sufficient for language translation, thus the title "attention is *all* you need".[^30] That hypothesis was against conventional wisdom at the time, and even his father [Hans Uszkoreit](https://en.wikipedia.org/wiki/Hans_Uszkoreit "Hans Uszkoreit"), a well-known computational linguist, was skeptical.[^30] In the same year, self-attention (called *intra-attention or* *intra-sentence attention*) was proposed for LSTMs.[^31]

On 2017-06-12, the original (100M-parameter) encoder–decoder transformer model was published in the " [Attention is all you need](https://en.wikipedia.org/wiki/Attention_is_all_you_need "Attention is all you need") " paper. At the time, the focus of the research was on improving [seq2seq](https://en.wikipedia.org/wiki/Seq2seq "Seq2seq") for [machine translation](https://en.wikipedia.org/wiki/Machine_translation "Machine translation"), by removing its recurrence to process all tokens in parallel, but preserving its dot-product attention mechanism to keep its text processing performance.[^3] This led to the introduction of a multi-head attention model that was easier to parallelize due to the use of independent heads and the lack of recurrence. Its parallelizability was an important factor to its widespread use in large neural networks.[^32]

### AI boom era

As early as spring 2017, even before the "Attention is all you need" preprint was published, one of the co-authors applied the "decoder-only" variation of the architecture to generate fictitious Wikipedia articles.[^33] Transformer architecture is now used alongside many [generative models](https://en.wikipedia.org/wiki/Generative_artificial_intelligence "Generative artificial intelligence") that contribute to the ongoing [AI boom](https://en.wikipedia.org/wiki/AI_boom "AI boom").

The "reference implementation" of the original Transformer was written in a TensorFlow library.[^34] [^35] In language modelling, [ELMo](https://en.wikipedia.org/wiki/ELMo "ELMo") (2018) was a bi-directional LSTM that produces contextualized [word embeddings](https://en.wikipedia.org/wiki/Word_embedding "Word embedding"), improving upon the line of research from [bag of words](https://en.wikipedia.org/wiki/Bag-of-words_model "Bag-of-words model") and [word2vec](https://en.wikipedia.org/wiki/Word2vec "Word2vec"). It was followed by [BERT](https://en.wikipedia.org/wiki/BERT_\(language_model\) "BERT (language model)") (2018), an encoder-only transformer model.[^36] In October 2019, Google started using BERT to process search queries.[^37] In 2020, Google Translate replaced the previous RNN-encoder–RNN-decoder model by a transformer-encoder–RNN-decoder model.[^38]

Starting in 2018, the OpenAI [GPT series](https://en.wikipedia.org/wiki/Generative_pre-trained_transformer "Generative pre-trained transformer") of decoder-only transformers became state of the art in [natural language generation](https://en.wikipedia.org/wiki/Natural_language_generation "Natural language generation"). In the end of 2022, a chatbot based on GPT-3, [ChatGPT](https://en.wikipedia.org/wiki/ChatGPT "ChatGPT"), became unexpectedly [^39] popular, triggering a boom around [large language models](https://en.wikipedia.org/wiki/Large_language_model "Large language model").[^40] [^41]

Transformers have been applied in modalities beyond text. 4 days after the publication of "Attention is All You Need", a [multimodal](https://en.wikipedia.org/wiki/Multimodal_learning "Multimodal learning") transformer architecture, MultiModel, was published by most authors of that paper.[^42] Other examples include the [vision transformer](https://en.wikipedia.org/wiki/Vision_transformer "Vision transformer"),[^43] speech recognition,[^44] robotics,[^45] and multimodal.[^46] The vision transformer, in turn, stimulated new developments in [convolutional neural networks](https://en.wikipedia.org/wiki/Convolutional_neural_network "Convolutional neural network").[^47] Image and video generators like [DALL-E](https://en.wikipedia.org/wiki/DALL-E "DALL-E") (2021), [Stable Diffusion 3](https://en.wikipedia.org/wiki/Stable_Diffusion "Stable Diffusion") (2024),[^48] and [Sora](https://en.wikipedia.org/wiki/Sora_\(text-to-video_model\) "Sora (text-to-video model)") (2024), use transformers to analyse input data (like text prompts) by breaking it down into "tokens" and then calculating the relevance between each token using self-attention, which helps the model understand the context and relationships within the data.

## Training

While the primary focus of the paper at the time was to improve machine translation, the paper also discussed the use of the architecture on English [Constituency Parsing](https://en.wikipedia.org/wiki/Constituent_\(linguistics\) "Constituent (linguistics)"), both with limited and large-sized datasets, achieving a high-score without specific tuning for the task indicating the promising nature of the model for use in a wide-variety of general purpose of seq2seq tasks.

- Dataset - The English-to-German translation model was trained on the 2014 WMT (Workshop on Statistical Machine Translation) English-German dataset, consisting of nearly 4.5 million sentences derived from TED Talks and high-quality news articles. A separate translation model was trained on the much larger 2014 WMT English-French dataset, consisting of 36 million sentences. Both datasets were encoded with byte-pair encoding.
- Hardware - The models were trained using 8 [NVIDIA P100 GPUs](https://en.wikipedia.org/wiki/Nvidia_Tesla "Nvidia Tesla"). The base models were trained for 100,000 steps, and the big models were trained for 300,000 steps - each step taking about 0.4 seconds to complete for the base models and 1.0 seconds for the big models. The base model was trained for a total of 12 hours, and the big model was trained for a total of 3.5 days. Both the base and big models outperform the 2017 state-of-the-art in both English-German and English-French, while achieving the comparatively lowest training cost.[^3]
- Hyperparameters and regularization - For their 100M-parameter Transformer model, the authors increased the [learning rate](https://en.wikipedia.org/wiki/Learning_rate "Learning rate") linearly for the first 4000 (warmup) steps and decreased it proportionally to the inverse square root of the current step number. Dropout layers were applied to the output of each sub-layer before normalization, the sums of the embeddings, and the positional encodings. The dropout rate was set to 0.1. Label smoothing was applied with a value of 0.1, which "improves accuracy and BLEU score".[^3]

## Notes

[^1]: [Gated recurrent units](https://en.wikipedia.org/wiki/Gated_recurrent_units "Gated recurrent units") (2014) further reduced its complexity.

[^2]: Some architectures, such as RWKV or state space models, avoid the issue.

[^3]: [Vaswani, Ashish](https://en.wikipedia.org/wiki/Ashish_Vaswani "Ashish Vaswani"); [Shazeer, Noam](https://en.wikipedia.org/wiki/Noam_Shazeer "Noam Shazeer"); Parmar, Niki; Uszkoreit, Jakob; Jones, Llion; [Gomez, Aidan N](https://en.wikipedia.org/wiki/Aidan_Gomez "Aidan Gomez"); Kaiser, Łukasz; Polosukhin, Illia (December 2017). ["Attention is All you Need"](https://papers.nips.cc/paper_files/paper/2017/file/3f5ee243547dee91fbd053c1c4a845aa-Paper.pdf) (PDF). In I. Guyon and U. Von Luxburg and S. Bengio and H. Wallach and R. Fergus and S. Vishwanathan and R. Garnett (ed.). *[31st Conference on Neural Information Processing Systems (NIPS)](https://en.wikipedia.org/wiki/Conference_on_Neural_Information_Processing_Systems "Conference on Neural Information Processing Systems")*. Advances in Neural Information Processing Systems. Vol. 30. Curran Associates, Inc. [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1706.03762](https://arxiv.org/abs/1706.03762).

[^4]: Bahdanau, Dzmitry; Cho, Kyunghyun; Bengio, Yoshua (19 May 2016). "Neural Machine Translation by Jointly Learning to Align and Translate". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1409.0473](https://arxiv.org/abs/1409.0473) \[[cs.CL](https://arxiv.org/archive/cs.CL)\].

[^5]: Shinde, Gitanjali; Wasatkar, Namrata; Mahalle, Parikshit (6 June 2024). [*Data-Centric Artificial Intelligence for Multidisciplinary Applications*](https://books.google.com/books?id=tqUIEQAAQBAJ&pg=PA75). [CRC Press](https://en.wikipedia.org/wiki/CRC_Press "CRC Press"). p. 75. [ISBN](https://en.wikipedia.org/wiki/ISBN_\(identifier\) "ISBN (identifier)") [9781040031131](https://en.wikipedia.org/wiki/Special:BookSources/9781040031131 "Special:BookSources/9781040031131").

[^6]: Murgia, Madhumita (23 July 2023). ["Transformers: the Google scientists who pioneered an AI revolution"](https://www.ft.com/content/37bb01af-ee46-4483-982f-ef3921436a50). *[Financial Times](https://en.wikipedia.org/wiki/Financial_Times "Financial Times")*. Retrieved 22 March 2024.

[^7]: Marche, Stephen (23 August 2024). ["Was Linguistic A.I. Created by Accident?"](https://www.newyorker.com/science/annals-of-artificial-intelligence/was-linguistic-ai-created-by-accident). *The New Yorker*. [ISSN](https://en.wikipedia.org/wiki/ISSN_\(identifier\) "ISSN (identifier)") [0028-792X](https://search.worldcat.org/issn/0028-792X). Retrieved 24 August 2024.

[^8]: Pearson, Helen; Ledford, Heidi; Hutson, Matthew; Van Noorden, Richard (15 April 2025). ["Exclusive: the most-cited papers of the twenty-first century"](https://www.nature.com/articles/d41586-025-01125-9). *[Nature](https://en.wikipedia.org/wiki/Nature_\(journal\) "Nature (journal)")*. **640** (8059): 588–592. [Bibcode](https://en.wikipedia.org/wiki/Bibcode_\(identifier\) "Bibcode (identifier)"):[2025Natur.640..588P](https://ui.adsabs.harvard.edu/abs/2025Natur.640..588P). [doi](https://en.wikipedia.org/wiki/Doi_\(identifier\) "Doi (identifier)"):[10.1038/d41586-025-01125-9](https://doi.org/10.1038%2Fd41586-025-01125-9). [PMID](https://en.wikipedia.org/wiki/PMID_\(identifier\) "PMID (identifier)") [40234577](https://pubmed.ncbi.nlm.nih.gov/40234577). Retrieved 18 April 2025.

[^9]: Murgia, Madhumita (23 July 2023). ["Transformers: the Google scientists who pioneered an AI revolution"](https://www.ft.com/content/37bb01af-ee46-4483-982f-ef3921436a50). *Financial Times*. Retrieved 22 March 2025.

[^10]: ["Meet the $4 Billion AI Superstars That Google Lost"](https://www.bloomberg.com/opinion/features/2023-07-13/ex-google-scientists-kickstarted-the-generative-ai-era-of-chatgpt-midjourney). *Bloomberg*. 13 July 2023 – via www.bloomberg.com.

[^11]: Levy, Steven. ["8 Google Employees Invented Modern AI. Here's the Inside Story"](https://www.wired.com/story/eight-google-employees-invented-modern-ai-transformers-paper/). *Wired*. [ISSN](https://en.wikipedia.org/wiki/ISSN_\(identifier\) "ISSN (identifier)") [1059-1028](https://search.worldcat.org/issn/1059-1028). Retrieved 20 March 2024.

[^12]: Feldman, J. A.; Ballard, D. H. (1 July 1982). ["Connectionist models and their properties"](https://www.sciencedirect.com/science/article/pii/S0364021382800013). *Cognitive Science*. **6** (3): 205–254. [doi](https://en.wikipedia.org/wiki/Doi_\(identifier\) "Doi (identifier)"):[10.1016/S0364-0213(82)80001-3](https://doi.org/10.1016%2FS0364-0213%2882%2980001-3). [ISSN](https://en.wikipedia.org/wiki/ISSN_\(identifier\) "ISSN (identifier)") [0364-0213](https://search.worldcat.org/issn/0364-0213).

[^13]: Rumelhart, David E.; McClelland, James L.; Hinton, Geoffrey E. (29 July 1987). [*Parallel Distributed Processing, Volume 1: Explorations in the Microstructure of Cognition: Foundations, Chapter 2*](https://stanford.edu/~jlmcc/papers/PDP/Chapter2.pdf) (PDF). Cambridge, Mass: Bradford Books. [ISBN](https://en.wikipedia.org/wiki/ISBN_\(identifier\) "ISBN (identifier)") [978-0-262-68053-0](https://en.wikipedia.org/wiki/Special:BookSources/978-0-262-68053-0 "Special:BookSources/978-0-262-68053-0").

[^14]: Giles, C. Lee; Maxwell, Tom (1 December 1987). ["Learning, invariance, and generalization in high-order neural networks"](https://opg.optica.org/abstract.cfm?URI=ao-26-23-4972). *Applied Optics*. **26** (23): 4972–4978. [doi](https://en.wikipedia.org/wiki/Doi_\(identifier\) "Doi (identifier)"):[10.1364/AO.26.004972](https://doi.org/10.1364%2FAO.26.004972). [ISSN](https://en.wikipedia.org/wiki/ISSN_\(identifier\) "ISSN (identifier)") [0003-6935](https://search.worldcat.org/issn/0003-6935). [PMID](https://en.wikipedia.org/wiki/PMID_\(identifier\) "PMID (identifier)") [20523475](https://pubmed.ncbi.nlm.nih.gov/20523475).

[^15]: [Schmidhuber, Jürgen](https://en.wikipedia.org/wiki/J%C3%BCrgen_Schmidhuber "Jürgen Schmidhuber") (1992). ["Learning to control fast-weight memories: an alternative to recurrent nets"](https://archive.org/download/wikipedia-scholarly-sources-corpus/10.1162.zip/10.1162%252Fneco.1992.4.1.131.pdf) (PDF). *Neural Computation*. **4** (1): 131–139. [doi](https://en.wikipedia.org/wiki/Doi_\(identifier\) "Doi (identifier)"):[10.1162/neco.1992.4.1.131](https://doi.org/10.1162%2Fneco.1992.4.1.131). [S2CID](https://en.wikipedia.org/wiki/S2CID_\(identifier\) "S2CID (identifier)") [16683347](https://api.semanticscholar.org/CorpusID:16683347).

[^16]: Christoph von der Malsburg: The correlation theory of brain function. Internal Report 81-2, MPI Biophysical Chemistry, 1981. [http://cogprints.org/1380/1/vdM\_correlation.pdf](http://cogprints.org/1380/1/vdM_correlation.pdf) See Reprint in Models of Neural Networks II, chapter 2, pages 95–119. Springer, Berlin, 1994.

[^17]: Jerome A. Feldman, "Dynamic connections in neural networks," Biological Cybernetics, vol. 46, no. 1, pp. 27–39, Dec. 1982.

[^18]: Hinton, Geoffrey E.; Plaut, David C. (1987). ["Using Fast Weights to Deblur Old Memories"](https://escholarship.org/uc/item/0570j1dp). *Proceedings of the Annual Meeting of the Cognitive Science Society*. **9**.

[^19]: Katharopoulos, Angelos; Vyas, Apoorv; Pappas, Nikolaos; Fleuret, François (2020). ["Transformers are RNNs: Fast autoregressive Transformers with linear attention"](https://proceedings.mlr.press/v119/katharopoulos20a.html). *ICML 2020*. PMLR. pp. 5156–5165.

[^20]: Schlag, Imanol; Irie, Kazuki; [Schmidhuber, Jürgen](https://en.wikipedia.org/wiki/Juergen_Schmidhuber "Juergen Schmidhuber") (2021). "Linear Transformers Are Secretly Fast Weight Programmers". *ICML 2021*. Springer. pp. 9355–9366.

[^21]: Cho, Kyunghyun; van Merriënboer, Bart; Gulcehre, Caglar; Bahdanau, Dzmitry; Bougares, Fethi; Schwenk, Holger; Bengio, Yoshua (October 2014). ["Learning Phrase Representations using RNN Encoder–Decoder for Statistical Machine Translation"](https://aclanthology.org/D14-1179). In Moschitti, Alessandro; Pang, Bo; Daelemans, Walter (eds.). *Proceedings of the 2014 Conference on Empirical Methods in Natural Language Processing (EMNLP)*. Doha, Qatar: Association for Computational Linguistics. pp. 1724–1734. [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1406.1078](https://arxiv.org/abs/1406.1078). [doi](https://en.wikipedia.org/wiki/Doi_\(identifier\) "Doi (identifier)"):[10.3115/v1/D14-1179](https://doi.org/10.3115%2Fv1%2FD14-1179).

[^22]: Sutskever, Ilya; Vinyals, Oriol; Le, Quoc Viet (14 December 2014). "Sequence to sequence learning with neural networks". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1409.3215](https://arxiv.org/abs/1409.3215) \[[cs.CL](https://arxiv.org/archive/cs.CL)\]. \[first version posted to arXiv on 10 Sep 2014\]

[^23]: Chung, Junyoung; Gulcehre, Caglar; Cho, KyungHyun; Bengio, Yoshua (2014). "Empirical Evaluation of Gated Recurrent Neural Networks on Sequence Modeling". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1412.3555](https://arxiv.org/abs/1412.3555) \[[cs.NE](https://arxiv.org/archive/cs.NE)\].

[^24]: Gruber, N.; Jockisch, A. (2020), "Are GRU cells more specific and LSTM cells more sensitive in motive classification of text?", *Frontiers in Artificial Intelligence*, **3** 40, [doi](https://en.wikipedia.org/wiki/Doi_\(identifier\) "Doi (identifier)"):[10.3389/frai.2020.00040](https://doi.org/10.3389%2Ffrai.2020.00040), [PMC](https://en.wikipedia.org/wiki/PMC_\(identifier\) "PMC (identifier)") [7861254](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC7861254), [PMID](https://en.wikipedia.org/wiki/PMID_\(identifier\) "PMID (identifier)") [33733157](https://pubmed.ncbi.nlm.nih.gov/33733157), [S2CID](https://en.wikipedia.org/wiki/S2CID_\(identifier\) "S2CID (identifier)") [220252321](https://api.semanticscholar.org/CorpusID:220252321)

[^25]: Sutskever, Ilya; Vinyals, Oriol; Le, Quoc V (2014). ["Sequence to Sequence Learning with Neural Networks"](https://proceedings.neurips.cc/paper/2014/hash/a14ac55a4f27472c5d894ec1c3c743d2-Abstract.html). *Advances in Neural Information Processing Systems*. **27**. Curran Associates, Inc. [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1409.3215](https://arxiv.org/abs/1409.3215).

[^26]: Luong, Minh-Thang; Pham, Hieu; Manning, Christopher D. (2015). "Effective Approaches to Attention-based Neural Machine Translation". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1508.04025](https://arxiv.org/abs/1508.04025) \[[cs.CL](https://arxiv.org/archive/cs.CL)\].

[^27]: Wu, Yonghui; et al. (1 September 2016). "Google's Neural Machine Translation System: Bridging the Gap between Human and Machine Translation". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1609.08144](https://arxiv.org/abs/1609.08144) \[[cs.CL](https://arxiv.org/archive/cs.CL)\].

[^28]: Lewis-Kraus, Gideon (14 December 2016). ["The Great A.I. Awakening"](https://web.archive.org/web/20230524052626/https://www.nytimes.com/2016/12/14/magazine/the-great-ai-awakening.html). *The New York Times*. [ISSN](https://en.wikipedia.org/wiki/ISSN_\(identifier\) "ISSN (identifier)") [0362-4331](https://search.worldcat.org/issn/0362-4331). Archived from [the original](https://www.nytimes.com/2016/12/14/magazine/the-great-ai-awakening.html) on 24 May 2023. Retrieved 22 June 2023.

[^29]: Parikh, Ankur P.; Täckström, Oscar; Das, Dipanjan; Uszkoreit, Jakob (25 September 2016). "A Decomposable Attention Model for Natural Language Inference". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1606.01933](https://arxiv.org/abs/1606.01933) \[[cs.CL](https://arxiv.org/archive/cs.CL)\].

[^30]: Levy, Steven. ["8 Google Employees Invented Modern AI. Here's the Inside Story"](https://www.wired.com/story/eight-google-employees-invented-modern-ai-transformers-paper/). *Wired*. [ISSN](https://en.wikipedia.org/wiki/ISSN_\(identifier\) "ISSN (identifier)") [1059-1028](https://search.worldcat.org/issn/1059-1028). [Archived](https://web.archive.org/web/20240320101528/https://www.wired.com/story/eight-google-employees-invented-modern-ai-transformers-paper/) from the original on 20 March 2024. Retrieved 6 August 2024.

[^31]: Cheng, Jianpeng; Dong, Li; Lapata, Mirella (November 2016). ["Long Short-Term Memory-Networks for Machine Reading"](https://aclanthology.org/D16-1053/). In Su, Jian; Duh, Kevin; Carreras, Xavier (eds.). *Proceedings of the 2016 Conference on Empirical Methods in Natural Language Processing*. Austin, Texas: Association for Computational Linguistics. pp. 551–561. [doi](https://en.wikipedia.org/wiki/Doi_\(identifier\) "Doi (identifier)"):[10.18653/v1/D16-1053](https://doi.org/10.18653%2Fv1%2FD16-1053).

[^32]: Peng, Bo; Alcaide, Eric; Anthony, Quentin; Albalak, Alon; Arcadinho, Samuel; Biderman, Stella; Cao, Huanqi; Cheng, Xin; Chung, Michael (10 December 2023), *RWKV: Reinventing RNNs for the transformer Era*, [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[2305.13048](https://arxiv.org/abs/2305.13048)

[^33]: Marche, Stephen (23 August 2024). ["Was Linguistic A.I. Created by Accident?"](https://www.newyorker.com/science/annals-of-artificial-intelligence/was-linguistic-ai-created-by-accident). *The New Yorker*. [ISSN](https://en.wikipedia.org/wiki/ISSN_\(identifier\) "ISSN (identifier)") [0028-792X](https://search.worldcat.org/issn/0028-792X). Retrieved 27 August 2024.

[^34]: Vaswani, Ashish; Bengio, Samy; Brevdo, Eugene; Chollet, Francois; Gomez, Aidan; Gouws, Stephan; Jones, Llion; Kaiser, Łukasz; Kalchbrenner, Nal; Parmar, Niki; Sepassi, Ryan; Shazeer, Noam; Uszkoreit, Jakob (March 2018). Cherry, Colin; Neubig, Graham (eds.). ["Tensor2Tensor for Neural Machine Translation"](https://aclanthology.org/W18-1819/). *Proceedings of the 13th Conference of the Association for Machine Translation in the Americas (Volume 1: Research Track)*. Boston, MA: Association for Machine Translation in the Americas: 193–199.

[^35]: Kaiser, Łukasz (19 June 2017). ["Accelerating Deep Learning Research with the Tensor2Tensor Library"](https://research.google/blog/accelerating-deep-learning-research-with-the-tensor2tensor-library/). *Google Research Blog*. Google.

[^36]: Devlin, Jacob; Chang, Ming-Wei; Lee, Kenton; Toutanova, Kristina (11 October 2018). "BERT: Pre-training of Deep Bidirectional Transformers for Language Understanding". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[1810.04805v2](https://arxiv.org/abs/1810.04805v2) \[[cs.CL](https://arxiv.org/archive/cs.CL)\].

[^37]: ["Google: BERT now used on almost every English query"](https://searchengineland.com/google-bert-used-on-almost-every-english-query-342193). *Search Engine Land*. 15 October 2020. Retrieved 24 November 2020.

[^38]: Caswell, Isaac; Liang, Bowen (8 June 2020). ["Recent Advances in Google Translate"](https://research.google/blog/recent-advances-in-google-translate/). *Google Research*. [Archived](https://web.archive.org/web/20240704042433/https://research.google/blog/recent-advances-in-google-translate/) from the original on 4 July 2024. Retrieved 7 August 2024.

[^39]: ["The inside story of how ChatGPT was built from the people who made it"](https://www.technologyreview.com/2023/03/03/1069311/inside-story-oral-history-how-chatgpt-built-openai/). *MIT Technology Review*. Retrieved 6 August 2024.

[^40]: ["Improving language understanding with unsupervised learning"](https://openai.com/research/language-unsupervised). *openai.com*. 11 June 2018. [Archived](https://web.archive.org/web/20230318210736/https://openai.com/research/language-unsupervised) from the original on 18 March 2023. Retrieved 18 March 2023.

[^41]: [*finetune-transformer-lm*](https://github.com/openai/finetune-transformer-lm), OpenAI, 11 June 2018, retrieved 1 May 2023

[^42]: Kaiser, Lukasz; Gomez, Aidan N.; Shazeer, Noam; Vaswani, Ashish; Parmar, Niki; Jones, Llion; Uszkoreit, Jakob (16 June 2017). ["One Model To Learn Them All"](https://arxiv.org/abs/1706.05137v1). *arXiv.org*.

[^43]: Dosovitskiy, Alexey; Beyer, Lucas; Kolesnikov, Alexander; Weissenborn, Dirk; Zhai, Xiaohua; Unterthiner, Thomas; Dehghani, Mostafa; Minderer, Matthias; Heigold, Georg; Gelly, Sylvain; Uszkoreit, Jakob (3 June 2021). "An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[2010.11929](https://arxiv.org/abs/2010.11929) \[[cs.CV](https://arxiv.org/archive/cs.CV)\].

[^44]: Gulati, Anmol; Qin, James; Chiu, Chung-Cheng; Parmar, Niki; Zhang, Yu; Yu, Jiahui; Han, Wei; Wang, Shibo; Zhang, Zhengdong; Wu, Yonghui; Pang, Ruoming (2020). "Conformer: Convolution-augmented Transformer for Speech Recognition". [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[2005.08100](https://arxiv.org/abs/2005.08100) \[[eess.AS](https://arxiv.org/archive/eess.AS)\].

[^45]: Chen, Lili; Lu, Kevin; Rajeswaran, Aravind; Lee, Kimin; Grover, Aditya; Laskin, Michael; Abbeel, Pieter; Srinivas, Aravind; Mordatch, Igor (24 June 2021), *Decision Transformer: Reinforcement Learning via Sequence Modeling*, [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[2106.01345](https://arxiv.org/abs/2106.01345)

[^46]: Choromanski, Krzysztof; Likhosherstov, Valerii; Dohan, David; Song, Xingyou; Gane, Andreea; Sarlos, Tamas; Hawkins, Peter; Davis, Jared; Mohiuddin, Afroz (19 November 2022), *Rethinking Attention with Performers*, [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[2009.14794](https://arxiv.org/abs/2009.14794)

[^47]: Liu, Zhuang; Mao, Hanzi; Wu, Chao-Yuan; Feichtenhofer, Christoph; Darrell, Trevor; Xie, Saining (2022). [*A ConvNet for the 2020s*](https://openaccess.thecvf.com/content/CVPR2022/html/Liu_A_ConvNet_for_the_2020s_CVPR_2022_paper.html). Conference on Computer Vision and Pattern Recognition ([CVPR](https://en.wikipedia.org/wiki/CVPR "CVPR")). pp. 11976–11986.

[^48]: Esser, Patrick; Kulal, Sumith; Blattmann, Andreas; Entezari, Rahim; Müller, Jonas; Saini, Harry; Levi, Yam; Lorenz, Dominik; Sauer, Axel (5 March 2024), *Scaling Rectified Flow Transformers for High-Resolution Image Synthesis*, [arXiv](https://en.wikipedia.org/wiki/ArXiv_\(identifier\) "ArXiv (identifier)"):[2403.03206](https://arxiv.org/abs/2403.03206)