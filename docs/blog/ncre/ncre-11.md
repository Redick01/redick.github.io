# 论文二 论原型模型及应用<!-- {docsify-ignore-all} -->



摘要：

​    2022年，某一线城市地铁提出建设地铁官方APP项目，由于我司在移动互联联网，轨道交通领域有一定的业务积累，所以很容易的中标此项目，我很荣幸的在此项目中担任软件架构师，期间主要负责需求收集、需求分类、系统设计、技术方案、系统资源平台、系统上线发布等工作；在项目初期与地铁业主的沟通过程中发现业主对于地铁官方APP的的需求并不明确，因此我提出了通过原型模型的软件开发模型完成此项目，经过公司内部与地铁业主的开会研究，认为我提出的通过原型模型软件开发是可行的，个并且公司和领导一致赞成改方案，最后经过7个月的时间，某市地铁官方APP成功上线，获得了业主与广大用户的一致好评。

 

正文：

​    随着移动互联网时代的到来，移动端（手机）应用在老百姓的日常生活中变得越来越重要，各行各业都兴起建设行业应用（APP），增加了企业效益的同时，也为老百姓的生活提供了便捷，2022年初，某一线城市轨道运营公司提出建设该城市“地铁官方APP项目”，APP主要的功能包含出行信息、志愿者服务、票务服务、信息发布、爱心预约、失物招领、延误证明等，轨道运营公司对外发起项目招标，我司参与招标，由于我司拥有丰富的移动应用开发经验以及拥有轨道交通领域的业务知识储备，我司有幸成功中标，项目建设价格为800万，中标后公司针对此项目成立专门项目组，我很荣幸在此项目中担任系统架构师，主要负责需求收集、需求分类、系统设计、技术方案、系统上线发布等工作。在项目开展初期经过我与地铁运营公司业主的交流发现，地铁运营公司对APP的需求很不明确，并且需求大概率经常变化，鉴于此情况发，我提出了以“原型模型”作为软件开发模型的思路，并阐述了原型模型开发模型为何适合此项目，经过公司项目组、地铁运营部的评审后，一致赞成使用我提出的“原型模型”软件开发模型来开发此项目。

​    本篇文章以此项目为例，将论述“原型模型”的软件开发模型在此项目成功交付过程中起到的作用，首先我将论述“原型模型”的基本概念，然后论述“原型模型”适用的开发场景，最后论述“原型模型”在此项目建设过程中所起到的作用。

​    原型模型提出能够快速开发出一个用户看得见摸得着的实际系统原型，能及早的交付用户体验原型系统，原型模型的流程是针对用户对系统的描述制定快速计划，通过快速的设计方式建模，然后快速构建模型，接着部署交付原型，原型提交测试，有测试人员、项目业主反馈问题并进行交流后进行改进，循环往复的进行制定快速计划、快速设计建模、快速构建模型、部署交付、测试交流，最终实现最终的用户、业主认可的原型并交付上线。能够采用原型模型也是因为开发工具的快速发展，以及项目组本身具备快速构建原型的能力。

​    原型模型非常适用用户对系统的详细需求模糊不清，这也是我选用此模型的目的之一，因为通过我与轨道运营公司业主的交流发现业主知识提出了功能点，而对功能细节无法给出具体的需求，比如，业主提出一个功能点“出行信息”，但是业主却无法清楚地表述出该功能点应该具备什么功能；其次我司具备成熟的开发环境和快速构建应用的能力，并且对轨道交通业务也十分了解，有能力快速构建原型交付到业主手中；再次经过我与业主的沟通发现此项目的规模并不是很大，所设计的业务也不是很复杂，这一点也十分契合“原型模型”开发模型的使用场景。

​    分析完“原型模型”的相关概念、开发流程、注意事项和从业主那里获取到的初始需求，我开始着手制定基于“原型模型”开发模型的开发计划。

​    研发团队根据轨道交通运营公司业主提出的初始功能点迅速构建出一个软件原型，软件原型包含“出行信息”功能，该功能包含简单的线路查询、全部站点、附近站点、出站服务设施查询功能，构建原型程序后，对原型进行发布并交付到业主手中，让业主体验原型程序，在体验期间与业主进行密切的沟通，在业主体验原型程序的过程中挖掘业主真正的需求，并且就轨道交通的业务层面的实际情况与业主进行深入沟通，及时的收敛需求的范围，因为在“原型模型”开发模型中，如果对原型多次修改，并且不收敛目标范围的话，很可能会造成项目失败。

​    初版原型程序经过业主的体验后，反复地与业主进行沟通后，细化需求，将出行信息功能细化出6个子功能：

1. 线路查询：APP向乘客提供线路查询功能，展示线网图、地铁线路、车站信息，不包含公交线路查询；

2. 附近站点：APP获取到乘车授权的位置信息后，能够在线网线路图中显示距离乘客最近的站点，并表明距离；

3. 首末车时刻表：用户在APP选择线网图中选择某条线路的某个站点后，能够显示该站点列车的首末车时间；

4. 车站服务设施查询：用户在APP选择线网图中选择某个站点后，能够查询车站的服务设施，包含卫生间、闸机口、售票机、售票厅、自动贩卖机等等车站服务设施位置；

5. 外部接驳信息查询：用户在APP选择线网图选择站点后，可以选择查询外部接口信息，用于查询外部的公交站、出租车上车点、共享单车摆放位置等；

6. 线路拥挤度查询：用户在APP中点击线路拥挤度查询，选择线路点击查询，可以查询到当前线路列车当前的拥挤度；

​    功能细化后制定快速计划，通过快速设计的方式进行建模，在对原型功能设计完成后，项目开发人员快速针对模型快速构建原型应用，构建完成后将原型应用交付到业主手中进行体验和交流，经过业主体验后，业主对“原型模型”软件开发模型理念的支持，并且与我进行了交流，提出改进建议，经过几轮的原型的快速设计、建模、构建、交付、交流的过程，APP出行功能完成最终的封版。

​    其他的功能如志愿者服务、票务服务、信息发布、爱心预约、失物招领、延误证明等，也按照原型模型的理念进行设计、建模、构建原型、交付、反馈交流进行开发，在经过近7个月的开发与迭代的过程中，某一线城市地铁官方APP项目成功上线并面向用户使用，现阶段已经该APP已经累计注册用户500万，获得了广大用户和轨道运功公司业主的一致好评，我自己也深感骄傲。

​    在此项目使用“原型模型”开发的过程中，也遇到过在讨论的过程中，由于业主对实际业务情况的认识模糊，提出一些无法实现，甚至是不着边际的需求的情况，遇到此类问题，我与业主进行了富有成效的沟通，我从业务实际情况出发，耐心的向业主进行劝解，业主了解到实际情况后也欣然接受，所以，原型模型固然能够快速交付原型程序，但是如果后续的迭代，变更的过程中，如果不收敛需求，那么轻则影响项目进度，重则整个项目都要推到重来。通过此项目让我深刻的认识到作为一个系统架构师，不仅要有过硬的技术能力，而且在系统开发过程中，选择一个合适的软件开发模型也是至关重要的，好的软件开发模型能做到事半功倍。

