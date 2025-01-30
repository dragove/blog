---
layout: "../../layouts/MarkdownLayout.astro"
title: "一篇文章入门大语言模型开发（通义千问 + LangChain 本地pdf问答）"
date: "2023-10-06"
---

- > 大语言模型已经火了很久了，希望这篇文章能帮到还没尝试过的新手
- 本教程特色
	- 使用可离线使用的本地模型，确保数据安全
	- 使用最新最好用的 python 包管理技术，确保除硬件相关外功能容易复现
	- 作者现学现卖，更适合没有相关技术背景的人
	- 代码量极少，不教 python，但教调包，本文代码即使你不会 python，我相信聪明的你也能读懂，来和我一起成为调包侠吧！
- 词汇表
	- vector（向量）：表示方向和大小的数学概念，由有限个数字组成，例如向量 (3, 4) 表示二维空间中从点 (0, 0) 指向点 (3, 4) 的向量，数字越多表示维度越大，同时由原点出发的两个向量可以通过计算向量夹角的cosine值来判断相似度，cosine值越大表示两个向量越相似（余弦函数在180度以内夹角越小值越大）
	- embedding（嵌入）：嵌入是将文本转换为向量的技术，目前基本也是由深度学习模型实现该过程。
	- 推理：使用训练好的模型，给定模型输入数据，模型处理、计算和转换出最终结果的过程称作推理
- 硬件要求说明
	- 如果仅使用cpu推理，则要求设备至少有16GB内存（在笔者的设备上hello world测试代码大概占用13.8G内存）
	- 如果使用gpu推理，则要求设备至少有8G内存，显卡要求为Nvidia显卡且显存至少为7G（笔者的gpu为RTX 3070 maxq 8G 刚比要求多一点，但是在多轮对话中仍然会爆显存）
	- 当然，如果你有 openai gpt 的 key 可以跳过通义千问部署运行的部分，直接从 langchain 部分开始学习，代码请自行调整
- 准备工作
	- 自备科学上网工具，为了方便，请开启透明代理（例如tun模式）
	- [Rye](https://rye-up.com/)
		- Rye 是 python 的一站式包管理工具，就像 npm、maven 这类工具对于 node.js 和 java 一样，rye 更近一步，它本身不依赖 python 就能运行，同时，它还能管理 python 的版本。
		- 安装 Rye
			- Windows
			  
			  ```powershell
			  # Windows 用户，使用 powershell
			  # 安装包管理器 scoop
			  Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
			  irm get.scoop.sh | iex
			  # 添加安装源
			  scoop bucket add main
			  scoop bucket add extras
			  scoop update
			  # 安装 rye
			  scoop install rye
			  ```
			- Mac OS
			  
			  ```bash
			  # MacOS 用户，使用 bash 或者 zsh
			  # 安装 homebrew
			  /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
			  # 安装 rye
			  brew install rye
			  ```
			- Linux
			  如果你的包管理器能帮你解决，就用包管理器，如果不能，则使用官方安装方式
			  
			  ```bash
			  curl -sSf https://rye-up.com/get | bash
			  ```
	- GPU 推理用户额外操作 （Mac 用户请直接跳过，因为 Mac 没有 nvidia 显卡用）
		- 安装 cuda
		  
		  ```bash
		  # Windows 用户可以用 scoop 安装
		  scoop install cuda
		  
		  # Linux 用户请通过包管理器安装，例如
		  sudo pacman -S cuda
		  # 更多详细信息移步 nvidia 官方安装文档 https://developer.nvidia.com/cuda-downloads
		  ```
	- 创建一个项目
		- 找到一个放置项目的风水宝地，cd 进去，然后使用 rye 创建项目
		  
		  ```bash
		  # 个人爱好的目录组织方式，请替换为你自己习惯的目录
		  cd ~/Workspace/Python
		  # 使用 rye 创建项目，项目名称为 chatllm
		  rye init chatllm
		  cd chatllm
		  # 为了简单，我们直接删除 src 目录
		  rm -rf src
		  ```
		- 给项目绑定一个python版本并运行
		  
		  ```bash
		  # 指定使用 python 3.11，最新的 python 3.12 于 10月2日发布，暂不建议使用
		  rye pin cpython@3.11
		  # 执行同步，该操作会帮你下载这个项目的所有依赖和python解释器
		  rye sync
		  # 执行进入带有当前项目环境的bash
		  rye bash
		  # 执行 python
		  python
		  # 退出python
		  exit()
		  ```
- 部署运行大模型
	- [通义千问](https://github.com/QwenLM/Qwen)
	  通义千问 是阿里云开源的一个大语言模型，对中文的支持较好，是目前最好的开源中文模型，由于目前开源的最好的版本Qwen-14B对设备要求较高，本教程将使用Qwen-7B作为演示（GPU则使用 Qwen-7B-int4 量化模型以降低对显存的要求）
	- 添加依赖
	  
	  ```bash
	  rye add transformers torch tiktoken einops  transformers_stream_generator accelerate optimum auto-gptq
	  # 添加依赖后要执行 sync 才会下载依赖到当前环境
	  rye sync
	  ```
	- 使用你最喜欢的开发工具（例如 vscode 或者 PyCharm）在这个目录下新建一个 `common.py` 文件，写入以下内容
	  
	  ```python
	  from transformers import AutoTokenizer, AutoModelForCausalLM
	  
	  # CPU 用户请使用以下代码，注意 CPU 推理可能较慢，需要耐心等待
	  # tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen-7B-Chat",
	  #                                           trust_remote_code=True)
	  # model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen-7B-Chat",
	  #                                              device_map="cpu",
	  #                                              trust_remote_code=True)
	  # GPU 用户请使用以下代码
	  tokenizer = AutoTokenizer.from_pretrained("Qwen/Qwen-7B-Chat-Int4",
	                                            trust_remote_code=True)
	  model = AutoModelForCausalLM.from_pretrained("Qwen/Qwen-7B-Chat-Int4",
	                                               device_map="cuda",
	                                               trust_remote_code=True)
	  
	  
	  model = model.eval()
	  
	  
	  def chat(prompt, history=[]):
	      res, his = model.chat(tokenizer, prompt, history)
	      return res, his
	  
	  
	  def print_chat(prompt, history=[]):
	      print(f"Q: {prompt}\n")
	      response = chat(prompt, history)[0]
	      print(f"A: {response}\n")
	      
	  
	  if __name__ == "__main__":
	      print_chat("你好")
	  ```
	- 尝试运行
	  
	  ```bash
	  # 使用命令行运行或者直接使用IDE来运行（vscode可以直接按F5运行，pycharm右键文件可以运行）
	  python common.py
	  
	  # 输出大概长下面这个样子
	  Q: 你好
	  A: 你好！有什么我能帮助你的吗？
	  ```
- 学会构建 prompt
	- 这部分内容基于吴恩达的 Prompt Engineering 课程编写，由于是快速入门，这里只写几个要点，完整课程可以在 [deeplearning.ai](https://www.deeplearning.ai/short-courses/chatgpt-prompt-engineering-for-developers/) 上获取到，该网站还有很多其他的免费课程，其课件的中文翻译可以从 [面向开发者的 LLM 入门课程](https://datawhalechina.github.io/llm-cookbook) 中获取
	- prompt 编写原则总结
		- 指令单一且清晰
		- 用分隔符（如 ```，"""，< >，<tag> </tag>）拆分输入的不同部分
		- 明确告诉模型你需要的数据返回结构（例如json）
		- 给模型提供一些问答样例
	- 笔者的prompt示例代码，在项目下新建一个 `prompt.py` 文件写入以下内容
	  
	  ```python
	  import time
	  from common import print_chat, chat
	  
	  
	  # 告诉gpt读取哪一部分内容做处理
	  def prompt1():
	      text = """
	      在公众场合对他人非隐私部位拍摄或录影，无论被拍摄者是否同意，拍摄者都不违反任何法律，\
	      纯拍摄行为也没有侵害肖像权的问题 \
	      但若是在公众场合拍摄他人(无论是否为隐私部位)，然后发视频上网说被拍摄的人是色狼或是做出任何有损他社会评价的评论，\
	      那么这就是真正侵害了被拍摄者的肖像权的行为 \
	      此外，在公众场合拍摄他人并且将视频上网造谣诽谤侮辱被拍摄的人，或是捏造事实诽谤的，\
	      轻者行政拘留，罚款，重者则有刑事责任 \
	      还有一点，那就是任何路人(非警察)或地铁保安或小区保安，都无权查看你的手机内存 \
	      《刑法》第二百四十六条 \
	      以暴力或者其他方法公然侮辱他人或者捏造事实诽谤他人，情节严重的，处三年以下有期徒刑 \
	      、拘役、管制或者剥夺政治权利。    """
	      print_chat(f'请帮我尽可能精简以下句子\n"""{text}"""')
	  
	  
	  # 告诉gpt生成的数据格式
	  def prompt2():
	      print_chat("生成三本书，要求包含书名、作者姓名、体裁，将结果输出成json格式，\
	              要求包含 bookId、title、authorName、genre 字段")
	  
	  
	  # 告诉gpt输出的内容的样式
	  def prompt3():
	      text = """
	      编辑并运行C语言代码有这样几步：\n
	      1. 通过包管理器安装 gcc 编译器\n\
	      2. 使用 `gcc ${filename}.c -o ${executablename}` 命令来编译代码; \n\
	      3. 在终端中输入 `./${executablename}` 来执行代码
	      """
	      print_chat(f"""
	      提供给你一段使用<s>和</s>包裹的文段。如果这段话包含了一系列的步骤，请将它重写成如下格式：
	      步骤1：……
	      步骤2：……
	      ……
	      步骤N：……
	      如果这段话不包含一系列步骤，则回答我：“文段无任何步骤”
	      <s>{text}</s>
	      """)
	  
	  
	  # 告诉gpt回复的方式
	  def prompt4():
	      print_chat("""
	      你的任务是使用统一的格式回答问题
	  
	      学生：教我什么是耐心
	  
	      老师：耐心是一种重要的品质，指在面对困难、挑战或不确定性时，保持冷静、坚定和持久的态度。\
	  耐心意味着愿意承受压力和不确定性，并保持冷静和专注。这意味着可以在面对困难和挑战时，\
	  保持耐心和毅力，不轻易放弃或失去信心。以下是一些可以培养耐心的方法：\
	  1. 挑战自己：尝试去接受一些挑战，无论是学习新技能、完成一个任务、解决一个难题，\
	  都可以让你更加有耐心。
	  2. 坚持锻炼：锻炼可以增强你的耐心和毅力。你可以通过定期锻炼来培养耐心，\
	  例如每天进行一定的有氧运动或力量训练。
	  3. 培养良好的习惯：良好的习惯可以帮助你培养耐心。例如，坚持规律的作息时间、\
	  保持良好的饮食习惯、定期休息等等。
	  4. 与他人交流：与他人交流可以帮助你培养耐心。与亲朋好友分享你的困难，\
	  寻求他们的建议和支持，可以让你更加有耐心。
	  5. 学会放松：放松可以帮助你保持冷静和耐心。你可以通过冥想、瑜伽、\
	  听音乐等方式来放松自己，这可以帮助你更好地应对压力和不确定性。
	  通过以上方法，你可以培养自己的耐心和毅力，更好地应对生活中的挑战和不确定性。
	  
	      学生：教我什么是诚信
	      """)
	  
	  
	  # 告诉gpt有哪些任务
	  def prompt5():
	      print_chat("""
	      阅读以下文章并回答问题：
	  
	      小石潭记
	      从小丘西行百二十步，隔篁竹，闻水声，如鸣珮环，心乐之。伐竹取道，下见小潭，水尤清冽。全石以为底，近岸，卷石底以出，为坻，为屿，为嵁，为岩。青树翠蔓，蒙络摇缀，参差披拂。
	  潭中鱼可百许头，皆若空游无所依。日光下彻，影布石上，佁然不动；俶尔远逝，往来翕忽。似与游者相乐。
	  潭西南而望，斗折蛇行，明灭可见。其岸势犬牙差互，不可知其源。
	  坐潭上，四面竹树环合，寂寥无人，凄神寒骨，悄怆幽邃。以其境过清，不可久居，乃记之而去。
	  同游者：吴武陵，龚古，余弟宗玄。隶而从者，崔氏二小生：曰恕己，曰奉壹。
	  
	      1、用“/”标出如下面句子的朗读节奏。
	      一、潭中鱼可百许头 二、其岸势犬牙差互
	      2、作者通过写小石潭人迹罕至、凄清幽静的环境，意在表现怎样的思想感情？
	      3、请从文中找出你最喜欢的句子，并说明理由。
	      """)
	  
	  
	  def summarize():
	      print_chat("""你的任务是从电子商务网站上生成一个产品评论的简短摘要。
	  请对<s>和</s>之间的评论文本进行概括，最多30个字。
	  评论:
	  <s>
	  这个熊猫公仔是我给女儿的生日礼物，她很喜欢，去哪都带着。
	  公仔很软，超级可爱，面部表情也很和善。但是相比于价钱来说，
	  它有点小，我感觉在别的地方用同样的价钱能买到更大的。
	  快递比预期提前了一天到货，所以在送给女儿之前，我自己玩了会。
	  </s>
	      """)
	  
	  
	  lamp_review = """
	  我需要一盏漂亮的卧室灯，这款灯具有额外的储物功能，价格也不算太高。\
	  我很快就收到了它。在运输过程中，我们的灯绳断了，但是公司很乐意寄送了一个新的。\
	  几天后就收到了。这款灯很容易组装。我发现少了一个零件，于是联系了他们的客服，他们很快就给我寄来了缺失的零件！\
	  在我看来，Lumina 是一家非常关心顾客和产品的优秀公司！
	  """
	  
	  
	  def infer1():
	      print_chat(f"""
	      以下用<s>和</s>分隔的产品评论的情感是什么？\
	  评论文本: <s>{lamp_review}</s>
	  """)
	  
	  
	  def infer2():
	      print_chat(f"""以下用<s>和</s>分隔的产品评论的情感是什么？\
	  用一个单词回答：「正面」或「负面」。\
	  评论文本: <s>{lamp_review}</s>""")
	  
	  
	  def infer3():
	      print_chat(f"""
	  使用包含不超过5个词汇识别以下评论的作者表达的情感。将答案格式化为以逗号分隔的单词列表。\
	  评论文本: <s>{lamp_review}</s>\
	  """)
	  
	  
	  def infer4():
	      print_chat(f"""
	  以下评论的作者是否表达了愤怒？评论用<s>和</s>分隔。给出是或否的答案。\
	  评论文本: <s>{lamp_review}</s>\
	  """)
	  
	  
	  def infer5():
	      print_chat(f"""从评论文本中识别以下项目：
	  - 情绪（正面或负面）
	  - 审稿人是否表达了愤怒？（是或否）
	  - 评论者购买的物品
	  - 制造该物品的公司
	  
	  评论用三个反引号分隔。将您的响应格式化为 JSON 对象，以 “Sentiment”、\
	  “Anger”、“Item” 和 “Brand” 作为键。
	  如果信息不存在，请使用 “未知” 作为值。
	  让你的回应尽可能简短。
	  将 Anger 值格式化为布尔值。
	  
	  评论文本: <s>{lamp_review}</s>""")
	  
	  
	  # 忘记为什么有这个了，没什么用，直接忽略吧（逃
	  def infer6():
	      print_chat("""c++ 中如何分离模板的声明和定义？""")
	  
	  
	  def trans1():
	      prompt = """
	  将以下中文翻译成英文:
	  <s>您好，我想订购一个搅拌机。</s>
	  """
	      response = print_chat(prompt)
	      print(response)
	  
	  
	  def trans2():
	      prompt = """
	  请告诉我以下文本是什么语种:
	  <s>Combien coûte le lampadaire?</s>
	  """
	      response = print_chat(prompt)
	      print(response)
	  
	  
	  def trans3():
	      prompt = """
	  请将以下文本翻译成中文，分别展示成正式与非正式两种语气:
	  <s>Would you like to order a pillow?</s>
	  """
	      response = print_chat(prompt)
	      print(response)
	  
	  
	  def trans4():
	      user_messages = [
	          # System performance is slower than normal
	          "La performance du système est plus lente que d'habitude.",
	          # My monitor has pixels that are not lighting
	          "Mi monitor tiene píxeles que no se iluminan.",
	          # My mouse is not working
	          "Il mio mouse non funziona",
	          # My keyboard has a broken control key
	          "Mój klawisz Ctrl jest zepsuty",
	          # My screen is flashing
	          "我的屏幕在闪烁"
	      ]
	      for issue in user_messages:
	          time.sleep(20)
	          prompt = f"告诉我以下文本是什么语种，直接输出语种，如法语，无需输出标点符号: \
	              <s>{issue}</s>"
	          lang = print_chat(prompt)
	          print(f"原始消息 ({lang}): {issue}\n")
	  
	          prompt = f"""
	          将以下消息分别翻译成英文和中文，并写成
	          中文翻译：xxx
	          英文翻译：yyy
	          的格式：
	          <s>{issue}</s>
	          """
	          response = print_chat(prompt)
	          print(response, "\n=========================================")
	  
	  
	  def expand1():
	      prompt = """
	  你是一个客服AI助手，
	  你需要根据客户的评论的内容给客户发送邮件以感谢客户的评论。
	  邮件需要以“鸽子的AI助理”署名。
	  客户评论的内容下：
	  <s>
	  才买了五天就降价了！破玩意儿，再也不买了！
	  </s>"""
	      print_chat(prompt)
	      
	  
	  # 笔者的一些恶趣味
	  def chatbot():
	      history=[["你需要称呼我为主人",
	                           "好的，主人，我将使用“主人”这个称呼来称呼您"]]
	      while prompt := input("Q: "):
	          res, history = chat(prompt, history)
	          print(f"A: {res}")
	  
	  
	  
	  
	  if __name__ == '__main__':
	      # prompt1()
	      # prompt2()
	      # prompt3()
	      # prompt4()
	      # prompt5()
	      # summarize()
	      # infer1()
	      # infer2()
	      # infer3()
	      # infer4()
	      # infer5()
	      # infer6()
	      # trans1()
	      # trans2()
	      # trans3()
	      # trans4()
	      # expand1()
	      # 注意 chatbot 函数是一个交互式对话函数，需要用户手动输入内容
	      chatbot()
	     
	  ```
	- 这部分代码使用 cpu 跑会比较吃力，在笔者的设备上需要运行20分钟才能出一个结果，为了方便，笔者将自己用gpu运行的结果放到这里，供大家参考
	  
	  ```
	  - 场景1：简单的句子总结
	  Q: 请帮我尽可能精简以下句子"""
	      在公众场合对他人非隐私部位拍摄或录影，无论被拍摄者是否同意，拍摄者都不违反任何法律，    纯拍摄行为也没有侵害肖像权的问题     但若是在公众场合拍摄他人(无论是否为隐私部位)，然后发视频上网说被拍摄的人是色狼或是做出任何有损他社会评价的评论，    那么这就是真正侵害了被拍摄者的肖像权的行为     此外，在公众场合拍摄他人并且将视频上网造谣诽谤侮辱被拍摄的人，或是捏造事实诽谤的，    轻者行政拘留，罚款，重者则有刑事责任     还有一点，那就是任何路人(非警察)或地铁保安或小区保安，都无权查看你的手机内存     《刑法》第二百四十六条     以暴力或者其他方法公然侮辱他人或者捏造事实诽谤他人，情节严重的，处三年以下有期徒刑     、拘役、管制或者剥夺政治权利。    """
	  
	  A: 在公众场合拍摄他人非隐私部位，不违法。但若将视频上网造谣诽谤侮辱被拍摄者，轻者行政拘留罚款，重者有刑事责任。路人无权查看手机内存。《刑法》规定，公然侮辱他人或捏造事实诽谤他人，情节严重的，处三年以下有期徒刑、拘役、管制或剥夺政治权利。
	  
	  
	  - 场景2：返回特定格式的数据，案例使用json返回
	  Q: 生成三本书，要求包含书名、作者姓名、体裁，将结果输出成json格式，            要求包含 bookId、title、authorName、genre 字段
	  A: json
	  [
	      {
	          "bookId": 1,
	          "title": "哈利·波特与魔法石",
	          "authorName": "J.K.罗琳",
	          "genre": "奇幻"
	      },
	      {
	          "bookId": 2,
	          "title": "1984",
	          "authorName": "乔治·奥威尔",
	          "genre": "科幻"
	      },
	      {
	          "bookId": 3,
	          "title": "百年孤独",
	          "authorName": "加西亚·马尔克斯",
	          "genre": "魔幻现实主义"
	      }
	  ]
	  
	  
	  - 场景3：提前阻止生成内容
	  Q: 
	      提供给你一段使用<s>和</s>包裹的文段。如果这段话包含了一系列的步骤，请将它重写成如下格式：
	      步骤1：……
	      步骤2：……
	      ……
	      步骤N：……
	      如果这段话不包含一系列步骤，则回答我：“文段无任何步骤”
	      <s>
	      编辑并运行C语言代码有这样几步：
	      1. 通过包管理器安装 gcc 编译器    2. 使用 `gcc ${filename}.c -o ${executablename}` 命令来编译代码; 
	      3. 在终端中输入 `./${executablename}` 来执行代码    </s>
	  
	  A: 步骤1：通过包管理器安装gcc编译器。
	  步骤2：使用`gcc ${filename}.c -o ${executablename}`命令来编译代码。
	  步骤3：在终端中输入`./${executablename}`来执行代码。
	  
	  
	  - 场景4：给予LLM一个样例
	  Q:  你的任务是使用统一的格式回答问题
	      学生：教我什么是耐心
	      老师：耐心是一种重要的品质，指在面对困难、挑战或不确定性时，保持冷静、坚定和持久的态度。耐心意味着愿意承受压力和不确定性，并保持冷静和专注。这意味着可以在面对困难和挑战时，保持耐心和毅力，不轻易放弃或失去信心。以下是一些可以培养耐心的方法：1. 挑战自己：尝试去接受一些挑战，无论是学习新技能、完成一个任务、解决一个难题，都可以让你更加有耐心。2. 坚持锻炼：锻炼可以增强你的耐心和毅力。你可以通过定期锻炼来培养耐心，例如每天进行一定的有氧运动或力量训练。3. 培养良好的习惯：良好的习惯可以帮助你培养耐心。例如，坚持规律的作息时间、保持良好的饮食习惯、定期休息等等。4. 与他人交流：与他人交流可以帮助你培养耐心。与亲朋好友分享你的困难，寻求他们的建议和支持，可以让你更加有耐心。5. 学会放松：放松可以帮助你保持冷静和耐心。你可以通过冥想、瑜伽、听音乐等方式来放松自己，这可以帮助你更好地应对压力和不确定性。通过以上方法，你可以培养自己的耐心和毅力，更好地应对生活中的挑战和不确定性。
	      学生：教我什么是诚信    
	  
	  A: 诚信是一种重要的品质，指在与他人交往中，坚持诚实、公正、守信的原则，不欺骗、不欺诈、不背叛他人。诚信意味着言行一致，遵守承诺，不撒谎，不隐瞒，不欺骗他人。诚信也是一种责任感，意味着对自己的行为负责，对自己的承诺负责，对自己的行为和承诺负责。以下是一些可以培养诚信的方法：1. 坚持诚实：诚实是诚信的基础，意味着不撒谎，不隐瞒，不欺骗他人。你可以通过诚实的行为来培养诚信，例如在工作中遵守承诺，不撒谎，不隐瞒，不欺骗他人。2. 坚持公正：公正意味着公平对待他人，不偏袒，不歧视，不偏袒任何一方。你可以通过公正的行为来培养诚信，例如在工作中公平对待他人，不偏袒，不歧视，不偏袒任何一方。3. 坚持守信：守信意味着遵守承诺，不违约，不违背诺言。你可以通过守信的行为来培养诚信，例如在工作中遵守承诺，不违约，不违背诺言。4. 坚持负责任：负责任意味着对自己的行为负责，对自己的承诺负责，对自己的行为和承诺负责。你可以通过负责任的行为来培养诚信，例如在工作中对自己的行为负责，对自己的承诺负责，对自己的行为和承诺负责。通过以上方法，你可以培养自己的诚信，更好地与他人交往，建立良好的人际关系。
	  
	  
	  - 场景5：一次性给 LLM 多个任务
	  Q:  阅读以下文章并回答问题：
	      小石潭记    从小丘西行百二十步，隔篁竹，闻水声，如鸣珮环，心乐之。伐竹取道，下见小潭，水尤清冽。全石以为底，近岸，卷石底以出，为坻，为屿，为嵁，为岩。青树翠蔓，蒙络摇缀，参差披拂。潭中鱼可百许头，皆若空游无所依。日光下彻，影布石上，佁然不动；俶尔远逝，往来翕忽。似与游者相乐。潭西南而望，斗折蛇行，明灭可见。其岸势犬牙差互，不可知其源。坐潭上，四面竹树环合，寂寥无人，凄神寒骨，悄怆幽邃。以其境过清，不可久居，乃记之而去。同游者：吴武陵，龚古，余弟宗玄。隶而从者，崔氏二小生：曰恕己，曰奉壹。
	      1、用“/”标出如下面句子的朗读节奏。    一、潭中鱼可百许头 二、其岸势犬牙差互 
	      2、作者通过写小石潭人迹罕至、凄清幽静的环境，意在表现怎样的思想感情？
	      3、请从文中找出你最喜欢的句子，并说明理由。
	  
	  A: 1、一、潭中鱼/可/百许头 二、其岸势/犬牙/差互
	  2、通过写小石潭人迹罕至、凄清幽静的环境，意在表现作者被贬后孤独、抑郁的心情，以及对自然美景的热爱和欣赏。
	  3、我最喜欢的句子是“潭中鱼可百许头，皆若空游无所依。日光下彻，影布石上，佁然不动；俶尔远逝，往来翕忽。似与游者相乐。”因为这句话生动形象地描绘了潭中鱼儿自由自在、活泼可爱的样子，给人以美的享受。
	  
	  
	  - 场景6：复杂一点的总结
	  Q: 你的任务是从电子商务网站上生成一个产品评论的简短摘要。请对<s>和</s>之间的评论文本进行概括，最多30个字。评论:
	  <s>
	  这个熊猫公仔是我给女儿的生日礼物，她很喜欢，去哪都带着。公仔很软，超级可爱，面部表情也很和善。但是相比于价钱来说，它有点小，我感觉在别的地方用同样的价钱能买到更大的。快递比预期提前了一天到货，所以在送给女儿之前，我自己玩了会。</s>
	      
	  A: 这个熊猫公仔非常可爱，面部表情和善，但相比价格稍小。
	  
	  
	  - 场景7：总结用户情感
	  Q: 
	      以下用<s>和</s>分隔的产品评论的情感是什么？评论文本: <s>
	  我需要一盏漂亮的卧室灯，这款灯具有额外的储物功能，价格也不算太高。我很快就收到了它。在运输过程中，我们的灯绳断了，但是公司很乐意寄送了一个新的。几天后就收到了。这款灯很容易组装。我发现少了一个零件，于是联系了他们的客服，他们很快就给我寄来了缺失的零件！在我看来，Lumina 是一家非常关心顾客和产品的优秀公司！</s>
	  
	  A: 这条产品评论的情感是积极的。
	  
	  Q: 以下用<s>和</s>分隔的产品评论的情感是什么？用一个单词回答：「正面」或「负面」。评论文本: <s>
	  我需要一盏漂亮的卧室灯，这款灯具有额外的储物功能，价格也不算太高。我很快就收到了它。在运输过程中，我们的灯绳断了，但是公司很乐意寄送了一个新的。几天后就收到了。这款灯很容易组装。我发现少了一个零件，于是联系了他们的客服，他们很快就给我寄来了缺失的零件！在我看来，Lumina 是一家非常关心顾客和产品的优秀公司！</s>
	  
	  A: 正面
	  
	  Q: 
	  使用包含不超过5个词汇识别以下评论的作者表达的情感。将答案格式化为以逗号分隔的单词列表。评论文本: <s>
	  我需要一盏漂亮的卧室灯，这款灯具有额外的储物功能，价格也不算太高。我很快就收到了它。在运输过程中，我们的灯绳断了，但是公司很乐意寄送了一个新的。几天后就收到了。这款灯很容易组装。我发现少了一个零件，于是联系了他们的客服，他们很快就给我寄来了缺失的零件！在我看来，Lumina 是一家非常关心顾客和产品的优秀公司！</s>
	  
	  A: 积极, 肯定, 满意
	  
	  Q: 
	  以下评论的作者是否表达了愤怒？评论用<s>和</s>分隔。给出是或否的答案。评论文本: <s>
	  我需要一盏漂亮的卧室灯，这款灯具有额外的储物功能，价格也不算太高。我很快就收到了它。在运输过程中，我们的灯绳断了，但是公司很乐意寄送了一个新的。几天后就收到了。这款灯很容易组装。我发现少了一个零件，于是联系了他们的客服，他们很快就给我寄来了缺失的零件！在我看来，Lumina 是一家非常关心顾客和产品的优秀公司！</s>
	  
	  A: 是
	  
	  
	  - 场景8：总结内容，推理情感，并返回成json
	  Q: 从评论文本中识别以下项目：- 情绪（正面或负面）- 审稿人是否表达了愤怒？（是或否）- 评论者购买的物品- 制造该物品的公司
	  评论用三个反引号分隔。将您的响应格式化为 JSON 对象，以 “Sentiment”、“Anger”、“Item” 和 “Brand” 作为键。如果信息不存在，请使用 “未知” 作为值。让你的回应尽可能简短。将 Anger 值格式化为布尔值。
	  评论文本: <s>
	  我需要一盏漂亮的卧室灯，这款灯具有额外的储物功能，价格也不算太高。我很快就收到了它。在运输过程中，我们的灯绳断了，但是公司很乐意寄送了一个新的。几天后就收到了。这款灯很容易组装。我发现少了一个零件，于是联系了他们的客服，他们很快就给我寄来了缺失的零件！在我看来，Lumina 是一家非常关心顾客和产品的优秀公司！</s>
	  
	  A: json
	  {
	    "Sentiment": "Positive",
	    "Anger": "No",
	    "Item": "Lumina Bedroom Light",
	    "Brand": "Lumina"
	  }
	  
	  - 场景9：文本转换（和翻译）
	  Q: 
	  将以下中文翻译成英文:
	  <s>您好，我想订购一个搅拌机。</s>
	  
	  
	  A: Hello, I would like to order a blender.
	  
	  Q: 
	  请告诉我以下文本是什么语种:
	  <s>Combien coûte le lampadaire?</s>
	  
	  
	  A: 这段文本是法语。
	  Q: 
	  请将以下文本翻译成中文，分别展示成正式与非正式两种语气:
	  <s>Would you like to order a pillow?</s>
	  
	  
	  A: 正式语气：您是否想要订购一个枕头？非正式语气：你想订购一个枕头吗？
	  原始消息 (法语): La performance du système est plus lente que d'habitude.
	  
	  Q: 
	          将以下消息分别翻译成英文和中文，并写成        中文翻译：xxx
	          英文翻译：yyy
	          的格式：        <s>La performance du système est plus lente que d'habitude.</s>
	          
	  
	  A: 中文翻译：系统的性能比平时慢。英文翻译：The performance of the system is slower than usual.
	  
	  
	  =========================================
	  原始消息 (西班牙语): Mi monitor tiene píxeles que no se iluminan.
	  
	  Q: 
	          将以下消息分别翻译成英文和中文，并写成        中文翻译：xxx
	          英文翻译：yyy
	          的格式：        <s>Mi monitor tiene píxeles que no se iluminan.</s>
	          
	  
	  A: 中文翻译：我的显示器有一些像素不亮。英文翻译：My monitor has pixels that do not illuminate.
	  
	  
	  =========================================
	  原始消息 (意大利语): Il mio mouse non funziona
	  
	  Q: 
	          将以下消息分别翻译成英文和中文，并写成        中文翻译：xxx
	          英文翻译：yyy
	          的格式：        <s>Il mio mouse non funziona</s>
	          
	  
	  A: 中文翻译：我的鼠标不工作。英文翻译：My mouse doesn't work.
	  
	  
	  =========================================
	  原始消息 (波兰语): Mój klawisz Ctrl jest zepsuty
	  
	  Q: 
	          将以下消息分别翻译成英文和中文，并写成        中文翻译：xxx
	          英文翻译：yyy
	          的格式：        <s>Mój klawisz Ctrl jest zepsuty</s>
	          
	  
	  A: 中文翻译：我的Ctrl键坏了。英文翻译：My Ctrl key is broken.
	  
	  
	  =========================================
	  原始消息 (中文): 我的屏幕在闪烁
	  Q: 
	          将以下消息分别翻译成英文和中文，并写成        中文翻译：xxx
	          英文翻译：yyy
	          的格式：        <s>我的屏幕在闪烁</s>
	          
	  
	  A: 中文翻译：我的屏幕在闪烁。英文翻译：My screen is flickering.
	  
	  
	  =========================================
	  
	  
	  - 场景10：文本扩写
	  Q: 
	  你是一个客服AI助手，你需要根据客户的评论的内容给客户发送邮件以感谢客户的评论。邮件需要以“鸽子的AI助理”署名。客户评论的内容下：<s>
	  才买了五天就降价了！破玩意儿，再也不买了！</s>
	  
	  A: 尊敬的客户，
	  感谢您对鸽子的AI助理的支持和反馈。我们非常抱歉听到您对产品的不满，我们会认真对待您的意见，并不断改进我们的产品和服务。
	  如果您有任何疑问或需要帮助，请随时联系我们。我们非常重视每一位客户的反馈，您的意见将帮助我们更好地满足您的需求。
	  再次感谢您的支持和理解。
	  祝您生活愉快！
	  鸽子的AI助理
	  ```
- 添加 openai 接口支持并接入 langchain
	- 添加 openai 接口支持
	  collapsed:: true
		- 由于大部分大语言模型的生态都和openai的gpt接壤，通义千问提供了一个转换的接口，使得用户可以通过openai的调用方式调用到通义千问
		- 添加依赖
		  
		  ```bash
		  rye add fastapi uvicorn openai pydantic sse-starlette
		  rye sync
		  ```
		- 添加openai_api兼容代码，源代码可以从 [这里](https://github.com/QwenLM/Qwen/blob/main/openai_api.py) 获取，新建 `openai_api.py`文件写入以下内容
		  
		  ```python
		  # coding=utf-8
		  # Implements API for Qwen-7B in OpenAI's format. (https://platform.openai.com/docs/api-reference/chat)
		  # Usage: python openai_api.py
		  # Visit http://localhost:8000/docs for documents.
		  
		  import re
		  import copy
		  import json
		  import time
		  from argparse import ArgumentParser
		  from contextlib import asynccontextmanager
		  from typing import Dict, List, Literal, Optional, Union
		  
		  import torch
		  import uvicorn
		  from fastapi import FastAPI, HTTPException
		  from fastapi.middleware.cors import CORSMiddleware
		  from pydantic import BaseModel, Field
		  from sse_starlette.sse import EventSourceResponse
		  from transformers import AutoTokenizer, AutoModelForCausalLM
		  from transformers.generation import GenerationConfig
		  
		  
		  @asynccontextmanager
		  async def lifespan(app: FastAPI):  # collects GPU memory
		      yield
		      if torch.cuda.is_available():
		          torch.cuda.empty_cache()
		          torch.cuda.ipc_collect()
		  
		  
		  app = FastAPI(lifespan=lifespan)
		  
		  app.add_middleware(
		      CORSMiddleware,
		      allow_origins=["*"],
		      allow_credentials=True,
		      allow_methods=["*"],
		      allow_headers=["*"],
		  )
		  
		  
		  class ModelCard(BaseModel):
		      id: str
		      object: str = "model"
		      created: int = Field(default_factory=lambda: int(time.time()))
		      owned_by: str = "owner"
		      root: Optional[str] = None
		      parent: Optional[str] = None
		      permission: Optional[list] = None
		  
		  
		  class ModelList(BaseModel):
		      object: str = "list"
		      data: List[ModelCard] = []
		  
		  
		  class ChatMessage(BaseModel):
		      role: Literal["user", "assistant", "system", "function"]
		      content: Optional[str]
		      function_call: Optional[Dict] = None
		  
		  
		  class DeltaMessage(BaseModel):
		      role: Optional[Literal["user", "assistant", "system"]] = None
		      content: Optional[str] = None
		  
		  
		  class ChatCompletionRequest(BaseModel):
		      model: str
		      messages: List[ChatMessage]
		      functions: Optional[List[Dict]] = None
		      temperature: Optional[float] = None
		      top_p: Optional[float] = None
		      max_length: Optional[int] = None
		      stream: Optional[bool] = False
		      stop: Optional[List[str]] = None
		  
		  
		  class ChatCompletionResponseChoice(BaseModel):
		      index: int
		      message: ChatMessage
		      finish_reason: Literal["stop", "length", "function_call"]
		  
		  
		  class ChatCompletionResponseStreamChoice(BaseModel):
		      index: int
		      delta: DeltaMessage
		      finish_reason: Optional[Literal["stop", "length"]]
		  
		  
		  class ChatCompletionResponse(BaseModel):
		      model: str
		      object: Literal["chat.completion", "chat.completion.chunk"]
		      choices: List[
		          Union[ChatCompletionResponseChoice, ChatCompletionResponseStreamChoice]
		      ]
		      created: Optional[int] = Field(default_factory=lambda: int(time.time()))
		  
		  
		  @app.get("/v1/models", response_model=ModelList)
		  async def list_models():
		      global model_args
		      model_card = ModelCard(id="gpt-3.5-turbo")
		      return ModelList(data=[model_card])
		  
		  
		  # To work around that unpleasant leading-\n tokenization issue!
		  def add_extra_stop_words(stop_words):
		      if stop_words:
		          _stop_words = []
		          _stop_words.extend(stop_words)
		          for x in stop_words:
		              s = x.lstrip("\n")
		              if s and (s not in _stop_words):
		                  _stop_words.append(s)
		          return _stop_words
		      return stop_words
		  
		  
		  def trim_stop_words(response, stop_words):
		      if stop_words:
		          for stop in stop_words:
		              idx = response.find(stop)
		              if idx != -1:
		                  response = response[:idx]
		      return response
		  
		  
		  TOOL_DESC = """{name_for_model}: Call this tool to interact with the {name_for_human} API. What is the {name_for_human} API useful for? {description_for_model} Parameters: {parameters}"""
		  
		  REACT_INSTRUCTION = """Answer the following questions as best you can. You have access to the following APIs:
		  
		  {tools_text}
		  
		  Use the following format:
		  
		  Question: the input question you must answer
		  Thought: you should always think about what to do
		  Action: the action to take, should be one of [{tools_name_text}]
		  Action Input: the input to the action
		  Observation: the result of the action
		  ... (this Thought/Action/Action Input/Observation can be repeated zero or more times)
		  Thought: I now know the final answer
		  Final Answer: the final answer to the original input question
		  
		  Begin!"""
		  
		  _TEXT_COMPLETION_CMD = object()
		  
		  
		  #
		  # Temporarily, the system role does not work as expected.
		  # We advise that you write the setups for role-play in your query,
		  # i.e., use the user role instead of the system role.
		  #
		  # TODO: Use real system role when the model is ready.
		  #
		  def parse_messages(messages, functions):
		      if all(m.role != "user" for m in messages):
		          raise HTTPException(
		              status_code=400,
		              detail=f"Invalid request: Expecting at least one user message.",
		          )
		  
		      messages = copy.deepcopy(messages)
		      default_system = "You are a helpful assistant."
		      system = ""
		      if messages[0].role == "system":
		          system = messages.pop(0).content.lstrip("\n").rstrip()
		          if system == default_system:
		              system = ""
		  
		      if functions:
		          tools_text = []
		          tools_name_text = []
		          for func_info in functions:
		              name = func_info.get("name", "")
		              name_m = func_info.get("name_for_model", name)
		              name_h = func_info.get("name_for_human", name)
		              desc = func_info.get("description", "")
		              desc_m = func_info.get("description_for_model", desc)
		              tool = TOOL_DESC.format(
		                  name_for_model=name_m,
		                  name_for_human=name_h,
		                  # Hint: You can add the following format requirements in description:
		                  #   "Format the arguments as a JSON object."
		                  #   "Enclose the code within triple backticks (`) at the beginning and end of the code."
		                  description_for_model=desc_m,
		                  parameters=json.dumps(func_info["parameters"], ensure_ascii=False),
		              )
		              tools_text.append(tool)
		              tools_name_text.append(name_m)
		          tools_text = "\n\n".join(tools_text)
		          tools_name_text = ", ".join(tools_name_text)
		          system += "\n\n" + REACT_INSTRUCTION.format(
		              tools_text=tools_text,
		              tools_name_text=tools_name_text,
		          )
		          system = system.lstrip("\n").rstrip()
		  
		      dummy_thought = {
		          "en": "\nThought: I now know the final answer.\nFinal answer: ",
		          "zh": "\nThought: 我会作答了。\nFinal answer: ",
		      }
		  
		      _messages = messages
		      messages = []
		      for m_idx, m in enumerate(_messages):
		          role, content, func_call = m.role, m.content, m.function_call
		          if content:
		              content = content.lstrip("\n").rstrip()
		          if role == "function":
		              if (len(messages) == 0) or (messages[-1].role != "assistant"):
		                  raise HTTPException(
		                      status_code=400,
		                      detail=f"Invalid request: Expecting role assistant before role function.",
		                  )
		              messages[-1].content += f"\nObservation: {content}"
		              if m_idx == len(_messages) - 1:
		                  messages[-1].content += "\nThought:"
		          elif role == "assistant":
		              if len(messages) == 0:
		                  raise HTTPException(
		                      status_code=400,
		                      detail=f"Invalid request: Expecting role user before role assistant.",
		                  )
		              last_msg = messages[-1].content
		              last_msg_has_zh = len(re.findall(r"[\u4e00-\u9fff]+", last_msg)) > 0
		              if func_call is None:
		                  if functions:
		                      content = dummy_thought["zh" if last_msg_has_zh else "en"] + content
		              else:
		                  f_name, f_args = func_call["name"], func_call["arguments"]
		                  if not content:
		                      if last_msg_has_zh:
		                          content = f"Thought: 我可以使用 {f_name} API。"
		                      else:
		                          content = f"Thought: I can use {f_name}."
		                  content = f"\n{content}\nAction: {f_name}\nAction Input: {f_args}"
		              if messages[-1].role == "user":
		                  messages.append(
		                      ChatMessage(role="assistant", content=content.lstrip("\n").rstrip())
		                  )
		              else:
		                  messages[-1].content += content
		          elif role == "user":
		              messages.append(
		                  ChatMessage(role="user", content=content.lstrip("\n").rstrip())
		              )
		          else:
		              raise HTTPException(
		                  status_code=400, detail=f"Invalid request: Incorrect role {role}."
		              )
		  
		      query = _TEXT_COMPLETION_CMD
		      if messages[-1].role == "user":
		          query = messages[-1].content
		          messages = messages[:-1]
		  
		      if len(messages) % 2 != 0:
		          raise HTTPException(status_code=400, detail="Invalid request")
		  
		      history = []  # [(Q1, A1), (Q2, A2), ..., (Q_last_turn, A_last_turn)]
		      for i in range(0, len(messages), 2):
		          if messages[i].role == "user" and messages[i + 1].role == "assistant":
		              usr_msg = messages[i].content.lstrip("\n").rstrip()
		              bot_msg = messages[i + 1].content.lstrip("\n").rstrip()
		              if system and (i == len(messages) - 2):
		                  usr_msg = f"{system}\n\nQuestion: {usr_msg}"
		                  system = ""
		              for t in dummy_thought.values():
		                  t = t.lstrip("\n")
		                  if bot_msg.startswith(t) and ("\nAction: " in bot_msg):
		                      bot_msg = bot_msg[len(t) :]
		              history.append([usr_msg, bot_msg])
		          else:
		              raise HTTPException(
		                  status_code=400,
		                  detail="Invalid request: Expecting exactly one user (or function) role before every assistant role.",
		              )
		      if system:
		          assert query is not _TEXT_COMPLETION_CMD
		          query = f"{system}\n\nQuestion: {query}"
		      return query, history
		  
		  
		  def parse_response(response):
		      func_name, func_args = "", ""
		      i = response.rfind("\nAction:")
		      j = response.rfind("\nAction Input:")
		      k = response.rfind("\nObservation:")
		      if 0 <= i < j:  # If the text has `Action` and `Action input`,
		          if k < j:  # but does not contain `Observation`,
		              # then it is likely that `Observation` is omitted by the LLM,
		              # because the output text may have discarded the stop word.
		              response = response.rstrip() + "\nObservation:"  # Add it back.
		          k = response.rfind("\nObservation:")
		          func_name = response[i + len("\nAction:") : j].strip()
		          func_args = response[j + len("\nAction Input:") : k].strip()
		      if func_name:
		          choice_data = ChatCompletionResponseChoice(
		              index=0,
		              message=ChatMessage(
		                  role="assistant",
		                  content=response[:i],
		                  function_call={"name": func_name, "arguments": func_args},
		              ),
		              finish_reason="function_call",
		          )
		          return choice_data
		      z = response.rfind("\nFinal Answer: ")
		      if z >= 0:
		          response = response[z + len("\nFinal Answer: ") :]
		      choice_data = ChatCompletionResponseChoice(
		          index=0,
		          message=ChatMessage(role="assistant", content=response),
		          finish_reason="stop",
		      )
		      return choice_data
		  
		  
		  # completion mode, not chat mode
		  def text_complete_last_message(history, stop_words_ids, gen_kwargs):
		      im_start = "<|im_start|>"
		      im_end = "<|im_end|>"
		      prompt = f"{im_start}system\nYou are a helpful assistant.{im_end}"
		      for i, (query, response) in enumerate(history):
		          query = query.lstrip("\n").rstrip()
		          response = response.lstrip("\n").rstrip()
		          prompt += f"\n{im_start}user\n{query}{im_end}"
		          prompt += f"\n{im_start}assistant\n{response}{im_end}"
		      prompt = prompt[: -len(im_end)]
		  
		      _stop_words_ids = [tokenizer.encode(im_end)]
		      if stop_words_ids:
		          for s in stop_words_ids:
		              _stop_words_ids.append(s)
		      stop_words_ids = _stop_words_ids
		  
		      input_ids = torch.tensor([tokenizer.encode(prompt)]).to(model.device)
		      output = model.generate(input_ids, stop_words_ids=stop_words_ids, **gen_kwargs).tolist()[0]
		      output = tokenizer.decode(output, errors="ignore")
		      assert output.startswith(prompt)
		      output = output[len(prompt) :]
		      output = trim_stop_words(output, ["<|endoftext|>", im_end])
		      print(f"<completion>\n{prompt}\n<!-- *** -->\n{output}\n</completion>")
		      return output
		  
		  
		  @app.post("/v1/chat/completions", response_model=ChatCompletionResponse)
		  async def create_chat_completion(request: ChatCompletionRequest):
		      global model, tokenizer
		  
		      gen_kwargs = {}
		      if request.temperature is not None:
		          if request.temperature < 0.01:
		              gen_kwargs['top_k'] = 1  # greedy decoding
		          else:
		              # Not recommended. Please tune top_p instead.
		              gen_kwargs['temperature'] = request.temperature
		      if request.top_p is not None:
		          gen_kwargs['top_p'] = request.top_p
		  
		      stop_words = add_extra_stop_words(request.stop)
		      if request.functions:
		          stop_words = stop_words or []
		          if "Observation:" not in stop_words:
		              stop_words.append("Observation:")
		  
		      query, history = parse_messages(request.messages, request.functions)
		  
		      if request.stream:
		          if request.functions:
		              raise HTTPException(
		                  status_code=400,
		                  detail="Invalid request: Function calling is not yet implemented for stream mode.",
		              )
		          generate = predict(query, history, request.model, stop_words, gen_kwargs)
		          return EventSourceResponse(generate, media_type="text/event-stream")
		  
		      stop_words_ids = [tokenizer.encode(s) for s in stop_words] if stop_words else None
		      if query is _TEXT_COMPLETION_CMD:
		          response = text_complete_last_message(history, stop_words_ids=stop_words_ids, gen_kwargs=gen_kwargs)
		      else:
		          response, _ = model.chat(
		              tokenizer,
		              query,
		              history=history,
		              stop_words_ids=stop_words_ids,
		              append_history=False,
		              **gen_kwargs
		          )
		          print(f"<chat>\n{history}\n{query}\n<!-- *** -->\n{response}\n</chat>")
		      response = trim_stop_words(response, stop_words)
		      if request.functions:
		          choice_data = parse_response(response)
		      else:
		          choice_data = ChatCompletionResponseChoice(
		              index=0,
		              message=ChatMessage(role="assistant", content=response),
		              finish_reason="stop",
		          )
		      return ChatCompletionResponse(
		          model=request.model, choices=[choice_data], object="chat.completion"
		      )
		  
		  
		  async def predict(
		      query: str, history: List[List[str]], model_id: str, stop_words: List[str], gen_kwargs: Dict,
		  ):
		      global model, tokenizer
		      choice_data = ChatCompletionResponseStreamChoice(
		          index=0, delta=DeltaMessage(role="assistant"), finish_reason=None
		      )
		      chunk = ChatCompletionResponse(
		          model=model_id, choices=[choice_data], object="chat.completion.chunk"
		      )
		      yield "{}".format(chunk.model_dump_json(exclude_unset=True))
		  
		      current_length = 0
		      stop_words_ids = [tokenizer.encode(s) for s in stop_words] if stop_words else None
		      if stop_words:
		          # TODO: It's a little bit tricky to trim stop words in the stream mode.
		          raise HTTPException(
		              status_code=400,
		              detail="Invalid request: custom stop words are not yet supported for stream mode.",
		          )
		      response_generator = model.chat_stream(
		          tokenizer, query, history=history, stop_words_ids=stop_words_ids, **gen_kwargs
		      )
		      for new_response in response_generator:
		          if len(new_response) == current_length:
		              continue
		  
		          new_text = new_response[current_length:]
		          current_length = len(new_response)
		  
		          choice_data = ChatCompletionResponseStreamChoice(
		              index=0, delta=DeltaMessage(content=new_text), finish_reason=None
		          )
		          chunk = ChatCompletionResponse(
		              model=model_id, choices=[choice_data], object="chat.completion.chunk"
		          )
		          yield "{}".format(chunk.model_dump_json(exclude_unset=True))
		  
		      choice_data = ChatCompletionResponseStreamChoice(
		          index=0, delta=DeltaMessage(), finish_reason="stop"
		      )
		      chunk = ChatCompletionResponse(
		          model=model_id, choices=[choice_data], object="chat.completion.chunk"
		      )
		      yield "{}".format(chunk.model_dump_json(exclude_unset=True))
		      yield "[DONE]"
		  
		  
		  def _get_args():
		      parser = ArgumentParser()
		      parser.add_argument(
		          "-c",
		          "--checkpoint-path",
		          type=str,
		          default="QWen/QWen-7B-Chat-Int4",
		          help="Checkpoint name or path, default to %(default)r",
		      )
		      parser.add_argument(
		          "--cpu-only", action="store_true", help="Run demo with CPU only"
		      )
		      parser.add_argument(
		          "--server-port", type=int, default=8000, help="Demo server port."
		      )
		      parser.add_argument(
		          "--server-name",
		          type=str,
		          default="127.0.0.1",
		          help="Demo server name. Default: 127.0.0.1, which is only visible from the local computer."
		          " If you want other computers to access your server, use 0.0.0.0 instead.",
		      )
		  
		      args = parser.parse_args()
		      return args
		  
		  
		  if __name__ == "__main__":
		      args = _get_args()
		  
		      tokenizer = AutoTokenizer.from_pretrained(
		          args.checkpoint_path,
		          trust_remote_code=True,
		          resume_download=True,
		      )
		  
		      if args.cpu_only:
		          device_map = "cpu"
		      else:
		          device_map = "auto"
		  
		      model = AutoModelForCausalLM.from_pretrained(
		          args.checkpoint_path,
		          device_map=device_map,
		          trust_remote_code=True,
		          resume_download=True,
		      ).eval()
		  
		      model.generation_config = GenerationConfig.from_pretrained(
		          args.checkpoint_path,
		          trust_remote_code=True,
		          resume_download=True,
		      )
		  
		      uvicorn.run(app, host=args.server_name, port=args.server_port, workers=1)
		  
		  ```
		- 通过在命令行运行这段代码，我们就可以得到一个本地的openai服务端了
		  
		  ```bash
		  # gpu 用户可以直接使用上述代码运行
		  python openai_api.py
		  # cpu 用户请添加参数运行
		  python openai_api.py -c "QWen/QWen-7B-Chat" --cpu-only
		  
		  # 使用 curl 测试接口可用性
		  curl http://127.0.0.1:8000/v1/chat/completions \
		    -H "Content-Type: application/json" \
		    -d '{
		       "model": "gpt-3.5-turbo",
		       "messages": [{"role": "user", "content": "Say this is a test!"}],
		       "temperature": 0.7
		     }'
		     
		  # 输出结果可能如下（大语言模型存在不确定性，同一个问题可能每次给出的结果不一致）
		  {"model":"gpt-3.5-turbo","object":"chat.completion","choices":[{"index":0,"message":{"role":"assistant","content":"Sure, I'm ready to assist you with any questions or tasks you have. Just let me know what you need help with!","function_call":null},"finish_reason":"stop"}],"created":1696574042}
		  ```
	- 添加 langchain 依赖
	  collapsed:: true
		- [langchain](https://python.langchain.com/docs/get_started/introduction) 是一个语言模型开发框架，可以简化大语言模型开发
		  添加 langchain 依赖
		  
		  ```bash
		  rye add langchain
		  rye sync
		  ```
		- 编写一个 langchain 的案例
		  
		  ```python
		  from langchain.prompts import ChatPromptTemplate
		  from langchain.chat_models import ChatOpenAI
		  
		  # 构建一个Prompt模板（可以认为是话术）
		  prompt = ChatPromptTemplate.from_template("告诉我一个关于{joke_target}的笑话")
		  # 使用本地的通义千问模型，由于使用了兼容 openai 的接口，所以可以直接当作 chatgpt 使用
		  model = ChatOpenAI(openai_api_base="http://localhost:8000/v1", openai_api_key="none")
		  chain = prompt | model
		  print(chain.invoke({"joke_target": "Java开发者"}))
		  ```
		  
		  要运行该代码，请确保“添加 openai 接口支持”步骤已经完成并且启动了服务端，该代码的运行结果可能如下
		  
		  ```bash
		  content='Java开发者最喜欢的事情是什么？\n他们喜欢在代码中写入“//”注释，然后忘记它们！'
		  ```
- 使用 langchain 实现本地文档问答
	- 添加 huggingface embeddings 支持，这样 langchain 就可以使用来自于 huggingface 的句子转向量模型了
	  
	  ```bash
	  rye add sentence_transformers
	  rye sync
	  ```
	  
	  这里我们使用中文表现较好的 [stella 模型](https://huggingface.co/infgrad/stella-base-zh) 实现 embedding（具体的 embedding 模型榜单可以看[这里](https://huggingface.co/spaces/mteb/leaderboard)），接入 langchain 后的样例代码如下
	  
	  ```python
	  from langchain.embeddings import HuggingFaceEmbeddings
	  
	  model_name = "infgrad/stella-base-zh"
	  # 为了减轻 GPU 的压力，这部分使用 cpu 运行，即使使用 cpu，embedding过程也不会消耗太长时间
	  model_kwargs = {'device': 'cpu'}
	  encode_kwargs = {'normalize_embeddings': False}
	  embeddings = HuggingFaceEmbeddings(
	      model_name=model_name,
	      model_kwargs=model_kwargs,
	      encode_kwargs=encode_kwargs
	  )
	  print(embeddings.embed_query("欢迎来到e签宝"))
	  ```
	  
	  这个代码的输出是一个维度为768的向量
	- 添加 pdf 读取支持，langchain 可以使用 `pypdf` 库支持 pdf 读取
	  
	  ```bash
	  rye add pypdf
	  rye sync
	  ```
	  
	  尝试下载一个 pdf 并用 langchain 读取内容，这里我使用 hadoop 的论文 hdfs 为例，你可以从 [hdfs](https://storageconference.us/2010/Papers/MSST/Shvachko.pdf) 下载论文原文 pdf，下面代码以这个 pdf 为例读取
	  
	  ```python
	  from langchain.document_loaders import PyPDFLoader
	  pdf_path = "./hdfs.pdf"
	  loader = PyPDFLoader(pdf_path)
	  pages = loader.load_and_split()
	  print(pages[0].page_content)
	  ```
	- 将数据向量化并存储，这里使用 [Chroma](https://docs.trychroma.com/) 作为简易向量数据库（chroma 是使用 sqlite 作为底层的一个简易向量数据库，如果你有更多的数据和更高的要求，请搜索专业向量数据库），需要先安装 chromadb 依赖，之后调用 langchain 的封装直接使用，为了之后调试方便，我们可以建立一个目录持久化向量数据
	  
	  ```bash
	  rye add chromadb
	  rye sync
	  mkdir dbcache
	  ```
	  
	  下面代码将读取整个pdf，并使用上文提到的 embedding 模型转换为向量并存储到 chroma 中，最终持久化数据
	  
	  ```python
	  from langchain.embeddings import HuggingFaceEmbeddings
	  from langchain.document_loaders import PyPDFLoader
	  from langchain.vectorstores import Chroma
	  
	  embeddings = HuggingFaceEmbeddings(
	      model_name="infgrad/stella-base-zh",
	      model_kwargs={'device': 'cpu'},
	      encode_kwargs={'normalize_embeddings': False}
	  )
	  
	  loader = PyPDFLoader("./hdfs.pdf")
	  pages = loader.load_and_split()
	  
	  db = Chroma.from_documents(pages, embedding=embeddings,
	                                   persist_directory="./dbcache")
	  # 教学案例中这步也可省略
	  db.persist()
	  ```
	- 最终，我们使用 langchain 提供的 `ConversationalRetrievalChain` 类实现从 pdf 中查找问题并总结回答的流程，该类的工作原理是通过数据库获取与查询语句相似的 top k 数据（未给定参数时，k默认为4），组合所有内容，并向 LLM 提出如下结构的问题：
	  “使用如下的内容来回答用户的问题，如果你不知道答案，直接回复你不知道，不要尝试编造回答。\n————————\n” + topk文档内容 + 分隔符 + “user: {问题内容}”
	  不幸的是，笔者的硬件不够强大，在默认 topk 设置为4的情况下显存爆了，因此笔者这里的样例将k设置为1，最终完整代码如下
	  
	  ```python
	  from langchain.chat_models import ChatOpenAI
	  from langchain.embeddings import HuggingFaceEmbeddings
	  from langchain.document_loaders import PyPDFLoader
	  from langchain.vectorstores import Chroma
	  from langchain.chains import ConversationalRetrievalChain
	  
	  embeddings = HuggingFaceEmbeddings(
	      model_name="infgrad/stella-base-zh",
	      model_kwargs={'device': 'cpu'},
	      encode_kwargs={'normalize_embeddings': False}
	  )
	  
	  loader = PyPDFLoader("./hdfs.pdf")
	  pages = loader.load_and_split()
	  
	  db = Chroma.from_documents(pages, embedding=embeddings,
	                                   persist_directory="./dbcache")
	  
	  db.persist()
	  
	  llm = ChatOpenAI(openai_api_base="http://localhost:8000/v1", openai_api_key="none")
	  pdf_qa = ConversationalRetrievalChain.from_llm(llm,
	                                  retriever=db.as_retriever(search_kwargs={"k": 1}))
	  
	  history = [] 
	  while True:
	      query = input("Q: ")
	      result = pdf_qa({"question": query, "chat_history": history})
	      print(f'A: {result["answer"]}')
	      history.append((query, result["answer"]))
	  ```
	  
	  样例运行结果如下：
	  
	  ```bash
	  Q: 总结hdfs存储文件的过程
	  A: HDFS使用DistCp工具进行大集群内的文件复制。DistCp是一个MapReduce任务，每个map任务将源数据的一部分复制到目标文件系统。MapReduce框架自动处理并行任务调度、错误检测和恢复。
	  ```
- 总结
	- 在大模型开发上，笔者也只是现学现卖的菜鸟，这篇文章主要是为了让大家入个门，对大模型开发的总体流程有个印象。实际上 langchain 提供了很多模板和流程，方便你执行各种操作，不过默认提供的一些工具默认行为会使用英文作为 prompt，对于中文用户来说最终体验可能存在不友好的情况，最终可能还是自制 prompt 会更好。
	- 基于这篇内容，我再简单扩展一些思路给大家
		- 如何实现类似 new bing 的搜索效果？
			- 我们只要接入搜索 api， 然后将所有网页搜索的结果分段并向量化，然后通过向量相似度搜索出topk并转交给 LLM 处理总结就可以了，和 pdf 问答实际上是一回事儿。
		- 如何实现知识库存储和问答？
			- 依然和 pdf 问答是一回事儿，前文提到了向量数据的持久化，实际上现在有非常多的开源向量数据库可以使用，我们只要离线地将各类知识库数据存储到向量数据库中，然后检索让 llm 总结就可以了。这里有一个问题是检索准确度可能会因为提问者的内容不专业而下降，有一个简单的方案是先将知识库中的每个内容段交给 llm 总结，并控制总结内容字数，然后将总结内容当作补充信息放到原文中进行存储，这样可以提升匹配的准确度。
		- 跨模态语音和图片结合是如何实现的？
			- 可以直接使用支持跨模态的模型实现，例如通义千问的 [Qwen-VL](https://github.com/QwenLM/Qwen-VL)，不过这部分内容笔者也没研究过，设备跟不上（逃）
			- 当然，还有一种方式是将 LLM 和其他模型结合在一起，例如使用 LLM 将用户的输入转换为文生图模型的 prompt，然后交给文生图模型生成图片，最后拿到图片链接并转换为总结文本和链接返回给用户（当然，也可以返回为json然后代码中继续处理），langchain 中有通过 api [调用 steamship 生成图片的案例]( https://python.langchain.com/docs/use_cases/more/agents/multi_modal/multi_modal_output_agent)
	- 最后，感谢阅读！最终代码可以在 [笔者的Github](https://github.com/dragove/chatllm) 上 clone 测试
