# Tabular Deep Learning Models in R


I’d like to have more deep learning models available in R.

Existing examples:

- TabNet via <https://mlverse.github.io/tabnet/>
- TabPFN via <https://tabpfn.tidymodels.org/>
- Variational autoencoders via
  <https://github.com/SarahMilligan-hub/AutoTab>

The goal is to have idiomatic R APIs.

There are two strategies for making tabular DL models available in R:

- Local R implementation using torch or keras (preferably the former).
- Wrapping a Python API via reticulate.

A local implementation is preferred, but it’s also likely to be the
exception, since the Python packages might have other dependencies we
don’t want to deal with. [TabNet](https://mlverse.github.io/tabnet/) is
a great example of this.

For the latter, we should make it easy for users to install the
appropriate Python packages, add model checkpointing, implement proper
serialization for deployment, etc.

Below is a list of potential models. We should pick through these
judiciously.

I also plan on doing a lot more with TabPFN:

- Model checkpointing (and caching)
- Enable some of the [extension
  packages](https://github.com/priorlabs/tabpfn-extensions).
- Proper tidymodels integration, including quantile and censored
  regression models.
- More examples and benchmarking.

### Regularization Learning Networks

References:

- Shavitt, I., & Segal, E. (2018).[Regularization learning networks:
  deep learning for tabular
  datasets](https://proceedings.neurips.cc/paper_files/paper/2018/file/500e75a036dc2d7d2fec5da1b71d36cc-Paper.pdf).
  Advances in neural information processing systems, 31.

Implementations:

- <https://github.com/irashavitt/regularization_learning_networks>

### Automatic Feature Interaction Learning via Self-Attentive Neural Networks (AutoInt)

References:

- Song, W., Shi, C., Xiao, Z., Duan, Z., Xu, Y., Zhang, M., & Tang, J.
  (2019, November). [Autoint: Automatic feature interaction learning via
  self-attentive neural
  networks](https://dl.acm.org/doi/abs/10.1145/3357384.3357925). In
  Proceedings of the 28th ACM international conference on information
  and knowledge management (pp. 1161-1170).

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22autoInt%22&btnG=)

Implementations:

- <https://pytorch-tabular.readthedocs.io/en/latest/>

### Neural Oblivious Decision Ensembles (NODE)

References:

- Popov, S., Morozov, S., & Babenko, A. (2019). [Neural oblivious
  decision ensembles for deep learning on tabular
  data](https://arxiv.org/abs/1909.06312). arXiv preprint
  arXiv:1909.06312.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22Neural+Oblivious+Decision+Ensembles%22&btnG=)

Implementations:

- <https://pytorch-tabular.readthedocs.io/en/latest/>
- <https://github.com/Qwicen/node/tree/master>

### Gated Additive Tree Ensemble

References:

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22Gated+Additive+Tree+Ensemble%22&btnG=)

Implementations:

- <https://pytorch-tabular.readthedocs.io/en/latest/>

### TabTransformer

References:

- Huang, X., Khetan, A., Cvitkovic, M., & Karnin, Z. (2020).
  [Tabtransformer: Tabular data modeling using contextual embeddings]().
  arXiv preprint arXiv:2012.06678.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22TabTransformer%22+%22Huang%22&btnG=)

Implementations:

- <https://github.com/lucidrains/tab-transformer-pytorch>

### Gated Adaptive Network for Deep Automated Learning of Features (GANDALF)

References:

- Joseph, M., & Raj, H. (2022). [GANDALF: gated adaptive network for
  deep automated learning of
  features](https://arxiv.org/abs/2207.08548). arXiv preprint
  arXiv:2207.08548.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22Gated+Adaptive+Network+for+Deep+Automated+Learning+of+Features%22+GANDALF&btnG=)

Implementations:

- <https://pytorch-tabular.readthedocs.io/en/latest/>

### DANETs: Deep Abstract Networks for Tabular Data Classification and Regression

References:

- Chen, J., Liao, K., Wan, Y., Chen, D. Z., & Wu, J. (2022, June).
  [Danets: Deep abstract networks for tabular data classification and
  regression](https://ojs.aaai.org/index.php/AAAI/article/view/20309).
  In Proceedings of the AAAI Conference on Artificial Intelligence (Vol.
  36, No. 4, pp. 3930-3938).

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=title%3A+%22DANETs%22&btnG=)

Implementations:

- <https://pytorch-tabular.readthedocs.io/en/latest/>

### TabR

- Gorishniy, Y., Rubachev, I., Kartashev, N., Shlenskii, D., Kotelnikov,
  A., & Babenko, A. (2023). [Tabr: Tabular deep learning meets nearest
  neighbors in 2023](https://arxiv.org/abs/2307.14338). arXiv preprint
  arXiv:2307.14338.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=Tabr+%22Gorishniy%22&btnG=)

Implementations:

- <https://pypi.org/project/pytorch-tabr/>
- <https://github.com/yandex-research/tabular-dl-tabr>

### Tangos

References:

- Jeffares, A., Liu, T., Crabbé, J., Imrie, F., & van der Schaar, M.
  (2023). [TANGOS: Regularizing tabular neural networks through gradient
  orthogonalization and
  specialization.](https://arxiv.org/pdf/2303.05506) arXiv preprint
  arXiv:2303.05506.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22Tangos%22+%22Jeffares%22&btnG=)

Implementations:

- <https://github.com/alanjeffares/TANGOS>

### Trompt

References:

- Chen, K. Y., Chiang, P. H., Chou, H. R., Chen, T. W., & Chang, T. H.
  (2023). [Trompt: Towards a better deep neural network for tabular
  data](https://arxiv.org/pdf/2305.18446). arXiv preprint
  arXiv:2305.18446.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22Trompt%22+%22Chen%22&btnG=)

Implementations:

- 

### Mambular

References:

- Thielmann, A. F., Kumar, M., Weisser, C., Reuter, A., Säfken, B., &
  Samiee, S. (2024). [Mambular: A sequential model for tabular deep
  learning](https://arxiv.org/pdf/2408.06291). arXiv preprint
  arXiv:2408.06291.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22Mambular%22&btnG=)

Implementations:

- <https://github.com/Ovler-Young/DeepTabular>

### TabM

References:

- Gorishniy, Y., Kotelnikov, A., & Babenko, A. (2024). [Tabm: Advancing
  tabular deep learning with parameter-efficient
  ensembling](https://arxiv.org/pdf/2410.24210?). arXiv preprint
  arXiv:2410.24210.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=%22TabM%22+%22Gorishniy%22&btnG=)

Implementations:

- <https://github.com/OpenTabular/DeepTab>
- <https://mambular.readthedocs.io/en/latest/>

### ModernNCA

References:

- Ye, H. J., Yin, H. H., Zhan, D. C., & Chao, W. L. (2024). [Revisiting
  nearest neighbor for tabular data: A deep tabular baseline two decades
  later](https://arxiv.org/pdf/2407.03257). arXiv preprint
  arXiv:2407.03257.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=ModernNCA+%22ye%22&btnG=)

### ResNet

References:

- Gorishniy, Y., Rubachev, I., Khrulkov, V., & Babenko, A. (2021).
  [Revisiting deep learning models for tabular
  data](https://proceedings.neurips.cc/paper_files/paper/2021/hash/9d86d83f925f2149e9edb0ac3b49229c-Abstract.html).
  Advances in neural information processing systems, 34, 18932-18943.
  *see Equation 2*

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=tabular+resnet+Gorishniy&btnG=)

Implementations:

- Brulee (currently in a branch):
  <https://github.com/tidymodels/brulee/tree/resnet>

### iLTM: Integrated Large Tabular Model

References:

- Bonet, D., Cara, M. C., Calafell, A., Montserrat, D. M., &
  Ioannidis, A. G. (2025). [iLTM: Integrated Large Tabular
  Model](#iltm-integrated-large-tabular-model)(https://arxiv.org/abs/2511.15941.
  arXiv preprint arXiv:2511.15941.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=iLTM+Bonet&btnG=)

Implementations:

- <https://github.com/AI-sandbox/iLTM>
- <https://github.com/frankiethull/iltm>

### TabICLv2

References:

- Qu, J., Holzmüller, D., Varoquaux, G., & Morvan, M. L. (2026).
  [TabICLv2: A better, faster, scalable, and open tabular foundation
  model](https://arxiv.org/abs/2602.11139). arXiv preprint
  arXiv:2602.11139.

- [Scholar](https://scholar.google.com/scholar?hl=en&as_sdt=0%2C7&q=TabICLv2&btnG=)

Implementations:

- <https://github.com/soda-inria/tabicl>
- <https://github.com/frankiethull/tabicl>
- <https://github.com/cregouby/TabICL2>

### sap-rpt-1-oss (fka ConTextTab)

References:

- Marco Spinaci, Marek Polewczyk, Maximilian Schambach, Sam Thelin
  (2025). [ConTextTab: A Semantics-Aware Tabular In-Context
  Learner](https://arxiv.org/abs/2506.10707). arXiv preprint
  arXiv:2506.10707.

Implementations:

- <https://github.com/SAP-samples/sap-rpt-1-oss>
- <https://github.com/frankiethull/contexttab>

### TabDPT

References:

- Junwei Ma and Valentin Thomas and Rasa Hosseinzadeh and Alex Labach
  and Hamidreza Kamkari and Jesse C. Cresswell and Keyvan Golestan and
  Guangwei Yu and Anthony L. Caterini and Maksims Volkovs (2025).
  [TabDPT: Scaling Tabular Foundation Models on Real
  Data](https://arxiv.org/abs/2410.18164). arXiv preprint
  arXiv:2410.18164.

Implementations:

- <https://github.com/layer6ai-labs/TabDPT-inference>
- <https://github.com/frankiethull/tabdpt>

### LimiX

References:

- Zhang, Xingxuan and Ren, Gang and Yu, Han and Yuan, Hao and Wang, Hui
  and Li, Jiansheng and Wu, Jiayun and Mo, Lang and Mao, Li and Hao,
  Mingchao and others (2025). [LimiX: Unleashing Structured-Data
  Modeling Capability for Generalist
  Intelligence](https://arxiv.org/abs/2509.03505). arXiv preprint
  arXiv:2509.03505.

Implementations:

- <https://github.com/limix-ldm-ai/LimiX>
- <https://github.com/frankiethull/limix>

### Mitra

References:

- Xiyuan Zhang, Danielle C. Maddix, Junming Yin, Nick Erickson, Abdul
  Fatir Ansari, Boran Han, Shuai Zhang, Leman Akoglu, Christos
  Faloutsos, Michael W. Mahoney, Cuixiong Hu, Huzefa Rangwala, George
  Karypis, Bernie Wang (2025). [Mitra: Mixed Synthetic Priors for
  Enhancing Tabular Foundation
  Models](https://arxiv.org/abs/2510.21204). arXiv preprint
  arXiv:2510.21204.

Implementations:

- <https://github.com/autogluon/autogluon>
- <https://github.com/frankiethull/mitra>

### Orion-Bix

References:

- Mohamed Bouadi, Pratinav Seth, Aditya Tanna, Vinay Kumar Sankarapu
  (2025).[Orion-Bix: Bi-Axial Attention for Tabular In-Context
  Learning](https://arxiv.org/abs/2512.00181). arXiv preprint
  arXiv:2512.00181.

Implementations:

- <https://github.com/Lexsi-Labs/Orion-BiX>
- <https://github.com/frankiethull/orion>

### Orion-MSP

References:

- Mohamed Bouadi, Pratinav Seth, Aditya Tanna, Vinay Kumar Sankarapu
  (2025). [Orion-MSP: Multi-Scale Sparse Attention for Tabular
  In-Context Learning](https://arxiv.org/abs/2511.02818). arXiv preprint
  arXiv:2511.02818.

Implementations:

- <https://github.com/Lexsi-Labs/Orion-MSP>
- <https://github.com/frankiethull/orion>

### RealMLP

References:

- David Holzmüller, Léo Grinsztajn, Ingo Steinwart (2024). [Better by
  Default: Strong Pre-Tuned MLPs and Boosted Trees on Tabular
  Data](https://arxiv.org/abs/2407.04491). arXiv preprint
  arXiv:2407.04491.

Implementations:

- <https://github.com/dholzmueller/realmlp-td-s_standalone>
- <https://github.com/dholzmueller/pytabkit>
- <https://github.com/frankiethull/realmlp>

### KAN

References:

- Ziming Liu, Yixuan Wang, Sachin Vaidya, Fabian Ruehle, James
  Halverson, Marin Soljačić, Thomas Y. Hou, Max Tegmark (2024). [KAN:
  Kolmogorov-Arnold Networks](https://arxiv.org/abs/2404.19756). arXiv
  preprint arXiv:2404.19756.

Implementations:

- <https://github.com/kindxiaoming/pykan>
- <https://github.com/vpuri3/KolmogorovArnold.jl>
