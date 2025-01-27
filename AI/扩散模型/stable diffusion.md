# Diffusion Model

一次到位变为n次到位,增强稳定性

将任意维度数据映射到标准高斯分布的模型。

## 条件图像生成：引导扩散

在每一步扩散中添加条件信息y
$$
p_\theta(x_{0:T}|y)=p_\theta(x_T)\prod_{t=1}^Tp_\theta( x_{t-1}|x_t,y)
$$
一般来讲，引导扩散目标学习$\nabla \log p_\theta(x_t|y)$。因此使用贝叶斯规则，可以写出
$$
\begin{eqnarray}
\nabla_{x_t} log p_\theta(x_t|y)&=&\nabla_{x_t}\log (\frac{p_\theta(y|x_t)p_\theta(x_t)}{p_\theta (y)})\\
&=&\nabla_{x_t}\log p_\theta(x_t)+\nabla_{x_t}\log(p_\theta(y|x_t))
\end{eqnarray}
$$
$p_\theta(y)$因为和$x_t$无关，求梯度被移除，因此关于y没有梯度。

通过添加高斯标量项s，有

$\nabla \log p_\theta(x_t|y)=\nabla \log p_\theta(x_t)+s \cdot \nabla log(p_\theta(y|x_t))$

根据这个公式，让我们区分分类器和无分类器指导。因为执行最大似然估计可以分为两个部分，一个原先推导过的$\ log p_\theta(x_t)$,另一个则是$\log (p_\theta(y|x_t))$。

### 分类器引导

[Sohl-Dickstein et al](https://arxiv.org/abs/1503.03585). 以及最近[Dhariwal and Nichol](https://arxiv.org/abs/2105.05233)表示，可以使用第二个模型，一个分类器$f_\phi(y|x_t,t)$来引导，训练时扩散的方向到目标类y中。

为了实现它，我们可以训练一个分类器$f_\phi(y|x_t,t)$用于给定噪声图像$x_t$预测类别y。因此，我们可以使用梯度$\nabla \log (f_\phi(y|x_t))$类引导扩散。



我们使用均值$\mu_\theta(x_t|y)$以及方差$\Sigma_\theta(x_t|y)$构建一个条件类别扩散模型。

因为$p_\theta \sim \mathcal{N(\mu_\theta,\Sigma_\theta)}$，根据先前部分的梯度方程，我们可以表示均值为带标签y的$log f_\phi(y|x_t)$的梯度，结果为
$$
\hat \mu(x_t|y)=\mu_\theta(x_t|y)+s\cdot \Sigma_\theta(x_t|y)\nabla_{x_t}\log f_\phi(y|x_t,t)
$$
在[GLIDE paper by Nichol et al](https://arxiv.org/abs/2112.10741)中，作者扩展了这个想法，并且使用CLIP embeddings来引导扩散。

CLIP被[Saharia et al.](https://arxiv.org/abs/2205.11487)建议,由一个图片encoder g和文本encoder h组成。他减少了图像编码$g(x_t)$和文本编码$h(c)$.因此，我们可以改变梯度下降方式为$\hat \mu(x_t|c)=\mu(x_t|c)+s \cdot \Sigma_\theta(x_t|c)\nabla_{x_t}g(x_t)\cdot h(c)$.

最终，他们设法将生成过程引导到用户定义的文本标题上。

![image-20230615220230621](./stable%20diffusion.assets/image-20230615220230621.png)

### 无分类器引导

无分类器引导扩散模型目标可以定义为

$\nabla \log p(x_t|y)=s \cdot \nabla \log(p(x_t|y))+(1-s)\cdot \nabla \log p(x_t)$

梯度可以被表达为没有第二个分类器的模型，由[Ho & Salimans](https://openreview.net/forum?id=qw8AKxfYbI)提出。

并非训练一个模型作为分类器，作者训练了一个条件扩散模型 $\epsilon_\theta(x_t|y)$和无条件模型$\epsilon_\theta(x_t|0)$结合到了一起。事实上他们使用相同的神经网络。在训练期间，他们随机的设置类别y为0，以至于模型接受有条件和无条件的设置
$$
\begin{eqnarray}
\hat \epsilon_\theta &=& s \cdot \epsilon_\theta (x_t|y)+(1-s)\cdot \epsilon_\theta(x_t|0) \\
&=&\epsilon_\theta(x_t|0)+s \cdot (\epsilon_\theta(x_t|y)-\epsilon_\theta(x_t|0))
\end{eqnarray}
$$
这个模型有两个有点：

- 使用了单一模型来引导扩散
- 简化了当文本很难预测为一个分类器如Text embeddings时的引导

Imagen被 [Saharia et al](https://arxiv.org/abs/2205.11487)提出，使用了无分类器扩散结构，他们发现最重要的贡献是强文本图像关联对

## 放大扩散模型

扩散模型在高分辨率的情形下，U-net的计算成本十分昂贵，由两种方法将扩散模型扩展到高分辨率情况：

- 级联扩散模型
- 隐扩散模型

### 级联扩散模型

[Ho et al. 2021](https://arxiv.org/abs/2106.15282) 提出了级联扩散模型以生产高保真图像。

一个级联扩散模型由一连串的扩散模型流水线组成，来提高生成图像的分辨率。每一个模型通过连续对图像上采样并添加更高分辨率的细节，生成质量优于前一个样本。

为了通过级联架构获得良好的结果，对每个超分辨率模型的输入进行强大的数据增强是至关重要的。因为它减轻了先前级联模型的复合误差，以及由于训练测试不匹配造成的。结果发现，高斯模糊是实现高保真度的关键转变。他们将这种技术称为条件增强。

### 隐扩散模型

潜在扩散模型基于一个相当简单的想法：我们不是将扩散过程直接应用于高维输入，而是将输入投影到较小的潜在空间并在那里应用扩散。

对于更多细节，[Rombach et al](https://arxiv.org/abs/2112.10752).提出了编码网络，来将输入编码到隐空间表示。这个决定背后的思想更小的

决定背后的直觉是通过在较低维空间中处理输入来降低训练扩散模型的计算需求。之后，应用标准扩散模型 (U-Net) 生成新数据，这些数据由解码器网络进行上采样。

![image-20230522155757606](./stable%20diffusion.assets/image-20230522155757606.png)

#### 计算量

在图片尺寸位512*512下，双精度，stable diffusion计算量

| 模型名           | 参数量      | 占用空间 |
| ---------------- | ----------- | -------- |
| image_encoder    | 34,163,664  | 260MB    |
| image_decoder    | 49,490,199  | 377MB    |
| text_transformer | 123,060,480 | 938MB    |
| diffusion_model  | 859,520,964 | 6557MB   |

总占用约8个G

#### TextTransformer tensorflow 实现

使用基于Transofrmer的文本编码器，以实现条件控制，文本生成图像，使用CLIP的结构，可以使用预训练参数

```python
import tensorflow as tf
from tensorflow import keras
import numpy as np

# 默认使用CLIP配置

class TextEmbeddingBlock(keras.layers.Layer):
    """
        文本编码块
        输入为文本的token id和位置id (batch_size,2,max_text_size),
        输出为文本的embedding编码结果 (batch_size,max_text_size,embedding_size)
    """

    def __init__(self, voc_size=49408, max_text_size=77, embedding_dim=768):
        super().__init__()
        self.token_embedding = keras.layers.Embedding(
            voc_size, embedding_dim, name="token_embedding")
        self.position_embedding = keras.layers.Embedding(
            max_text_size, embedding_dim, name="position_embedding")

    def call(self, inputs):
        input_ids, position_ids = inputs
        return self.token_embedding(input_ids) + self.position_embedding(position_ids)


class CLIPAttention(keras.layers.Layer):
    def __init__(self):
        super().__init__()
        self.embed_dim = 768
        self.num_heads = 12
        self.head_dim = self.embed_dim // self.num_heads
        self.scale = self.head_dim**-0.5
        self.q_proj = keras.layers.Dense(self.embed_dim)
        self.k_proj = keras.layers.Dense(self.embed_dim)
        self.v_proj = keras.layers.Dense(self.embed_dim)
        self.out_proj = keras.layers.Dense(self.embed_dim)

    def _shape(self, tensor, seq_len: int, bsz: int):
        a = tf.reshape(tensor, (bsz, seq_len, self.num_heads, self.head_dim))
        # bs , n_head , seq_len , head_dim
        return keras.layers.Permute((2, 1, 3))(a)

    def call(self, inputs):
        hidden_states, causal_attention_mask = inputs
        bsz, tgt_len, embed_dim = hidden_states.shape
        query_states = self.q_proj(hidden_states) * self.scale
        key_states = self._shape(self.k_proj(hidden_states), tgt_len, -1)
        value_states = self._shape(self.v_proj(hidden_states), tgt_len, -1)

        proj_shape = (-1, tgt_len, self.head_dim)
        query_states = self._shape(query_states, tgt_len, -1)
        query_states = tf.reshape(query_states, proj_shape)
        key_states = tf.reshape(key_states, proj_shape)

        src_len = tgt_len
        value_states = tf.reshape(value_states, proj_shape)
        attn_weights = query_states @ keras.layers.Permute((2, 1))(key_states)

        attn_weights = tf.reshape(
            attn_weights, (-1, self.num_heads, tgt_len, src_len))
        attn_weights = attn_weights + causal_attention_mask
        attn_weights = tf.reshape(attn_weights, (-1, tgt_len, src_len))

        attn_weights = tf.nn.softmax(attn_weights)
        attn_output = attn_weights @ value_states

        attn_output = tf.reshape(
            attn_output, (-1, self.num_heads, tgt_len, self.head_dim)
        )
        attn_output = keras.layers.Permute((2, 1, 3))(attn_output)
        attn_output = tf.reshape(attn_output, (-1, tgt_len, embed_dim))

        return self.out_proj(attn_output)


class TextEncoderBlock(keras.layers.Layer):
    """
        使用attention机制生成文本编码结果
    Args:
        keras (_type_): _description_
    """

    def __init__(self, embedding_dim=768):
        super().__init__()
        self.embedding_dim = embedding_dim

        self.layer_norm1 = keras.layers.LayerNormalization(epsilon=1e-5)
        self.multi_head_attention = CLIPAttention()

        self.layer_norm2 = keras.layers.LayerNormalization(epsilon=1e-5)

        self.dense1 = keras.layers.Dense(4 * self.embedding_dim)
        self.dense2 = keras.layers.Dense(self.embedding_dim)

    def call(self, inputs):
        hidden_states, attention_mask = inputs
        residual = hidden_states

        hidden_states = self.layer_norm1(hidden_states)
        hidden_states = self.multi_head_attention(
            [hidden_states, attention_mask])
        hidden_states = residual+hidden_states

        residual = hidden_states
        hidden_states = self.layer_norm2(hidden_states)

        hidden_states = self.dense1(hidden_states)
        hidden_states = hidden_states * \
            tf.sigmoid(hidden_states * 1.702)  # qicke_gelu
        hidden_states = self.dense2(hidden_states)

        return residual+hidden_states


class TextEncoder(keras.layers.Layer):
    def __init__(self, encoder_block_num=12, embedding_dim=768):
        super().__init__()
        self.layers = [TextEncoderBlock(embedding_dim)
                       for i in range(encoder_block_num)]

    def call(self, inputs):
        [hidden_states, causal_attention_mask] = inputs
        for layer in self.layers:
            hidden_states = layer([hidden_states, causal_attention_mask])
        return hidden_states


class TextTransformerModel(keras.models.Model):
    """
        使用EmbeddingBock得到融合位置信息的词向量
        使用TextEncoder得到文本编码
        输入为单词索引与位置索引

    Args:
        keras (_type_): _description_
    """

    def __init__(self, encoder_block_num=12, voc_size=49408, max_text_size=77, embedding_dim=768):
        super().__init__()
        self.embeddings = TextEmbeddingBlock(
            voc_size=voc_size, max_text_size=max_text_size, embedding_dim=embedding_dim)
        self.encoder = TextEncoder(encoder_block_num,embedding_dim=embedding_dim)
        self.final_layer_norm = keras.layers.LayerNormalization(epsilon=1e-5)
        self.causal_attention_mask = tf.constant(
            np.triu(np.ones((1, 1, 77, 77), dtype="float32") * -np.inf, k=1)
        )
        # self.causal_attention_mask = tf.linalg.band_part(tf.ones((1, 1, 77, 77))*-tf.inf, -1, 0)

    def call(self, inputs):
        input_ids, position_ids = inputs
        x = self.embeddings([input_ids, position_ids])
        x = self.encoder([x, self.causal_attention_mask])
        return self.final_layer_norm(x)
    
    @staticmethod
    def get_model(max_text_len=77) -> keras.models.Model:
        input_word_ids = keras.layers.Input(
            shape=(max_text_len,), dtype="int32")
        input_pos_ids = keras.layers.Input(
            shape=(max_text_len,), dtype="int32")
        embeds = TextTransformerModel()([input_word_ids, input_pos_ids])
        text_encoder = keras.models.Model(
            [input_word_ids, input_pos_ids], embeds)
        return text_encoder

def test():
    model = TextTransformerModel.get_model()
    model.summary()
test()

```




#### ImageEncoder tensorflow 实现

将图像映射到隐空间下，使生成更加稳定

```python
from email.mime import image
import tensorflow as tf
import tensorflow_addons as tfa
from tensorflow import keras


class AttentionBlock(keras.layers.Layer):
    """
        采用一乘一卷积的注意力机制,每个位置会映射出filters个查询与结果
        输入输出形状相同，有多少个通道，设置多少个注意力头

    Args:
        filters (int): 相当于注意力头数，结果通道数

    """
    # def __init__(self, filters):

    #     super().__init__()
    #     self.norm = tfa.layers.GroupNormalization(epsilon=1e-5)
    #     self.q = PaddedConv2D(filters=filters,kernel_size=1)
    #     self.k = PaddedConv2D(filters=filters,kernel_size=1)
    #     self.v = PaddedConv2D(filters=filters,kernel_size=1)
    #     self.proj_out = PaddedConv2D(filters=filters,kernel_size=1)
    def __init__(self, filters):
        super().__init__()
        self.norm = tfa.layers.GroupNormalization(epsilon=1e-5)
        self.q = keras.layers.Conv2D(filters=filters, kernel_size=1)
        self.k = keras.layers.Conv2D(filters=filters, kernel_size=1)
        self.v = keras.layers.Conv2D(filters=filters, kernel_size=1)
        self.proj_out = keras.layers.Conv2D(filters=filters, kernel_size=1)

    def call(self, inputs):
        a = self.norm(inputs)
        q, k, v = self.q(a), self.k(a), self.v(a)

        batch, height, width, channels = q.shape
        # (h*w,c) h*w个位置的查询向量
        q = tf.reshape(q, (-1, height * width, channels))
        k = keras.layers.Permute((3, 1, 2))(k)  # (c,h,w)
        k = tf.reshape(k, (-1, channels, height * width))  # (c,h*w)
        # (h*w,h*w) w[i][j]=q[i][...] dot k[...][j]  行向量有意义  w[i][...]为第i个位置对其他位置查询的权重向量，
        w = q @ k
        w = w * (channels ** (-0.5))
        w = keras.activations.softmax(w)

        v = keras.layers.Permute((3, 1, 2))(v)  # (c,h,w)
        # (c,h*w) v[...][j]为第j个位置的价值向量
        v = tf.reshape(v, (-1, channels, height * width))
        w = keras.layers.Permute((2, 1))(w)  # w[...][j]为第j个位置对其他位置的查询权重
        # (c,h*w) a[i]为第i个注意力头的结果，a[i][j]=v[i][...] dot w[...][j] 第i个注意力头的价值和对应位置的权重相乘求和
        a = v @ w
        a = keras.layers.Permute((2, 1))(a)  # (h*w,c)
        a = tf.reshape(a, (-1, height, width, channels))  # (h,w,c)展开
        return inputs + self.proj_out(a)


class ResConvBlock(keras.layers.Layer):
    """
        swish加工，same padding卷积两次，如果输入输出通道不同，则会用卷积同步输入通道为输出通道，构建残差连接

    """

    def __init__(self, in_channels, out_channels):
        super().__init__()
        self.norm1 = tfa.layers.GroupNormalization(epsilon=1e-5)
        self.conv1 = keras.layers.Conv2D(
            filters=out_channels, kernel_size=3, padding="same")
        self.norm2 = tfa.layers.GroupNormalization(epsilon=1e-5)
        self.conv2 = keras.layers.Conv2D(
            filters=out_channels, kernel_size=3, padding="same")
        self.nin_shortcut = (
            keras.layers.Conv2D(filters=out_channels,
                                kernel_size=3, padding="same")
            if in_channels != out_channels
            else lambda x: x
        )

    def call(self, inputs):
        h = self.conv1(keras.activations.swish(self.norm1(inputs)))
        h = self.conv2(keras.activations.swish(self.norm2(h)))
        return self.nin_shortcut(inputs) + h


class ImageEncoder(keras.Sequential):
    """
        使用ResConvBlock进行特征提取，再通过步长为2的卷积减小图像尺寸，通道从128增加到512
        图像尺寸缩小为原来的1/8
        在512通道下使用注意力机制提取特征
        使用swish加工输出，然后再用卷积降到8个通道
        
    """
    def __init__(self):
        super().__init__(
            [
                # 输入为h,w,c
                keras.layers.Conv2D(
                    filters=128, kernel_size=3, padding="same"),
                ResConvBlock(in_channels=128, out_channels=128),
                ResConvBlock(in_channels=128, out_channels=128),
                keras.layers.ZeroPadding2D(padding=((0, 1), (0, 1))),
                keras.layers.Conv2D(filters=128, kernel_size=3, strides=2),
                # h/2,w/2,128

                ResConvBlock(in_channels=128, out_channels=256),
                ResConvBlock(in_channels=256, out_channels=256),
                keras.layers.ZeroPadding2D(padding=((0, 1), (0, 1))),
                keras.layers.Conv2D(filters=256, kernel_size=3, strides=2),
                # h/4,w/4,256

                ResConvBlock(in_channels=256, out_channels=512),
                ResConvBlock(in_channels=512, out_channels=512),
                keras.layers.ZeroPadding2D(padding=((0, 1), (0, 1))),
                keras.layers.Conv2D(filters=512, kernel_size=3, strides=2),
                # h/8,w/8,512

                ResConvBlock(in_channels=512, out_channels=512),
                ResConvBlock(in_channels=512, out_channels=512),

                ResConvBlock(in_channels=512, out_channels=512),
                AttentionBlock(filters=512),
                ResConvBlock(in_channels=512, out_channels=512),

                tfa.layers.GroupNormalization(epsilon=1e-5),
                keras.layers.Activation("swish"),
                keras.layers.Conv2D(filters=8, kernel_size=3, padding="same"),
                keras.layers.Conv2D(filters=8, kernel_size=1),
                keras.layers.Lambda(lambda x: x[..., :4] * 0.18215)
            ]
        )
    
    @staticmethod
    def get_models(img_height,img_width):
        img_latent_height = img_height // 8
        img_latent_width = img_width // 8
        inp_img = keras.layers.Input((img_height, img_width, 3))
        image_encoder = ImageEncoder()
        image_encoder = keras.models.Model(inp_img, image_encoder(inp_img))
        
        latent = keras.layers.Input((img_latent_height, img_latent_width, 4))
        image_decoder = ImageDecoder()
        image_decoder = keras.models.Model(latent, image_decoder(latent))
        return image_encoder,image_decoder


class ImageDecoder(keras.Sequential):
    """
        输入为8个通道，先一乘一卷积压缩到4个通道，扩展到512通道，再512通道下使用注意力编码，残差卷积，上采样扩大图像尺寸
        使用残差卷积减小通道数量，512到256到128，使用swish加工数据，卷积到三个通道

    Args:
        keras ([type]): [description]
    """

    def __init__(self):
        super().__init__(

            [
                keras.layers.Lambda(lambda x: 1 / 0.18215 * x),
                keras.layers.Conv2D(filters=4, kernel_size=1),
                keras.layers.Conv2D(
                    filters=512, kernel_size=3, padding="same"),
                ResConvBlock(512, 512),
                AttentionBlock(512),
                ResConvBlock(512, 512),
                ResConvBlock(512, 512),
                ResConvBlock(512, 512),
                ResConvBlock(512, 512),
                keras.layers.UpSampling2D(size=(2, 2)),
                keras.layers.Conv2D(
                    filters=512, kernel_size=3, padding="same"),
                ResConvBlock(512, 512),
                ResConvBlock(512, 512),
                ResConvBlock(512, 512),
                keras.layers.UpSampling2D(size=(2, 2)),
                keras.layers.Conv2D(filters=512, kernel_size=3, padding="same"),
                ResConvBlock(512, 256),
                ResConvBlock(256, 256),
                ResConvBlock(256, 256),
                keras.layers.UpSampling2D(size=(2, 2)),
                keras.layers.Conv2D(
                    filters=256, kernel_size=3, padding="same"),
                ResConvBlock(256, 128),
                ResConvBlock(128, 128),
                ResConvBlock(128, 128),
                tfa.layers.GroupNormalization(epsilon=1e-5),
                keras.layers.Activation("swish"),
                keras.layers.Conv2D(filters=3, kernel_size=3, padding="same"),
            ]
        )
 

def test():
    image_encoder,image_decoder=ImageEncoder.get_models(1000,1000)
    image_encoder.summary()
    image_decoder.summary()
test()
```



#### diffusion_model tensorlow实现

```python
import tensorflow as tf
from tensorflow import keras
import tensorflow_addons as tfa
from layers import apply_seq,GEGLU


class ResBlock(keras.layers.Layer):
    """
        融合时间编码的残差卷积
        输入为图像和时间编码，输出为图像和时间编码融合的结果，尺寸不变
        图像使用卷积映射到给定通道数上
        时间编码使用Dense映射到给定的通道数上
        使用卷积变换到给定通道数上，使用Dense编码为每个通道编码，并广播到每个位置上，并附加残差连接
    """

    def __init__(self, channels, out_channels, trainable=True, name=None, dtype=None, dynamic=False, **kwargs):
        super().__init__(trainable=trainable, name=name,
                         dtype=dtype, dynamic=dynamic, **kwargs)
        self.in_layers = [
            tfa.layers.GroupNormalization(epsilon=1e-5),
            keras.activations.swish,
            keras.layers.Conv2D(filters=out_channels,
                                kernel_size=3, padding="same")
        ]
        self.emb_layers = [
            keras.activations.swish,
            keras.layers.Dense(units=out_channels)
        ]
        self.out_layers = [
            tfa.layers.GroupNormalization(epsilon=1e-5),
            keras.activations.swish,
            keras.layers.Conv2D(filters=out_channels,
                                kernel_size=3, padding="same")
        ]
        self.skip_connection = (
            keras.layers.Conv2D(
                filters=out_channels, kernel_size=1) if channels != out_channels else lambda x: x
        )

    def call(self, inputs):
        x, emb = inputs
        h = apply_seq(x, self.in_layers)  # (b,h,w,out_channels)
        emb_out = apply_seq(emb, self.emb_layers)  # (b,out_channels)
        h = h + emb_out[:, None, None]
        h = apply_seq(h, self.out_layers)
        ret = self.skip_connection(x) + h
        return ret


class CrossAttention(keras.layers.Layer):
    """
    输入为(h*w,n_heads * d_head)
    输出为
    使用Dense映射到n_heads * d_head维度内
    可以附带context，计算kv，实现的注意力机制，若不附带，则为自注意力机制
    通过reshape，每个头的注意力值将会被压缩到一个轴内
    """

    def __init__(self, n_heads, d_head):
        super().__init__()
        self.to_q = keras.layers.Dense(n_heads * d_head, use_bias=False)
        self.to_k = keras.layers.Dense(n_heads * d_head, use_bias=False)
        self.to_v = keras.layers.Dense(n_heads * d_head, use_bias=False)
        self.scale = d_head ** -0.5
        self.num_heads = n_heads
        self.head_size = d_head
        self.to_out = keras.layers.Dense(n_heads * d_head)

    def call(self, inputs):
        assert type(inputs) is list
        if len(inputs) == 1:
            inputs = inputs + [None]
        x, context = inputs
        context = x if context is None else context
        q, k, v = self.to_q(x), self.to_k(context), self.to_v(context)
        assert len(x.shape) == 3
        q = tf.reshape(q, (-1, x.shape[1], self.num_heads, self.head_size))
        k = tf.reshape(
            k, (-1, context.shape[1], self.num_heads, self.head_size))
        v = tf.reshape(
            v, (-1, context.shape[1], self.num_heads, self.head_size))

        # (bs, num_heads, time, head_size)
        q = keras.layers.Permute((2, 1, 3))(q)
        # (bs, num_heads, head_size, time)
        k = keras.layers.Permute((2, 3, 1))(k)
        # (bs, num_heads, time, head_size)
        v = keras.layers.Permute((2, 1, 3))(v)

        # score = td_dot(q, k) * self.scale
        score = q @ k * self.scale  # (bs,num_heads,time,time)
        # (bs, num_heads, time, time) 第i行 为其他位置对i位置回应的权重
        weights = keras.activations.softmax(score)
        # attention = td_dot(weights, v)
        attention = weights @ v  # (bs,num_heads,time,head_size)
        attention = keras.layers.Permute((2, 1, 3))(
            attention
        )  # (bs, time, num_heads, head_size)
        h_ = tf.reshape(
            attention, (-1, x.shape[1], self.num_heads * self.head_size))
        return self.to_out(h_)


class BasicTransformerBlock(keras.layers.Layer):
    """
        输入包含输入，上下文两个部分
        首先使用自注意力编码，然后再使用包含上下文的交叉注意力编码，最终使用dense层映射与给定的dim维度内
    """

    def __init__(self, dim, n_heads, d_head):
        super().__init__()
        self.norm1 = keras.layers.LayerNormalization(epsilon=1e-5)
        self.attn1 = CrossAttention(n_heads, d_head)

        self.norm2 = keras.layers.LayerNormalization(epsilon=1e-5)
        self.attn2 = CrossAttention(n_heads, d_head)

        self.norm3 = keras.layers.LayerNormalization(epsilon=1e-5)
        self.geglu = GEGLU(dim * 4)
        self.dense = keras.layers.Dense(dim)

    def call(self, inputs):
        x, context = inputs
        x = self.attn1([self.norm1(x)]) + x
        x = self.attn2([self.norm2(x), context]) + x
        return self.dense(self.geglu(self.norm3(x))) + x


class SpatialTransformer(keras.layers.Layer):
    """
        输入为图像，与上下文信息 (b,h,w,c,2)

        使用注意力机制编码，使用1*1卷积将图像映射到到注意力所需要的通道数上n_heads * d_head
        然后reshape到二维(h*w,n_heads * d_head)传入BasicTransformerBlock

        输出为(h,w,channels)
    """

    def __init__(self, channels, n_heads, d_head):
        super().__init__()
        self.norm = tfa.layers.GroupNormalization(epsilon=1e-5)
        assert channels == n_heads * d_head
        self.proj_in = keras.layers.Conv2D(
            filters=n_heads * d_head, kernel_size=1)
        self.transformer_blocks = [BasicTransformerBlock(
            dim=channels, n_heads=n_heads, d_head=d_head)]
        self.proj_out = keras.layers.Conv2D(filters=channels, kernel_size=1)

    def call(self, inputs):
        x, context = inputs
        b, h, w, c = x.shape
        x_in = x
        x = self.norm(x)
        x = self.proj_in(x)  # b,h,w,n_heads * d_head
        x = tf.reshape(x, (-1, h * w, c))
        for block in self.transformer_blocks:
            x = block([x, context])
        x = tf.reshape(x, (-1, h, w, c))
        return self.proj_out(x) + x_in


class Downsample(keras.layers.Layer):
    """
        下采样为原来的二分之一，并通过卷积映射到给定通道数内
    """

    def __init__(self, channels):
        super().__init__()
        self.padding = keras.layers.ZeroPadding2D(padding=(1, 1))
        self.conv2d = keras.layers.Conv2D(
            filters=channels, kernel_size=3, strides=2)

    def call(self, x):
        x = self.padding(x)
        x = self.conv2d(x)
        return x


class Upsample(keras.layers.Layer):
    """
    上采样为原先尺寸的两倍，并通过卷积映射到给定通道内
    """

    def __init__(self, channels):
        super().__init__()
        self.ups = keras.layers.UpSampling2D(size=(2, 2))
        self.conv = keras.layers.Conv2D(
            filters=channels, kernel_size=3, padding="same")

    def call(self, x):
        x = self.ups(x)
        return self.conv(x)


class DiffusionModel(keras.models.Model):
    """
        输入包含图像，时间编码，上下文
        time_embed进一步编码时间，使用两个1280units的Dense层，swish
        时间编码使用两个1280个units的Dense层，使用ResBlock会附加时间编码信息，SpatialTransformer则是附加上下文信息

        encoder部分
        输入刚开始通过卷积映射到320通道中
        使用DownSample缩小图像尺寸，ResBlock增加通道，直到1280通道

        decoder部分
        concat 对应encoder层的输出，需要先使用ResBlock恢复通道数，再使用UpSample扩大图像尺寸

    Args:
        keras (_type_): _description_
    """

    def __init__(self):
        super().__init__()
        self.time_embed = [
            keras.layers.Dense(units=1280),
            keras.activations.swish,
            keras.layers.Dense(units=1280),
        ]
        self.input_blocks = [
            [keras.layers.Conv2D(filters=320, kernel_size=3,
                                 padding="same")],  # (h,w,320)
            [ResBlock(channels=320, out_channels=320), SpatialTransformer(
                channels=320, n_heads=8, d_head=40)],
            [ResBlock(channels=320, out_channels=320), SpatialTransformer(
                channels=320, n_heads=8, d_head=40)],
            [Downsample(channels=320)],  # (h/2,w/2.320)
            [ResBlock(channels=320, out_channels=640), SpatialTransformer(
                channels=640, n_heads=8, d_head=80)],
            [ResBlock(channels=640, out_channels=640), SpatialTransformer(
                channels=640, n_heads=8, d_head=80)],
            [Downsample(channels=640)],  # (h/4,w/2.640)
            [ResBlock(channels=640, out_channels=1280), SpatialTransformer(
                channels=1280, n_heads=8, d_head=160)],
            [ResBlock(channels=1280, out_channels=1280), SpatialTransformer(
                channels=1280, n_heads=8, d_head=160)],
            [Downsample(channels=1280)],  # (h/4,w/2.1280)
            [ResBlock(channels=1280, out_channels=1280)],
            [ResBlock(channels=1280, out_channels=1280)],
        ]
        self.middle_block = [
            ResBlock(channels=1280, out_channels=1280),
            SpatialTransformer(channels=1280, n_heads=8, d_head=160),
            ResBlock(channels=1280, out_channels=1280),
        ]
        self.output_blocks = [
            # [ResBlock(channels=1280, out_channels=1280)]
            [ResBlock(channels=2560, out_channels=1280)],
            # [ResBlock(channels=1280, out_channels=1280)]
            [ResBlock(channels=2560, out_channels=1280)],
            [ResBlock(channels=2560, out_channels=1280), Upsample(
                channels=1280)],  # [Downsample(channels=1280)]
            # [ResBlock(channels=1280, out_channels=1280), SpatialTransformer(channels=1280, n_heads=8, d_head=160)]
            [ResBlock(channels=2560, out_channels=1280), SpatialTransformer(
                channels=1280, n_heads=8, d_head=160)],
            # [ResBlock(channels=640, out_channels=1280), SpatialTransformer(channels=1280, n_heads=8, d_head=160)]
            [ResBlock(channels=2560, out_channels=1280), SpatialTransformer(
                channels=1280, n_heads=8, d_head=160)],
            [
                ResBlock(channels=1920, out_channels=1280),
                SpatialTransformer(channels=1280, n_heads=8, d_head=160),
                Upsample(channels=1280),
            ],  # [Downsample(channels=640)]  (h/4,w/4,640)
            # [ResBlock(channels=640, out_channels=640), SpatialTransformer(channels=640, n_heads=8, d_head=80)]
            [ResBlock(channels=1920, out_channels=640), SpatialTransformer(
                channels=640, n_heads=8, d_head=80)],
            # [ResBlock(channels=320, out_channels=640), SpatialTransformer(channels=640, n_heads=8, d_head=80)],
            [ResBlock(channels=1280, out_channels=640), SpatialTransformer(
                channels=640, n_heads=8, d_head=80)],
            [
                ResBlock(channels=960, out_channels=640),
                SpatialTransformer(channels=640, n_heads=8, d_head=80),
                Upsample(channels=640),
            ],  # [Downsample(channels=320)]
            # [ResBlock(channels=320, out_channels=320), SpatialTransformer(channels=320, n_heads=8, d_head=40)]
            [ResBlock(channels=960, out_channels=320), SpatialTransformer(
                channels=320, n_heads=8, d_head=40)],
            # [ResBlock(channels=320, out_channels=320), SpatialTransformer(channels=320, n_heads=8, d_head=40)]
            [ResBlock(channels=640, out_channels=320), SpatialTransformer(
                channels=320, n_heads=8, d_head=40)],
            # [keras.layers.Conv2D(filters=320, kernel_size=3, padding="same")]
            [ResBlock(channels=640, out_channels=320), SpatialTransformer(
                channels=320, n_heads=8, d_head=40)],
        ]
        self.out = [
            tfa.layers.GroupNormalization(epsilon=1e-5),
            keras.activations.swish,
            keras.layers.Conv2D(filters=4, kernel_size=3, padding="same"),
        ]

    def call(self, inputs):
        x, t_emb, context = inputs
        emb = apply_seq(t_emb, self.time_embed)

        def apply(x, layer):
            if isinstance(layer, ResBlock):
                x = layer([x, emb])
            elif isinstance(layer, SpatialTransformer):
                x = layer([x, context])
            else:
                x = layer(x)
            return x

        saved_inputs = []
        for b in self.input_blocks:
            for layer in b:
                x = apply(x, layer)
            saved_inputs.append(x)

        for layer in self.middle_block:
            x = apply(x, layer)

        for b in self.output_blocks:
            x = tf.concat([x, saved_inputs.pop()], axis=-1)
            for layer in b:
                x = apply(x, layer)
        return apply_seq(x, self.out)
    
    @staticmethod
    def get_model(image_height,image_width,max_text_len=77)->keras.models.Model:
        context = keras.layers.Input((max_text_len, 768))
        t_emb = keras.layers.Input((320,))
        latent = keras.layers.Input((image_height, image_width, 4))
        diffusion_model = DiffusionModel()
        diffusion_model = keras.models.Model(
            [latent, t_emb, context], diffusion_model([latent, t_emb, context])
        )
        return diffusion_model
```

#### Stable Diffusion 实现

```python
import numpy as np
from tqdm import tqdm
import math

import tensorflow as tf
from tensorflow import keras

from image_encoder import ImageDecoder, ImageEncoder
from diffusion_model import DiffusionModel
from text_transformer import TextTransformerModel
from clip_tokenizer import SimpleTokenizer
from constants import _UNCONDITIONAL_TOKENS, _ALPHAS_CUMPROD, PYTORCH_CKPT_MAPPING
from PIL import Image

MAX_TEXT_LEN = 77


class StableDiffusion:
    def __init__(self, img_height=1000, img_width=1000,max_text_len=77, jit_compile=False, download_weights=True):
        self.img_height = img_height
        self.img_width = img_width
        self.img_latent_height = img_height // 8
        self.img_latent_width=img_width//8
        
        self.tokenizer = SimpleTokenizer()
        
        self.text_encoder = TextTransformerModel.get_model(max_text_len)
        self.diffusion_model = DiffusionModel.get_model(self.img_latent_height, self.img_latent_width,max_text_len)
        self.image_encoder,self.image_decoder = ImageEncoder.get_models(self.img_height, self.img_width)
        if(download_weights):
            text_encoder_weights_fpath = keras.utils.get_file(
            origin="https://huggingface.co/fchollet/stable-diffusion/resolve/main/text_encoder.h5",
            file_hash="d7805118aeb156fc1d39e38a9a082b05501e2af8c8fbdc1753c9cb85212d6619",
        )
        diffusion_model_weights_fpath = keras.utils.get_file(
            origin="https://huggingface.co/fchollet/stable-diffusion/resolve/main/diffusion_model.h5",
            file_hash="a5b2eea58365b18b40caee689a2e5d00f4c31dbcb4e1d58a9cf1071f55bbbd3a",
        )
        decoder_weights_fpath = keras.utils.get_file(
            origin="https://huggingface.co/fchollet/stable-diffusion/resolve/main/decoder.h5",
            file_hash="6d3c5ba91d5cc2b134da881aaa157b2d2adc648e5625560e3ed199561d0e39d5",
        )

        encoder_weights_fpath = keras.utils.get_file(
            origin="https://huggingface.co/divamgupta/stable-diffusion-tensorflow/resolve/main/encoder_newW.h5",
            file_hash="56a2578423c640746c5e90c0a789b9b11481f47497f817e65b44a1a5538af754",
        )

        self.text_encoder.load_weights(text_encoder_weights_fpath)
        self.diffusion_model.load_weights(diffusion_model_weights_fpath)
        self.image_encoder.load_weights(encoder_weights_fpath)
        self.image_decoder.load_weights(decoder_weights_fpath)
        

        if jit_compile:
            self.text_encoder.compile(jit_compile=True)
            self.diffusion_model.compile(jit_compile=True)
            self.image_decoder.compile(jit_compile=True)
            self.image_encoder.compile(jit_compile=True)

        self.dtype = tf.float32
        if tf.keras.mixed_precision.global_policy().name == 'mixed_float16':
            self.dtype = tf.float16

    def generate(
        self,
        prompt,
        negative_prompt=None,
        batch_size=1,
        num_steps=25,
        unconditional_guidance_scale=7.5,
        temperature=1,
        seed=None,
        input_image=None,
        input_mask=None,
        input_image_strength=0.5,
    ):
        # Tokenize prompt (i.e. starting context)
        inputs = self.tokenizer.encode(prompt)
        assert len(inputs) < 77, "Prompt is too long (should be < 77 tokens)"
        phrase = inputs + [49407] * (77 - len(inputs))
        phrase = np.array(phrase)[None].astype("int32")
        phrase = np.repeat(phrase, batch_size, axis=0)

        # Encode prompt tokens (and their positions) into a "context vector"
        pos_ids = np.array(list(range(77)))[None].astype("int32")
        pos_ids = np.repeat(pos_ids, batch_size, axis=0)
        context = self.text_encoder.predict_on_batch([phrase, pos_ids])
        
        input_image_tensor = None
        if input_image is not None:
            if type(input_image) is str:
                input_image = Image.open(input_image)
                input_image = input_image.resize((self.img_width, self.img_height))

            elif type(input_image) is np.ndarray:
                input_image = np.resize(input_image, (self.img_height, self.img_width, input_image.shape[2]))
                
            input_image_array = np.array(input_image, dtype=np.float32)[None,...,:3]
            input_image_tensor = tf.cast((input_image_array / 255.0) * 2 - 1, self.dtype)

        if type(input_mask) is str:
            input_mask = Image.open(input_mask)
            input_mask = input_mask.resize((self.img_width, self.img_height))
            input_mask_array = np.array(input_mask, dtype=np.float32)[None,...,None]
            input_mask_array =  input_mask_array / 255.0
            
            latent_mask = input_mask.resize((self.img_width//8, self.img_height//8))
            latent_mask = np.array(latent_mask, dtype=np.float32)[None,...,None]
            latent_mask = 1 - (latent_mask.astype("float") / 255.0)
            latent_mask_tensor = tf.cast(tf.repeat(latent_mask, batch_size , axis=0), self.dtype)


        # Tokenize negative prompt or use default padding tokens
        unconditional_tokens = _UNCONDITIONAL_TOKENS
        if negative_prompt is not None:
            inputs = self.tokenizer.encode(negative_prompt)
            assert len(inputs) < 77, "Negative prompt is too long (should be < 77 tokens)"
            unconditional_tokens = inputs + [49407] * (77 - len(inputs))

        # Encode unconditional tokens (and their positions into an
        # "unconditional context vector"
        unconditional_tokens = np.array(unconditional_tokens)[None].astype("int32")
        unconditional_tokens = np.repeat(unconditional_tokens, batch_size, axis=0)
        unconditional_context = self.text_encoder.predict_on_batch(
            [unconditional_tokens, pos_ids]
        )
        timesteps = np.arange(1, 1000, 1000 // num_steps)
        input_img_noise_t = timesteps[ int(len(timesteps)*input_image_strength) ]
        latent, alphas, alphas_prev = self.get_starting_parameters(
            timesteps, batch_size, seed , input_image=input_image_tensor, input_img_noise_t=input_img_noise_t
        )

        if input_image is not None:
            timesteps = timesteps[: int(len(timesteps)*input_image_strength)]

        # Diffusion stage
        progbar = tqdm(list(enumerate(timesteps))[::-1])
        for index, timestep in progbar:
            progbar.set_description(f"{index:3d} {timestep:3d}")
            e_t = self.get_model_output(
                latent,
                timestep,
                context,
                unconditional_context,
                unconditional_guidance_scale,
                batch_size,
            )
            a_t, a_prev = alphas[index], alphas_prev[index]
            latent, pred_x0 = self.get_x_prev_and_pred_x0(
                latent, e_t, index, a_t, a_prev, temperature, seed
            )

            if input_mask is not None and input_image is not None:
                # If mask is provided, noise at current timestep will be added to input image.
                # The intermediate latent will be merged with input latent.
                latent_orgin, alphas, alphas_prev = self.get_starting_parameters(
                    timesteps, batch_size, seed , input_image=input_image_tensor, input_img_noise_t=timestep
                )
                latent = latent_orgin * latent_mask_tensor + latent * (1- latent_mask_tensor)

        # Decoding stage
        decoded = self.decoder.predict_on_batch(latent)
        decoded = ((decoded + 1) / 2) * 255

        if input_mask is not None:
          # Merge inpainting output with original image
          decoded = input_image_array * (1-input_mask_array) + np.array(decoded) * input_mask_array

        return np.clip(decoded, 0, 255).astype("uint8")

    def timestep_embedding(self, timesteps, dim=320, max_period=10000):
        half = dim // 2
        freqs = np.exp(
            -math.log(max_period) * np.arange(0, half, dtype="float32") / half
        )
        args = np.array(timesteps) * freqs
        embedding = np.concatenate([np.cos(args), np.sin(args)])
        return tf.convert_to_tensor(embedding.reshape(1, -1),dtype=self.dtype)

    def add_noise(self, x , t , noise=None ):
        batch_size,w,h = x.shape[0] , x.shape[1] , x.shape[2]
        if noise is None:
            noise = tf.random.normal((batch_size,w,h,4), dtype=self.dtype)
        sqrt_alpha_prod = _ALPHAS_CUMPROD[t] ** 0.5
        sqrt_one_minus_alpha_prod = (1 - _ALPHAS_CUMPROD[t]) ** 0.5

        return  sqrt_alpha_prod * x + sqrt_one_minus_alpha_prod * noise

    def get_starting_parameters(self, timesteps, batch_size, seed,  input_image=None, input_img_noise_t=None):
        n_h = self.img_height // 8
        n_w = self.img_width // 8
        alphas = [_ALPHAS_CUMPROD[t] for t in timesteps]
        alphas_prev = [1.0] + alphas[:-1]
        if input_image is None:
            latent = tf.random.normal((batch_size, n_h, n_w, 4), seed=seed)
        else:
            latent = self.encoder(input_image)
            latent = tf.repeat(latent , batch_size , axis=0)
            latent = self.add_noise(latent, input_img_noise_t)
        return latent, alphas, alphas_prev

    def get_model_output(
        self,
        latent,
        t,
        context,
        unconditional_context,
        unconditional_guidance_scale,
        batch_size,
    ):
        timesteps = np.array([t])
        t_emb = self.timestep_embedding(timesteps)
        t_emb = np.repeat(t_emb, batch_size, axis=0)
        unconditional_latent = self.diffusion_model.predict_on_batch(
            [latent, t_emb, unconditional_context]
        )
        latent = self.diffusion_model.predict_on_batch([latent, t_emb, context])
        return unconditional_latent + unconditional_guidance_scale * (
            latent - unconditional_latent
        )

    def get_x_prev_and_pred_x0(self, x, e_t, index, a_t, a_prev, temperature, seed):
        sigma_t = 0
        sqrt_one_minus_at = math.sqrt(1 - a_t)
        pred_x0 = (x - sqrt_one_minus_at * e_t) / math.sqrt(a_t)

        # Direction pointing to x_t
        dir_xt = math.sqrt(1.0 - a_prev - sigma_t**2) * e_t
        noise = sigma_t * tf.random.normal(x.shape, seed=seed) * temperature
        x_prev = math.sqrt(a_prev) * pred_x0 + dir_xt
        return x_prev, pred_x0

```

### 参数重整化

希望从高斯分布$N(\mu,\sigma)$中采样，可以先从标注只能分布$N(0,1)$采样出Z，在得到$\sigma z+\mu$，这样做将随机性转移到z这个常量上，而$\sigma,\mu$当做仿射变换网络的一部分

### Jensen不等式

$$
log\sum_j \lambda_jy_j\ge\sum_j\lambda_jlogy_j
$$

其中$\lambda_j \ge 0,\sum_j\lambda_j=1$

## 其他场景的应用

- WaveGrad

  语音合成

- Diffusion-LM

  文本生成，添加噪声到隐变量空间

- Diffusioin via Edit-based Reconstruction(EiffusER)

  文本生成，对文本添加mask噪声

## 类似方法

### Mask-Predict

训练一个图像的AutoEncoder

训练时，把一张图片的token某些部分换成masktoken，训练一个BidrectionalTransformer网络，再把mask预测出来

在生成时，使用全部是mask的token为初始，使用BidrectionalTransformer填充mask，并保留置信度高的填充结果，置信度低的再置为mask投入BidrectionalTransformer