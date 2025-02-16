
参考代码：
https://www.github.com/tensorflow/tensorflow/blob/r1.4/tensorflow/examples/tutorials/word2vec/word2vec_basic.py


**训练结果**

按照作业提示，设置了400,000个step，生成了dictionary.json(78K)、reversed_dictionary.json(88K)和embedding.npy(2.6M)三个文件，将训练结果降维后的输出图片如下：
图中圈出几处明显同类词的聚集范围：
- 金、银、宝、玉
- 一、二、三、四、五、六、八、九、十、百、千、万、几、半、数
- 小、浅、深、静、轻、微、薄、浓、淡、凝、多
- 言、语、声、闻、听、看、吟
- 独、孤、疏、断、残
- 垂、低、远、高
- 东、南、西、北
- 花、叶、林、树、枝
- 标点、。，）（集中在一起
- UNK字符位置也相对独立

<div align=center><img  src="https://github.com/jianjian123789/CSDN/blob/master/eleven_weeks/eleventh_weeks_homework/csdn_quiz_week11_output_word2vec.png"/></div>

**心得体会**

代码理解：
1. 准备数据：案例中和录播课程中使用的是从网上下载的text8.zip中的全英文单词的文本，作业中直接读取了QuanSongCi.txt里的中文文本。
2. 建立word与index的对应dictonary和reverse_dictionary：首先对word序列使用collections.Counter().most_common()函数进行了排序，取出频次最高的前4999个word和count对来填充字典，然后遍历所有word，将未出现在前4999的word计数并作为UNK字符加到字典中。
3. 构建生成训练data和lable的函数，关键是理解滑窗的逻辑。其中训练数据和label均是以word的index表示的。
4. 构建训练的计算图，将5000个不同word塞入到128维空间。重点理解skip-gram和cbow的区别、embedding_lookup原理、把多分类问题转为二分类优化计算的nce_loss、利用余弦距离计算相识度。
5. 使用了作业建议的400,000个step，最后输出训练结果embedding文件
6. 利用pca降维，使用matplotlib绘制图形化结果。

Word2Vec理解：
1. Word2Vec是Word Embedding的一种实现，将word表示由高维稀疏转为低维稠密(相对高维而言)。
2. OneHot Representation表达效率太低，词间关联没有体现，在使用中计算量过大。Word Embedding是Distributed Representaion，很好的解决了这方面的问题，尤其Word2Vec同类词的"对应关系"，可以用同向同大小的向量表示。
3. cbow和skip-gram都是在word2vec中用于将文本进行向量表示的实现方法。在cbow方法中，是用周围词预测中心词，从而利用中心词的预测结果情况，使用GradientDesent方法，不断的去调整周围词的向量。当训练完成之后，每个词都会作为中心词，把周围词的词向量进行了调整，这样也就获得了整个文本里面所有词的词向量。skip-gram是用中心词来预测周围的词。在skip-gram中，会利用周围的词的预测结果情况，使用GradientDecent来不断的调整中心词的词向量，最终所有的文本遍历完毕之后，也就得到了文本所有词的词向量。
4. Word2Vec的语言模型有两种对传统的神经网络改进的训练方法：一种是基于Hierarchical Softmax的（主要是时霍夫曼树，以前用到时主要是做压缩数据使用），另一种是基于Negative Sampling的。

## 作业 2.RNN训练
**数据集:**

没有再从新生成训练数据，而是直接使用作业1中生成的匹配的dictionary.json、reversed_dictionary.json和embedding.npy作为本部分的数据。

**代码**

其中utils.py的修改内容如下：
```python
def get_train_data(vocabulary, batch_size, num_steps):
    ################# My code here ###################
    # 有时不能整除，需要截掉一些字
    data_partition_size = len(vocabulary) // batch_size
    word_valid_count = batch_size * data_partition_size
    vocabulary_valid = np.array(vocabulary[: word_valid_count])
    word_x = vocabulary_valid.reshape([batch_size, data_partition_size])

    # 随机一个起始位置
    start_idx = random.randint(0, 16)
    while True:
        # 因为训练中要使用的是字和label(下一个字)的index，
        # 但这里没有dictionary，无法得到index
        # 所以将每个time step返回的训练数据长度是num_steps+1
        #     其中前num_steps个字用于训练(训练时转化为index)
        #     从第2个字起的num_steps个字用于训练的label(训练时转化为index)
        if start_idx + num_steps + 1 > word_valid_count:
            break

        yield word_x[:, start_idx: start_idx + num_steps + 1]
        start_idx += num_steps
    ##################################################
```

model.py的修改内容如下：
```python
with tf.variable_scope('rnn'):
    ################# My code here ###################
    cells = [tf.nn.rnn_cell.DropoutWrapper(
                          tf.nn.rnn_cell.BasicLSTMCell(self.dim_embedding),
                          output_keep_prob=self.keep_prob
                          ) for i in range(self.rnn_layers)]

    rnn_multi = tf.nn.rnn_cell.MultiRNNCell(cells, state_is_tuple=True)

    self.state_tensor = rnn_multi.zero_state(self.batch_size, tf.float32)

    outputs_tensor, self.outputs_state_tensor = tf.nn.dynamic_rnn(
        rnn_multi, data, initial_state = self.state_tensor, dtype=tf.float32)

    tf.summary.histogram('outputs_state_tensor', self.outputs_state_tensor)

seq_output = tf.concat(outputs_tensor, 1)
    ##################################################

# flatten it
seq_output_final = tf.reshape(seq_output, [-1, self.dim_embedding])

with tf.variable_scope('softmax'):
    ################# My code here ###################
    softmax_w = tf.Variable(tf.truncated_normal(
                [self.dim_embedding, self.num_words], stddev=0.1))
    softmax_b = tf.Variable(tf.zeros(self.num_words))

    tf.summary.histogram('softmax_w', softmax_w)
    tf.summary.histogram('softmax_b', softmax_b)

logits = tf.reshape(tf.matmul(seq_output_final, softmax_w) + softmax_b,
                  [self.batch_size, -1, self.num_words])
    ##################################################

tf.summary.histogram('logits', logits)

self.predictions = tf.nn.softmax(logits, name='predictions')

#print('logits shape:', logits.get_shape())
#print('labels shape:', self.Y.get_shape())
loss = tf.nn.sparse_softmax_cross_entropy_with_logits(
    logits=logits, labels=self.Y)
self.loss = tf.reduce_mean(loss)
tf.summary.scalar('logits_loss', self.loss)
```

train.py的修改内容如下：
```python
for dl in utils.get_train_data(vocabulary,
                  batch_size=FLAGS.batch_size, num_steps=FLAGS.num_steps):
    ################# My code here ###################
    # dl(data+label), dl.shape[0]是batch_size, shape[1]是num_steps+1，
    # 内容都是字，需要转为index
    # 注意：其中前num_steps个字用于训练(训练时转化为index)
    #      从第2个字起的num_steps个字用于训练的label(训练时转化为index)
    dl_word_len = dl.shape[1]
    dl_index = utils.index_data(dl, dictionary)  # 找到每一个字对应的index
    d_index = dl_index[:, :dl_word_len - 1]
    l_index = dl_index[:, 1:]
    feed_dict = {model.X: d_index, model.Y: l_index,
                  model.state_tensor: state, model.keep_prob: 0.5}
    ##################################################

    gs, _, state, l, summary_string = sess.run(
        [model.global_step, model.optimizer, model.outputs_state_tensor,
         model.loss, model.merged_summary_op], feed_dict=feed_dict)
    summary_string_writer.add_summary(summary_string, gs)
```



**训练结果**

训练的log中没有出现作业提示的文字转码问题，正常输出了文字
2018-12-07 03:05:09,705 - DEBUG - sample.py:46 - restore from [/output/model.ckpt-594600]
2018-12-07 03:05:10,146 - DEBUG - sample.py:82 - ==============[江神子]==============
2018-12-07 03:05:10,146 - DEBUG - sample.py:83 - 江神子

刘UNK

水调歌头（寿刘守）

一笑一年日，一笑一杯中。一杯一笑，一年一笑一杯中。一笑一杯一笑，不似一杯一笑，不用有人
2018-12-07 03:05:10,404 - DEBUG - sample.py:82 - ==============[蝶恋花]==============
2018-12-07 03:05:10,405 - DEBUG - sample.py:83 - 蝶恋花一笑一枝。
刘UNK

水调歌头（寿刘守）

一笑一年日，一笑一杯中。一杯一笑，一年一笑一杯中。一笑一杯一笑，不似一杯一笑
2018-12-07 03:05:10,669 - DEBUG - sample.py:82 - ==============[渔家傲]==============
2018-12-07 03:05:10,669 - DEBUG - sample.py:83 - 渔家傲
刘UNK

水调歌头（寿刘守）

一笑一年日，一笑一杯中。一杯一笑，一年一笑一杯中。一笑一杯一笑，不似一杯一笑，不用有人

**心得体会**

代码理解：
1. 梯度剪裁操作：tf.clip_by_global_norm(tf.gradients(loss, tvars), 5)，将梯度强行控制在(-5,5)区间，解决可能出现的梯度爆炸问题。
2. 录播课程代码讲解和作业中比较需要细心的地方是训练数据和lable的构造，都是以word的一下个word作为label。这样的效果就是在验证时，取一个word就会得到下一个word,类似背诵文章。

RNN和LSTM理解：
1. 相对于CNN，主要区别是：即时输入，存储之前的输入信息；
2. 结构简单，每个单元只有一个隐层，信息抽取能力弱，但增加了时间轴，在时间轴上形成了深度网络，每层共享权重。实际在处理文本时，是在输入数据上进行滑动，保证时序的连续性，来模拟即时输入。而CNN中划分数据时较为随机。
3. 反向传播包括layer间的传递和time_step间的传递，不是每处理一个输入数据就反向传播一次，而是经过几个time_step之后，统一计算loss之后，进行一次反向传播操作。
4. tensorflow的static_rnn实现时，会生成整个的计算图，且对输入的维度timesteps有固定要求；dynamic_rnn实现时，不真正生成整个计算图，而是在计算单元cell外包裹一个loop，对timesteps长度无固定要求，比较符合实际自然语言情况。
5. LSTM中增加了三个gate，分别控制要遗忘多少原来记住的信息、要记住多少当前输入的信息、要将多少信息输出到下一时刻。当然LSTM的“权重”涉及到的参数数量也增长了，变为普通rnn的4倍左右。

