# OGeek  
A competition about search results CTR prediction in real-time search scenario ! Rank : 26 / 2888 !   
[OGeek算法挑战赛](https://tianchi.aliyun.com/competition/entrance/231688/introduction?spm=5176.12281957.0.0.38b04c2azpDhUa)是天池平台上的一个大赛，主要内容是基于百万最新真实用户搜索数据的实时搜索场景下搜索结果ctr预估。给定用户输入prefix（用户输入，查询词前缀）以及文章标题、文章类型等数据，预测用户是否点击，评价标准采用F1 score 指标。  
最终排名：初赛：17 / 2888；复赛：26 / 2888  
附：[github地址](https://github.com/RHKeng/OGeek)  
  
* 1 **比赛简介**  
在搜索业务下有一个场景叫实时搜索（Instance Search）,就是在用户不断输入过程中，实时返回查询结果。此次赛题来自OPPO手机搜索排序优化的一个子场景，并做了相应的简化，意在解决query-title语义匹配的问题。简化后，本次题目内容主要为一个实时搜索场景下query-title的ctr预估问题。给定用户输入prefix（用户输入，查询词前缀）以及文章标题、文章类型等数据，预测用户是否点击，评价标准采用F1 score 指标。  
  
* 2 **问题分析**  
通过分析问题，我们尝试将本次比赛分解成两个子问题，第一个是从传统CTR角度思考，根据出现过的搜索前缀prefix和标题title等数据提取相关特征预测当今样本的点击率，可以尝试提取相关的点击率特征来解决；第二个是从搜索前缀prefix和标题title的语义相似度判断用户点击的概率，相似度越高代表用户点击的概率越大，可以尝试从文本语义匹配特征或者是神经网络的角度来解决。  
  
* 3 **数据分析与数据清洗**  
由于本次比赛的字段比较少，而且每个字段都有特定的含义，在初步分析过后发现数据中明显的噪声并不多，所以没有做比较特别的数据清洗。在从传统CTR角度做了一个baseline模型后，在线下验证时发现验证集中新prefix的预测值偏低，因此开始研究整个比赛的数据分布。  
（1）训练集，验证集和测试集的数据分布（初赛）  
思考：验证集中新prefix样本预测值偏低原因：统计特征是针对整个训练集的统计，也就是训练集中的数据都有其对应的历史统计特征。然而验证集中却有一部分数据在训练集中从未出现过，但模型在训练时却并没有碰到过带有缺失值的统计数据，因此在验证集的新数据上的预测自然会出现偏差。因此我们开始进行EDA发现以下：  
1、对于prefix来说，无论新旧数据，点击率的均值都是0.37+；而对于title，旧数据的点击率是0.37，但新数据的点击率是0.32，有一定的区别；  
2、有些统计特征是只缺失prefix，有些是只缺失title，有些则是prefix跟title都缺失。具体分布情况如下（后期主要关注prefix和title的新样本率）：  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-b423bf65491eb735.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
3、猜想出题方构造数据集的场景，这三个测试集应该是从一个全集的数据库中随机抽样出来的。比如训练集抽200w，再抽5w训练集，这5w训练集自然而然会有一些样本是原先训练集没有的。之后再抽5w的测试集，同样也会有之前两个数据集没有的。而在分布特性上还有个最直观的的数据就是新旧prefix和title的平均样本量。  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-77bd6d68a4f62396.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
新数据的平均样本量要明显低于整个数据集的均值，正好符合抽样时小概率样本容易缺失的规律。（从一个数据集中抽取数据时，样本量少的，即小概率样本，被抽到的概率相对来说就会比较小，所以在验证集和测试集中新prefix和title的平均样本量就会低）  
（2）训练集、验证集和测试集的数据生成猜想  
从训练集的样本记录中，我们发现了很多prefix是扎堆出现的：要不就是直接连着好几个prefix都一样，要不就是中间穿插着一些其他的prefix，还有些是临近的prefix内容比较接近。感觉这样的数据分布有点像是后端的日志记录。根据比赛最开始对数据场景的分析理解，用户的一次搜索行为应该会产生多条记录。比如推荐给用户的title有3个，那么就会对应产生3条样本数据，这三条样本数据要不只有一个被点击（标签是1），要不就全部都没被点击。如果这个样本数据真的是日志记录，那么就有时序性的关系在里面，也就不难理解为什么总是有多条相同prefix的记录相邻出现。  
  
* 4 **训练样本构造**  
本次比赛，其实构造出跟测试集接近的训练样本是关键点，因此研究数据分布，构造合适的训练样本基本贯穿了我们这次比赛，接下来详细说一下我们研究数据分布的几个阶段，以及每个阶段我们是如何构造训练样本的：  
（1）初赛  
在初赛初期，我们基于传统ctr的做法构造了一个baseline，直接将训练集基于全局提取点击率等特征（还没有考虑数据穿越的问题），结果发现验证集中新prefix样本预测出来的概率偏低，而实际新旧prefix对应的样本点击率都是0.37左右，因此我们开始研究如何构造出跟验证集和测试集分布接近的数据用以训练，从而“教会模型”如何预测有缺失值特征的样本。  
*****在训练集中模拟出新数据的样本（队友研究的）*****  
通过设定一定比例的只缺失prefix，只缺失title，同时缺失prefix和title样本，提取转化率，拟合具有缺失值的样本，具体做法如下：  
1、同样是对训练集复制一份“copy”数据集用来构造缺失特征；  
2、将样本打乱后，根据验证集统计的三种情况的比例，将42.33%的样本的prefix设置为空，11.7%的样本的title设置为空，剩下的样本prefix和title都设置为空； 
3、用训练集统计数据，构造copy数据集的样本特征；  
4、模型训练时根据缺失样本比例从copy中抽样并入训练集中，线下为30%，线上为15%；  
*****发现全局提取特征存在数据穿越问题，参考腾讯赛中无时间信息数据点击率特征的提取方法，进行五折交叉提特征，发现较好拟合了数据分布*****  
1、五折交叉提特征的具体做法：将训练集划分成五份，每一份的点击率特征是基于其他四份数据提取的，与全局提特征不同；  
2、五折交叉提特征避免穿越：五折交叉提特征时，每一折的数据的点击率特征都是基于其他四折数据提取的，跟本折数据没有关系，不会把自己算进去，这样就不会有穿越问题，不会利用到未来信息；  
3、五折交叉提特征拟合分布：前面在数据分析说过，很有可能主办方是先抽取200w训练集，再抽取5w验证集，最后抽取5w测试集。我们在提取点击率特征时，验证集是基于训练集来提的，测试集是基于训练集和验证集来提的，自然会有新的prefix和title样本出现。而当我们进行五折交叉提特征时，每一折是基于其他四折来提取的，这样也会产生新的prefix和title样本，过程一致，合理；  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-dcd3b1ccf5457498.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
4、五折相对于其他折数要好：为什么5折是最好的？尝试过40折，但是发现结果爆跌，3折和10折的成绩也都不如5折的好，我们猜测这个跟分布比较相关，但是分析过后没发现什么规律  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-66d21552723a449a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
我们对比了从训练集模拟新样本和五折交叉提特征两种方法，发现五折交叉提特征的方法相对来说要好一点，因此初赛一直沿用这种方法，最后构造训练样本的过程：训练集中的统计特征采用五折交叉提取的方法来做，验证集中的统计特征基于训练集来提取，测试集中的统计特征基于训练集和验证集来提取。  
（2）复赛  
复赛一开始拿到数据，我们就对数据分布进行了分析，好消息是复赛的验证集跟测试集同分布了，意味着线下用验证集验证会比初赛更准确。坏消息是测试集的新样本率暴增，训练集五折抽样的新样本率与验证集和测试集相去甚远，基本不再适用。  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-824fe47ecd9ca3b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
我们发现，新prefix的平均样本量竟然和旧数据差不多，甚至还更高，这根本就不是普通样本抽样应该会产生的数据分布。我们猜想，这近40%的新prefix数据，怕是出题方硬塞进去的。而由于样本的prefix跟title之间有很强的相关性，所以间接带动了title的新样本量的上升，至少从新旧title的平均样本量来看，title还有比较明显的抽样稀释的规律。后来又发现了在验证集和测试集中，有一条很明显的60%的新旧数据交界线，也证明了这一点。  
针对这一点，我们开始进行分工，一部分人进行模型的特征融合（因为这个比赛限制了只能使用两个模型），之前研究新样本拟合的队友进行数据分布拟合，结果如下：  
*****仿造官方从训练集里拿出一整份的新prefix来做特征******  
1、复制一份train数据集；  
2、对复制出来的数据集按prefix进行分组归类，然后再对prefix进行五折交叉统计特征，此时相同prefix的样本只会被分配到同一折中；  
3、上一步诞生出来的只是200w的“新prefix”数据集。对原本的train数据集，还是采用原来的五折交叉统计，为了避免这里五折产生的数据集影响到后边的新旧数据比例计算，因此对数据集中空缺了prefix或title的数据进行剔除，因此得到的是约170w的“旧prefix”数据集；  
4、按照40%的prefix新旧数据比例，从新prefix数据集中随机抽样一部分数据，并入上一步产生的“旧prefix”数据集中，使得最终数据集中的prefix新旧比例为4:6。由此得到的大约是290w的训练集数据；  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-ae10849c5caee137.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
*****发现随机与不随机的5折有着重大区别*****  
在做特征融合到主模型的时候，我们发现主模型（师兄的模型初赛成绩较好，我们用来当做复赛lgb的主模型）在做五折交叉时，用的不是StratifiedKFold，也没有shuffle，直接就是KFold，一开始也并没有觉得打乱跟不打乱有什么区别，只是觉得不打乱也可以，就没有去改过。后面留意到了训练集中prefix经常扎堆出现的情况，所以当看到kfold时，就开始怀疑打乱跟不打乱的kfold到底等不等价，结果发现不打乱的5折竟然也很接近测试集的分布：  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-20ad822d05e0f3a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
*****线上数据训练模型分布抽样方案*****  
我们之前做EDA一直都知道在线下不打乱5折的训练集跟测试集分布比较接近，那么对于线上的训练集与验证集合并后的数据，是不是对它们进行不打乱的5折依旧能保持这种比较接近的分布？结果我们通过EDA发现，新样本率已经不一致了，其他一些特征的均值对比以前也出现了稍大点的偏差。  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-10fedea7132b8686.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
最后我们采用的方案是，对训练集进行五折交叉统计，用训练集统计验证集，再把训练集跟验证集合并起来。这样构造出来的线上训练集依旧保持了与测试集接近的分布，而且由于完整保留了同分布的验证集，使得合并后数据集的相比单纯五折的训练集的分布要更加接近测试集。  
  
* 5 **特征工程**  
本次比赛由于限制了模型的数量，因此我们队只保留了一个lgb主模型，通过特征融合的方式，把我们初赛每个人所做的lgb模型的想法融合进来。我们本次比赛的最终主模型特征可以分为以下几个部分：  
（1）点击率相关特征（没有做贝叶斯平滑）：  
prefix  title  query_prediction  tag的五折点击率；  
prefix  title  tag两两之间交叉的五折点击率；  
思考：点击率相关的特征其实反映了某个维度的数据在过去一段时间的表现规律，所以在本时段他很有可能也有这个规律，因此提取点击率特征会有效果。  
（2）语义相似度特征：  
prefix和title的编辑距离，jaccard相似度，word2vec出来向量的余弦距离等；  
prefix和query_prediction各个key的编辑距离，jaccard相似度，word2vvec出来向量的余弦距离之和（这里针对每个key都乘上了对应的概率值）；  
title是否存在于query_prediction的key中；  
prefix是否是title的子字符串；  
思考：依据业务场景，prefix是用户的输入，title是推荐的内容，通常来说，title的内容和prefix内容（语义）越接近，用户点击的概率越大。  
（3）文本特征：  
title的长度；  
prefix的长度；  
query_prediction的长度；  
title的长度减去prefix的长度；  
prefix的长度与title的长度比值；  
query_prediction各个key长度的均值减去prefix的长度；  
query_prediction各个key长度的均值与title的长度比值；  
prefix在title中的位置，比如prefix为‘腾讯’，title为‘欧普大战腾讯’，那么返回的值为4，即prefix在title中的第四个字符位出现；  
prefix、title和query_prediction出来的50维word2vec表征特征；  
（4）统计特征：  
title和query_prediction中各个key的编辑距离的sum，最大值，最小值，均值，方差；  
title和query_prediction中各个key的余弦距离的sum，最大值，最小值，均值，方差；  
query_prediction预测概率值的最大、最小、均值、方差；  
（5）占比特征：  
prefix+title的出现次数/prefix的出现次数；  
（6）rank特征：  
tag_ctr在title下的排名；  
title_ctr在tag下的排名；  
  
附：几种距离的具体意义  
编辑距离：又称Levenshtein距离，是指两个字串之间，由一个转成另一个所需的最少编辑操作次数。许可的编辑操作包括将一个字符替换成另一个字符，插入一个字符，删除一个字符。一般来说，编辑距离越小，两个串的相似度越大。  
Jaccard相似度：两个集合的交集除以两个集合的并集，所得的就是两个集合的相似度。  
余弦距离：余弦相似度用向量空间中两个向量夹角的余弦值作为衡量两个个体间差异的大小。相比距离度量，余弦相似度更加注重两个向量在方向上的差异，而非距离或长度上。  
  
* 6 **模型选择**  
本次比赛最主流的模型还是lgb，鉴于主办方限制了模型的使用个数（不能超过两个），因此很多队伍都是做了一个lgb模型，然后再做一个nn模型用于匹配prefix和title的语义相似度。我们队伍也是做了一个lgb主模型和一个nn模型，由于我们nn模型是最后一天才融合进来，所以无法得知它的效果，接下来就先主要介绍我们的lgb模型，nn模型由于是队友做的，目前还不是特别清楚其具体的技术实现细节，这部分就等了解清楚后再补充。  
（1）lgb主模型  
*****训练方式*****  
我们本次比赛主模型lgb在线下验证时，采用训练集训练，验证集作为模型early_stopping的验证条件，通过验证集的表现来衡量特征/模型的好坏。而在线上的时候，我们是将训练集和验证集同时作为训练数据，至于模型的迭代次数，则根据线下最佳模型的迭代次数来设定。  
我们尝试过模型训练时不用验证集作为早停，而是从训练集中抽一部分数据作为早停的valid，再用迭代出的次数训练全部训练集来预测验证集，线下训练时很快就发现这种训练方案的迭代次数从原来的200多暴涨到了1000多，线上成绩也不例外的降了2个千。于是便意识到，抽样后的训练集分布到底并非跟验证集和测试集完全一致，因此官方才会提供了验证集以便我们用来验证模型在不同分布的数据上的鲁棒性。  
*****通过对验证集加权来拟合分布*****  
鉴于我们分析过，其实验证集与测试集的分布是最一致的，我们尝试过通过加大验证集的样本权重，使得线上模型的分布更偏向于验证集的分布（验证集与测试集同分布）。由于我们当时是同时加了几个元素进去（同时加了特征和改了验证集提取特征的方式），无法知道具体的提升，当时目测有两个千。  
*****对f1评价函数的探讨*****  
初赛我的模型采用的是AUC，之所以选择AUC而不是logloss，是因为想到F1指标是基于混淆矩阵的统计，AUC也是基于混淆矩阵的指标（只不过无视了阈值的影响），auc和f1其实都关注样本概率的相对大小，而不关心预测概率的均值，而logloss还比较关心预测概率均值，所以直觉上AUC会比logloss更适合。初赛阈值的确定，主要是根据验证集搜索遍历出来五个比较好的阈值来确定的，其实很明显可以看出是有两个峰值，一个是在0.37附近，一个是在0.4附近，我通常会选择第二个峰值作为阈值。  
复赛我们是以师兄的模型为主模型，师兄的模型采用的是logloss，我们有对比过auc和logloss的效果，下面的表格中每种评价指标都分别用不同随机种子测了三组结果：  
![image.png](https://upload-images.jianshu.io/upload_images/12207295-4dd28e9107f6ddc6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
迭代次数上AUC普遍比logloss要少几十次。模型早停的评分上，AUC与logloss的抖动都是0.001左右，从模型迭代过程的输出来看，两个指标的迭代都是比较稳定的，没有抖动很厉害的情况。最佳F1评分，线下测出来logloss的成绩似乎还要好一点点，最佳阈值的均值其实也差不太多。  
赛后我们打听了其他队伍自定义f1评价指标的做法，关于评价指标中阈值的选择，主要有两种思路：一种是将阈值当做模型的超参数，整个模型迭代过程都使用这一个阈值进行F1分数的计算，最佳阈值查找就跟搜索模型最优参数一样；另一种思路则是每一步迭代时，都去寻找最佳阈值的位置，把阈值对应的最佳F1评分作为本次迭代的分数。  
*****数据后处理*****  
在数据分析中我们曾经提到，我们发现了prefix扎堆出现的情况，具有时序性，用户的一次搜索行为应该会产生多条记录。比如推荐给用户的title有3个，那么就会对应产生3条样本数据，这三条样本数据要不只有一个被点击（标签是1），要不就全部都没被点击。于是，我们将连续的相同prefix且不同title_tag的记录判定为用户的一次搜索行为，在一组记录中，若模型预测有多个记录为1，则只保留模型预测概率最大的记录为1，其他记录更改为0。  
在线下的5w验证集上，这个后处理规则总共修改了13条预测记录，全部修改正确！因此这个假设很可能是成立的。这条后处理规则最后被我们用到了B榜最后一次模型提交上，在20w测试集上总共修改了117个数据，但由于最后一次提交变量太多，不确定效果究竟如何。假设这117个数据全部修改正确的话，大约是5个万的提升。  
（2）nn模型  
待补充！！！  
  
* 7 **赛后总结**  
本次比赛跟CTR和文本都有关联，因此我们都比较感兴趣。但由于开始时间跟神策杯的时间有一点冲突，我在北京答辩完回到学校才开始做，从开始到初赛结束时间跨度大概是一个星期左右，所以初赛用的时间和精力并不多。初赛主要花时间在研究分布和数据集构造上面，还有就是做特征工程，提取比较有用的特征，所以没有时间去研究nn那块。到了复赛组好队，我们队出现了一个问题，就是关于word2vec的随机性问题，我们发现我们复现不了线上最好的模型对应的word2vec特征。由于我们队直接加入了跟word2vec相关的150维特征，这样每次word2vec的结果对我们模型的精度影响很大，很容易产生波动，所以我们在复赛还在研究怎么保证word2vec特征的复现，以及复现一下跟最好成绩比较接近的word2vec模型。此外，复赛我的精力还在做特征融合那块，而且有队友对nn比较熟悉，所以没有特别多时间做nn，这是比较可惜的。由于现在比赛还没答辩，所以我们也拿不到特别好的方案，只是在一些大佬那里打听到一些本次比赛的关键点，如下：  
（1）关于点击率的平滑问题：据说，到了复赛，做了贝叶斯平滑和没做贝叶斯平滑成绩差了大概4到5个千。我们队在复赛没做贝叶斯平滑，我在初赛的时候是做了贝叶斯平滑的，但是当时没什么效果，做和没做差别不大，到了复赛就没想着把主模型的点击率特征做一下贝叶斯平滑，这实在可惜；  
（2）关于数据分布的问题：其实前排都知道数据分布很关键，每个队伍都有自己的方案，据说，因为验证集分布跟测试集最接近，第一名植物他们是直接拿验证集5w数据来训练，而训练集的数据只是拿来提取特征，很大胆，不过确实很有效果，这就是冠军方案吧。我们也想过，但是没去做，发现了问题，却没有用了最好的方法解决。  
  
附：  
保证word2vec模型能复现的方法：设定word2vec运行的核数是1，在运行word2vec模型是时，需要保证当时系统的环境变量PYTHONHASHSEED是同一个数，也可以通过设定系统临时变量来做，例：echo PYTHONSEED = 2018  

