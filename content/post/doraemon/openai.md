---
title: "OpenAI体验"
date: 2023-02-26T20:16:30+08:00
tags: [
	"openai",
]
categories: [
	"科技",
]

---

{{< neteasemusic id="1428273917" >}}

OpenAI chatGPT 很火，体验了一把，哇哦之后，心想这个会成为内容创造的辅助工具，目前大部分是通过搜索寻找来解答难题，以后可能收敛到具体应用场景中了，不过底层可能还是掌握大数据公司来提供模型资源。

<!--more-->

按使用方大致分为如下场景：

1. 企业ToB常应用在Saas办公软件，低代码，客服，生物医疗，教育当中，比如微软办公软件，notion等相关笔记软件, 飞书，钉钉，企业微信等办公聊天，文档，视屏会议等软件，以及银行智能客服，生物蛋白质生成和基因测序领域等等；付费模式，开放的模型Pass平台提供模型训练，以及微调，交互api等；按使用资源和更好的体验质量速度来收费，比如国内BAT；上层SaaS服务通过租户使用更多实用便利功能组合(三方资源和内部资源整合)来付费。
2. ToC主要是UGC的场景，随着多模交互场景下的大模型出现，AIGC方面的应用应该会更多，普通玩家更多，想象空间应该也更大，这块比ToB要大的多，而且较为通用，有UGC大数据公司才可能出大模型吧，并且开放给上游应用玩家使用，按功能体验质量来付费；比如国内抖音，微信这些app应用，以及和企业合作的实验室。可能还会有其他好玩的智能硬件出现。
3. 可能还有数据库方向，结合用户经常输入查询，结合数据库产品特性进行智能补偿纠错优化推荐等，类似tabnine, [Copilot](https://docs.github.com/en/copilot) 这类型工具，反正上层交互类的场景应该都可以渗透到。

总的概括玩家分为3种：底层大模型 -> 定制场景下的数据模型微调 -> 上层应用百花齐放；

想象空间有限，未来是怎样，以上说的可能有误:)，anyway，Just Do IT~

PS: 梯子不要选择香港，可以使用美国的节点；openai如何注册可以google一下，教程很多，注册的邮箱用的gmail，使用 https://sms-activate.org/ 代理接受短信验证码，可以选择🇮🇳印度。

chatGPT体验挺有意思的，如果问一些理性逻辑相关的case, 比如自然语言处理怎么学习相关的语义，可以给出相同的参考标准答案，逻辑套路满满，而且还可以纠正错别字意图，(关于如何学习的模版套路，可以用来建个思维导图，然后自己填充学习内容笔记，或者ppt之类的等等)；如果问一些感性的case, 比如一段歌词，一首诗歌，会给出相应的场景，文字还挺优美的，一个字绝，懂你的。感觉AI很适合逻辑套路，但是人类的情感是很难琢磨的哈，人心难测嘛；当然提问也是比较重要的，也有些bad case，比如："什么是快乐星球"，需要多沟通，让它理解上下文（只能意会不能言传-人类专属功能，它倒像个三体人），继续追问，会出现一本正经的错误回答，明明是胡歌，舒淇，张艺兴等小朋友~ （模型自我迭代学习，下次访问回答时已经就正确了，知错能改，善莫大焉）

![openai-case](https://raw.githubusercontent.com/weedge/mypic/master/doraemon/openai-case.png)



openai使用GPT模型(公开使用的是GPT-3以上), 底层具体对应的大模型已经训练好了，提供openAPI给应用开发者来进行微调模型使用。

作为一个开发者，当然想在通过开放的api来使用openai模型啦；如果是研究人员，虽然木有大数据和服务器计算资源来玩，也可以使用openai开放的GPT-3以及以上的模型来微调。

以下是chatGPT 对使用的回答(这个就相当于是智能客服场景)，提供的开放的(text/image/audio/video)多模交互api使用如下：

1. GPT-3, GPT-3.5 ... 模型，用于模型参数微调；文本补全，编辑；以及搜索(相关性排序)，聚类(相似性分组)，推荐(推荐相关文本)，异常检测(识别相关性很小的异常值)，多样性测试(分析相似性分布)，分类(最相似的标签分类)等embedding，通过向量列表表示，计算文本相关性(向量距离)；主要用于文本类交互，以及基于上下文聊天场景；
2. Codex api 通过描述文本提示**Prompt**生成对应代码；这个挺适合开发人员的，对于新语言的新手，结合ide，记事本通过Prompt提示词语来生成相关代码还是挺爽的，至少不用去google 来回找确认是否是需要方案代码，搜索则可以用于兜底方案；
3. DALL-E /2 api 通过描述文本提示**Prompt**生成图片或者编辑原始图片 ；插画，设计师 辅助类工具，大概构思草图；
4. 音频转换为文本, 使用开源大型 v2 [Whisper 模型](https://openai.com/blog/whisper/)。 这个用在硬件设备上挺好的，硬件操作系统如果有开放口子可以开发的话，直接就可以对接上赋能了。将音频转录成音频所使用的任何语言；将音频翻译并转录成英文；
5. 视频类的api暂时还没有；
6. 需要生成 [api-keys](https://platform.openai.com/account/api-keys) 用于api接口调用时使用；
7. 提供了不同开发语言的 [client库](https://platform.openai.com/docs/libraries/community-libraries) ，默认包括：python,nodejs, 还有其他语言三方包，比如golang: [sashabaranov/go-gpt3](https://github.com/sashabaranov/go-gpt3) ; 以及api 错误说明；
8. 可以在 [playground](https://platform.openai.com/playground) 中编辑描述文本提示**Prompt**对模型接口调试测试，还可以用语音生成描述文本(speech to text)，适合端到端的语音智能设备；
9. 而且在 [openai examples](https://platform.openai.com/examples) 中提供各种应用场景样例和Q&A；
10. 文档中还介绍了最佳实践：[**<u>安全最佳实践</u>**](https://platform.openai.com/docs/guides/safety-best-practices) 和 [**<u>生产最佳实践</u>**](https://platform.openai.com/docs/guides/production-best-practices) ；

![openai](https://raw.githubusercontent.com/weedge/mypic/master/doraemon/openai.png)

像国内BAT在这块也早已开始布局了，大概15,16年左右就已经开始搭建智能平台底座，只不过被国外chatGPT “大力出奇迹” 给引爆了；大模型的训练需要大量的数据和参数调整，而且需要消耗大量服务器计算资源，特别是GPU 。像百度的 [文心大模型](https://wenxin.baidu.com/) （塑造了一个二次元create大会） ; 中国素有基建狂魔之称，希望能在中国版的"大力神丸"上出奇迹。

Tips: 国内微信也已经有对应类似开放api，https://welm.weixin.qq.com/docs/api/  微信应该最适合的应用场景，对应背景论文：[WeLM: A Well-Read Pre-trained Language Model for Chinese](https://arxiv.org/pdf/2209.10372.pdf)

附学习demo: 

openai提供了开放接口，借这个AI东风，推进下工程方面的熟练。大部分是dev/app ops工作，业务由应用场景和idea来决定。

1. 本地命令行交互

   目的：<u>快速熟悉openai的调用接口进行参数设置， 或者对模型进行微调训练。</u>

   源码地址：[https://github.com/weedge/craftsman/tree/main/doraemon/openai](https://github.com/weedge/craftsman/tree/main/doraemon/openai)

```shel
git clone  https://github.com/weedge/craftsman && cd craftsman/doraemon/openai
# cmd chat Q&A
export OPENAI_API_SK=
make run
```

2. web交互(使用AWS [**Serverless**](https://serverlessland.com/)  架构搭建)☁️智能底座+上层轻/微应用，适合快速迭代的业务，just code serverless biz logic handler func run on the could  [lambda runtime](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html), like shell/c++/rust use [custom runtime](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-walkthrough.html) 特别是rust lambda runtime 是开源的，值得关注，对于使用运行时语言来进行无服务化平台化改造，比如数据模型训练是的pipeline，数据库cloud平台，而且在aws内部大量使用，Rust 已迅速成为大规模构建基础设施的关键语言，[Firecracker](https://firecracker-microvm.github.io/) 是一种开源虚拟化技术，为[AWS Lambda](https://aws.amazon.com/lambda/)和其他无服务器产品提供支持。 aws抽象出来的服务，复杂都留在后面，简单交互留给用户，按业务场景自由组装infrastructure。智能底座需要这样的抽象工程给上层应用赋能。

   目的： <u>熟悉aws serverless 事件驱动整体架构，以及整体lambda runtime运行原理； 在数据库cloud 或者内部/外部pass平台场景中，提供给客户使用serverless来实现具体业务逻辑。aws在这块做的深入，通过学习以便这些思想用于实际工作场景中。</u>

   源码地址：[https://github.com/weedge/craftsman/tree/main/cloud/aws/cdk/serverless-openai-chatbot](https://github.com/weedge/craftsman/tree/main/cloud/aws/cdk/serverless-openai-chatbot)

   使用aws无服务lambda系统设施架构如下：(push模块异步对接openai 推送结果，这里分不同开放语言，是为了熟悉lambda对不同语言runtime，具体语言根据公司组织应用场景而定，不过 golang挺适合push服务的，分channel治之)

   ![architecture](https://raw.githubusercontent.com/weedge/mypic/master/doraemon/aws-serverless-openai-chatbot.drawio.png)

   按照demo readme 配置好文件，后端服务可以一键部署这个demo应用，第一次部署过程可能比较长，主要是用docker容器来CI lambda不同语言所依赖的库，用于部署至aws lambda容器环境中；前段静态资源则需要手动配置, 看 [Tutorial: Configuring a static website on Amazon S3]( https://docs.aws.amazon.com/AmazonS3/latest/userguide/HostingWebsiteOnS3Setup.html) 这个教程就可以，配置好后可提供对象存储S3域名使用，如果需要配置公司组织域名，使用CDN加速，则自行查看相关文档解决~

3. k8s部署方式

   目的：<u>熟悉k8s资源工程化部署，了解整体生态， 熟练相关工具及原理。</u>

   源码地址：[https://github.com/weedge/craftsman/tree/main/doraemon/ai-creator](https://github.com/weedge/craftsman/tree/main/doraemon/ai-creator)

Tips: 技术上 不要把openAI 放大了，对于工程化方面来说只是多了一项方便调试的智能化接口，加上了更多的赋能，应用上玩出花，也只是在原有的产品功能上定制化数据场景模型的微调，至于算法模型，大部分都开源，关键是大数据场景下的训练资源调度调优，垂直领域场景下用于参数微调训练的数据吧；对边缘模型在边缘端自适应学习调优推理，占用少的资源就能快速响应的模型，可能离机器人智能不远了。

附2 好玩的网站：(国外玩出花了)

1. https://www.notion.so/product/ai 笔记思路智能套路
2. https://www.midjourney.com/ 需要注册Discord 在channel下通过聊天命令交互 生成图片 https://docs.midjourney.com/docs/quick-start
3. https://piggy.to/ ui设计师
4. https://soundraw.io/  寻找音乐灵感
5. **https://typeset.io/** 读论文神器，非常值得推荐，so nice~
6. **https://you.com/** 新一代人工智能搜索引擎(遵从用户隐私数据，结合AI技术,chat,code,study等)，加入了社交属性，搜索质量很高；(国内搜索引擎应该有跟进一波的吧~) ；一些反馈功能还在迭代完善 https://yousearch.canny.io/；[Richard Socher](https://www.socher.org/) - [**Natural Language Processing with Deep Learning**](https://www.youtube.com/playlist?list=PL3FW7Lu3i5Jsnh1rnUwq_TcylNr7EkRe6)( [当前课程](https://web.stanford.edu/class/cs224n/)挺贵的,可以看下[以前免费视频](https://www.youtube.com/playlist?list=PLoROMvodv4rOSH4v6133s9LFPRHjEmbmJ), 感觉听不懂先用notion记录下，再去找资料了解下，orz, ps: CMU数据库课程 [Advanced Database Systems](https://15721.courses.cs.cmu.edu/spring2023/) 类似学习，今年的学习OKR有了~)

携手AI前行，效率优先，压缩时间成本~

附3 openai 官方提供的应用类产品，

1. [chatGPT](https://chat.openai.com/): 这个大家都知道一款火爆应用产品，发现社交永远是人类永恒需求哈；
2. [DALL·E 2](https://labs.openai.com/)：使用文本生成图片；有相关的提示文本推荐，体验更好；
3. [yabble](https://www.yabble.com/)：数据洞察(insights)，进行归纳终结，并且帮助规划日程，提出建议；小助手类型工具，网上有用来分析炒股的~；
4. [jukebox](https://openai.com/research/jukebox)：使用文本生成音乐；涉及到音乐版权，数据资源可能不好弄~；
5. [waymark](https://waymark.com/)：使用文本生成视频；主要用于制作电视广告和数字视频广告；

以上应用产品底层大模型大多是基于GPT相关最新模型，官网提供的GPT-3: https://openai.com/blog/gpt-3-apps介绍。

附一张midjourney 生成的图片，采用的prompt 描述如下：

```shell
/imagine prompt a cute long distance couple with souls connection, asian, chinese, fantasy style 
```

![midjourney](https://raw.githubusercontent.com/weedge/mypic/master/doraemon/midjourney-aigc-chinese-couple.png)

## reference

https://openai.com/product (可以先了解清楚openai自己的应用根源产品，后续有时间整理下，感兴趣的话，然后去发散吧)

https://openai.com/blog (技术宅，可以订阅一波)

https://platform.openai.com/docs/introduction/overview (适合开发，模型微调)

https://platform.openai.com/examples/ (找灵感)

https://gpt3demo.com/  https://gpt4demo.com/ （潘多拉盒子）

[https://github.com/mli/paper-reading#自然语言处理-transformer](https://github.com/mli/paper-reading#%E8%87%AA%E7%84%B6%E8%AF%AD%E8%A8%80%E5%A4%84%E7%90%86---transformer) (背后模型原理导读)
