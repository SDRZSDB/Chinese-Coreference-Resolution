background：
此项目为北京理工大学-计算机学院-辛欣-2024知识工程-第一次大作业-指代消解的源码
data：
出于版权限制，不能提供train等文件夹的json数据集，结构可参考图片
abstract：
数据预处理
1.读取：读入txt文本数据，先删去空行，再读取每个句子前20（或19）个字符制成索引。 
2.清洗：删去每个句子开始的编号，再用正则表达式清洗，注意task中指出的先行词和照应语给出了在其句子中的索引位置，而该索引的计算包括标点位置，所以清洗时应保留中文标点符号和全角字符，否则会严重影响准确性。
3.分词：由于task中已给出了位置索引，所以分词必须按照txt给出的标准划分，不能使用jieba，二者分词结果有明显差异。
4.词性提取：指代消解任务的先行词和照应词为名次或代词，故获取出现频率最高的4900个名词和305个代词，作为后续独热编码的属性。
5.独热编码：句子的属性为5205个热词的词频，编码后得到一张大的二维表。
6.json文件处理：读取键值，循环配对”0”、”1”和”pronoun”，处理多个先行词和一个照应词的情况。注意这一阶段需要多次特判，有数十个task文件是空的，还有一个task文件给出的id是不存在的，容易出现bug。
7.系统兼容：win系统源码可以直接运行，macos需要将json文件读取编码设为gb18030。
简化版embedding
1.预处理：判断先行词和照应词是否在同一个句子，二者谁先谁后，并按照不同情况分别处理，获得句子长度、先行词和照应词在整段文本（可能有多个句子）中的位置索引，以供后续使用。
2.独热编码成分：将任务涉及的句子对应的独热编码进行叠加，矩阵形状不变，再乘一个权重，作为特征的第一种成分。
3.距离成分：将先行词和照应词的距离乘一个权重作为第二种成分。由于不能保证文本清洗时保留了所有应保存的标点，在先行词和照应词距离很近时可能出现距离为负值的情况，这种情况特殊处理将距离都设为0。
4.位置成分：模仿bert的position embedding使用正余弦函数处理，由于独热编码缺乏单个词汇的特征向量，故将先行词位置正弦处理，照应词位置余弦处理，实现弱化版的位置嵌入，作为特征的第三种成分。
5.构建数据集：先将正例的特征和label的1存入数据集，然后按上述处理负例，注意先行词和照应词的先后问题，这一点任务要求描述不清晰，本文思路是先行词在前时将文本开始处到照应词之间的词语作为负例（正例位置除外），先行词在后时将照应词到文本末尾之间的词语作为负例（正例位置除外）。
6.正则化：由于数据集里存在一些任务先行词和照应词的位置很远，导致特征的距离成分很大，进而产生特大值，导致后续训练时loss梯度爆炸，故先在此处正则化处理一次。
7.过采样：该任务明显正例远少于负例，为了防止模型只输出0就能得到很好的结果，进行1:1的过采样。
模型构建
1.多层感知机：多层全连接层和激活函数组成，输出前先dropout再按二分类要求sigmoid
2.优化器：Adam优化器，并设置scheduler实现学习率动态衰减
3.损失函数：MSELoss（模型已经过Sigmoid层）
训练
1.GPU训练：将模型、数据传入GPU
2.迭代训练：求输出、损失，进行反向传播、权重更新、学习率更新，长时间不进步就early stopping
3.本地优化：电脑配置内存16G*2，显卡3070Ti（显存8G），配置不高既爆内存也爆显存，
故要在一个batch训练完后及时释放部分已用完数据的显存，batch_size调的小一点，如果还是爆显存可以考虑关闭GPU的并行计算用调试方法运行。
4.评价标准：accurracy，recall，prcision，F1_score，PR图
requirement：
python=3.9.18
torch=2.0.0+cu117
torchvision=0.15.1+cu117
visdom=0.2.4
其余包自配
attention：
文件给出了详细的源码和注释，同届同学请勿抄袭，每届作业不同，后来者可参考
