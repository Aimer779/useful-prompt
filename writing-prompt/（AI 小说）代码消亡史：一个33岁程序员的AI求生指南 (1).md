![Image](https://mmbiz.qpic.cn/sz_mmbiz_png/6fHkSrNbAAtNvXCOymRF3N4bicTlGP0Omkcj7Nb2c0KDSUaKnoAfItPoBE2YaQT7pMjmK2m4bKJhkrhhKolVd2w/640?wx_fmt=png&from=appmsg&watermark=1&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)

## 第一章 代码审查

## 1

2025 年 1 月 20 日，23:47。

李晓明的手指在键盘上悬停了三秒。

屏幕上，IntelliJ IDEA的深色主题里，一段支付模块的代码静静地躺着。绿色的注释，蓝色的关键字，橙色的字符串，组成了一幅他再熟悉不过的画面。这是刘洋下午提交的pull request，commit message写着："feat: 使用AI优化支付流程，性能提升200%"。

200%。李晓明推了推眼镜。这个数字像个讽刺。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-python">// AI-generated code by GitHub Copilotpublic async processPayment(orderId: string, amount: number) {&nbsp; const lockKey = `payment_lock_${orderId}`;&nbsp; const lockValue = Math.random().toString(); // Use a random value&nbsp;for&nbsp;safe release&nbsp; const lock = await this.redisClient.set(&nbsp; &nbsp; lockKey,&nbsp; &nbsp; lockValue,&nbsp; &nbsp;&nbsp;'EX',&nbsp; &nbsp; 30,&nbsp; &nbsp;&nbsp;'NX'&nbsp; );if&nbsp;(!lock) {&nbsp; &nbsp; throw new Error('Payment already processing');&nbsp; }&nbsp; try {&nbsp; &nbsp; // Missing userId&nbsp;in&nbsp;parameters&nbsp; &nbsp; await this.deductBalance(userId, amount);&nbsp; &nbsp; await this.createTransaction(orderId, amount);&nbsp; &nbsp; await this.notifyUser(userId, orderId);&nbsp; } finally {&nbsp; &nbsp; // Unsafe release: This could delete a lock acquired by another process&nbsp; &nbsp; //&nbsp;if&nbsp;the current process's lock expired.&nbsp; &nbsp; await this.redisClient.del(lockKey);&nbsp; }}</code></section>
```

第 47 行。那个 `del`。

李晓明皱起眉头。在分布式环境下，如果 `deductBalance` 成功但 `createTransaction` 失败，这个锁会被释放，但用户的余额已经被扣除。下一次重试会再次扣款。这是个典型的分布式事务和幂等性问题。更糟糕的是，这个锁的释放方式是极其危险的，如果当前操作超时，锁被 Redis 自动释放后又被另一个请求获取，这里的 `finally` 代码块会错误地删除掉另一个请求的锁。Copilot 显然不理解这个上下文。

他删掉整个代码块，手指开始在键盘上飞舞。机械键盘发出清脆的哒哒声，在空旷的办公室里回响，像某种古老的仪式。Cherry MX 青轴，2019 年买的，键帽上的字母已经有些磨损。W、A、S、D 四个键最为明显 —— 那是他偶尔玩 FPS 游戏留下的痕迹，尽管已经一年多没打开过游戏了。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-typescript">public async processPayment(orderId: string, userId: string, amount: number) {&nbsp; &nbsp; const idempotencyKey = `payment:${orderId}`;&nbsp; &nbsp; // Use idempotency key to prevent duplicate processing&nbsp; &nbsp; const existingTransaction = await this.transactionRepo.findByOrderId(orderId);&nbsp; &nbsp;&nbsp;if&nbsp;(existingTransaction) {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;return; // Already processed&nbsp; &nbsp; }&nbsp; &nbsp; // Start a&nbsp;local&nbsp;database transaction&nbsp; &nbsp; await this.db.transaction(async (tx) =&gt; {&nbsp; &nbsp; &nbsp; &nbsp; // Use Saga pattern&nbsp;for&nbsp;distributed operations&nbsp; &nbsp; &nbsp; &nbsp; // Step 1: Deduct balance within the transaction&nbsp; &nbsp; &nbsp; &nbsp; await this.walletRepo.deduct(tx, userId, amount);&nbsp; &nbsp; &nbsp; &nbsp; // Step 2: Create transaction record within the transaction&nbsp; &nbsp; &nbsp; &nbsp; await this.transactionRepo.create(tx, { orderId, userId, amount });&nbsp; &nbsp; &nbsp; &nbsp; // Step 3: Write event to transactional outbox&nbsp;for&nbsp;async notification&nbsp; &nbsp; &nbsp; &nbsp; await this.outboxRepo.append(tx, {&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'PaymentSucceeded',&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; payload: { orderId, userId }&nbsp; &nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; });&nbsp; &nbsp; // The outbox will be processed by a separate worker to reliably notify the user}</code></section>
```

窗外，国贸三期的灯光依次熄灭。北京的冬夜，即使在 CBD 的中心，也带着一种无法驱散的寒意。中央空调的风力在晚上十点被物业准时调到了最低，为的是 “节能减排”。李晓明拉了拉身上的冲锋衣，那是 2022 年公司年会的纪念品，胸前印着已经改过两次的公司 logo。

他的目光扫过办公室。一百多个工位，现在只剩下零星几盏屏幕的光。最远处，运维的小王还在值班，耳机里大概放着郭德纲的相声 —— 李晓明能看到他时不时露出的笑容。左前方，产品经理赵姐的工位上，一盆多肉植物在屏幕的蓝光下显得格外诡异。她下午就走了，说是要去接孩子。

李晓明的视线回到屏幕上。他开始写测试用例。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-cpp">// Using Jest&nbsp;for&nbsp;testingdescribe('Concurrent Payment Tests', () =&gt; {&nbsp; it('should only process a payment once under concurrent requests', async () =&gt; {&nbsp; &nbsp; const orderId =&nbsp;"order_001";&nbsp; &nbsp; const userId =&nbsp;"user_123";&nbsp; &nbsp; const amount = 100.00;&nbsp; &nbsp; const concurrency = 100;&nbsp; &nbsp; const promises = [];&nbsp; &nbsp;&nbsp;for&nbsp;(let&nbsp;i = 0; i &lt; concurrency; i++) {&nbsp; &nbsp; &nbsp; promises.push(paymentService.processPayment(orderId, userId, amount));&nbsp; &nbsp; }&nbsp; &nbsp; // Await all concurrent calls to complete&nbsp; &nbsp; await Promise.allSettled(promises);&nbsp; &nbsp; // Assert business invariants:&nbsp; &nbsp; // 1. User's balance should be deducted only once.&nbsp; &nbsp; const finalBalance = await walletService.getBalance(userId);&nbsp; &nbsp; expect(finalBalance).toEqual(initialBalance - amount);&nbsp; &nbsp; // 2. Only one transaction record should be created.&nbsp; &nbsp; const transactionCount = await transactionRepo.countByOrderId(orderId);&nbsp; &nbsp; expect(transactionCount).toBe(1);&nbsp; });});</code></section>
```

手机震了一下。微信消息。

陈宇："老李，睡了吗？"

李晓明没有立即回复。他知道陈宇想说什么。上周的同学聚会上，陈宇就在游说他加入创业公司。"AI 辅助编程工具的下一个风口"，陈宇是这么说的。拿了 3000 万 A 轮，估值 2 个亿，期权激励，财务自由的梦想。

他看了眼时间。23:58。

又一条消息："我们真的需要你这样的架构师。不是写 CRUD，是真正的创造。"

CRUD。Create, Read, Update, Delete。这四个单词，概括了他九年程序生涯的大部分时间。即使是刚才修复的支付模块，本质上也只是 CRUD 的变种。只不过加了分布式事务，加了并发控制，加了各种保证数据一致性的机制。

但这就是创造吗？

李晓明站起身，走到窗边。十二层的高度，能看到东三环的车流。即使接近午夜，这条环路上依然有着稀疏的光点在移动。每一个光点背后，大概都是一个疲惫的灵魂。有多少是程序员？有多少在担心 35 岁之后的出路？

他想起白天的会议。张威站在投影前，PPT上是一条陡峭上升的曲线。"Q1的目标很明确，"这个比他小两岁的上司说，"全面拥抱AI工具，效率提升50%。GitHub Copilot、Cursor、Claude，该用的都要用起来。"

"这不是选择题，" 张威的目光扫过会议室，"是生存题。"

生存。

李晓明回到座位上。电脑屏幕已经进入了屏保，上面是他和林静的婚纱照。照片是 2020 年拍的，在巴厘岛的海滩，夕阳，白纱，笑容。他们终于在 2023 年完成了延期三年的蜜月旅行。那时候他刚升到高级工程师，月薪刚过 20K，觉得未来充满希望。

现在，月薪 28K，房贷 20K，希望呢？

他动了动鼠标，屏保消失。代码重新出现。他提交了修改。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-sql">git add .git commit -m&nbsp;"fix: 修复 AI 生成代码的并发与事务问题- 使用基于数据库的幂等检查和本地事务替代不安全的 Redis 锁- 引入 Transactional Outbox 模式确保消息可靠传递- 补充并发测试用例，验证业务不变量而非请求成功率根本原因：Copilot 不理解企业级分布式系统的上下文，生成的代码存在严重的数据一致性隐患 "git push origin fix/payment-concurrent-issue</code></section>
```

推送完成。他打开飞书，在刘洋的私信窗口里敲下：

"支付模块有严重 bug，已修复。明早详聊。"

发送。

然后他看到了刘洋两小时前的消息："李哥，我用Cursor重构了整个订单模块，代码量减少了60%！AI真的太强了！"

李晓明盯着这条消息，没有回复。

## 2

00:45，他错过了末班地铁。

10 号线，国贸站到团结湖站，三站。李晓明站在路边打车，整个街道只有零星的车辆。一个外卖员，骑着电瓶车飞驰而过。两个刚下夜班的商场导购，用东北口音聊着今天的业绩。角落里一对情侣，女孩靠在男孩肩上，两人共用一副耳机。

李晓明掏出手机。知乎的推送弹了出来：《35 岁程序员真的没有出路了吗？》，3.2K 赞同，892 评论。

他点了进去。

高赞回答来自一个认证为 "前大厂 P8，现独立开发者" 的用户：

"说实话，35岁不是年龄问题，是价值问题。当一个25岁的应届生+AI工具能完成你80%的工作，而薪资只要你的一半时，公司为什么要留你？

我见过太多中年程序员的焦虑：技术跟不上，管理转不了，创业没资源。最后要么降薪去小公司，要么转行卖保险。

但我也见过另一类人：他们不是在写代码，而是在设计系统；不是在使用工具，而是在创造工具；不是在解决问题，而是在定义问题。这类人，永远不会被淘汰。

问题是，你是哪一类？"

李晓明关掉知乎，打开 GitHub。今天的 Trending 又是清一色的 AI 项目：

-   awesome-llm-apps (⭐ 15.2k)
    
-   openai-cookbook (⭐ 58.7k)
    
-   cursor-enhance (⭐ 3.4k)
    
-   replace-developers-with-ai (⭐ 892) 最后一个项目名字格外刺眼。他点进去看了一眼，README的第一句话是："This project aims to automate 90% of junior developer tasks using LLM."
    

初级开发者的90%任务。那中级呢？高级呢？架构师呢？

网约车到了，车窗上映出他的脸。33 岁的脸，眼角已经有了细纹，发际线比五年前后退了两厘米。他记得刚毕业时，满怀激情地写下第一行 "Hello World"。那时候觉得代码能改变世界。

现在，代码本身正在被改变。被 AI 改变。

手机又震了。

林静："还在公司吗？粥在锅里，回来热一下。"

简单的十几个字，却让李晓明鼻子有点发酸。他回复了一个 "马上到家" 的表情。

车开到小区门口，他看到刘洋正在便利店里买红牛。隔着玻璃，两人目光相遇。刘洋举起手里的红牛，露出一个有些尴尬的笑容。

李晓明点了点头，继续往前走。

身后传来刘洋的声音："李哥！等一下！"

他停下脚步，转过身。刘洋小跑过来，呼出的白气在寒风中迅速消散。

"李哥，我看到你的消息了。那个 bug... 对不起，我应该更仔细地 review。"

李晓明看着这个 25 岁的年轻人。North Face 的羽绒服，AirPods Pro 挂在耳朵上，手里拿着 iPhone 15 Pro。典型的大厂新生代程序员形象。月薪 15K，没有房贷压力，未来还有十年。

"不只是 review 的问题，" 李晓明说，"是理解的问题。AI 不理解业务，你也不理解。"

刘洋低下头："我知道。其实... 其实我也发现了，越用 AI，越觉得自己什么都不会。但不用的话，又跟不上进度。"

这话让李晓明愣了一下。

"张总今天找我谈话，" 刘洋继续说，"说我的 AI 使用率是团队最高的，让我下季度带新人，教他们怎么用 AI 工具。可是李哥，我真的配吗？我连一个数据一致性 bug 都看不出来。"

寒风吹过，两人都沉默了。

"回去早点休息，" 李晓明最后说，"明天我们详细过一遍支付模块的设计。"

"谢谢李哥。"

看着刘洋走远的背影，李晓明突然意识到，这个年轻人的焦虑，可能不比他少。

## 3

凌晨 1:30，李晓明推开家门。

客厅的灯还亮着，是林静特意留的。茶几上有张便条，林静娟秀的字迹：

" 粥在锅里，菜在冰箱。

别太晚睡。

爱你。"

最后两个字下面，画了一个歪歪扭扭的笑脸。这是林静的习惯，从恋爱时就有。那时候是在 QQ 上发表情，后来是微信，现在回归到了纸笔。

李晓明走进厨房，掀开锅盖。小米粥还温着，旁边的盘子里是两个茶叶蛋，一碟酱黄瓜。简单，但是用心。

他端着粥碗，走到书房。书架上，技术书占了四排：《深入理解 Java 虚拟机》《微服务架构设计》《分布式系统原理》《高性能 MySQL》... 最上面一排，是这两年新买的：《深度学习入门》《Transformer 架构详解》《大语言模型原理与实践》。

但翻得最多的，还是那本《人月神话》。

1975 年的书，比他的年龄还大。书页已经泛黄，书脊有些开裂。这是他工作第一年，当时的技术总监送给他的。扉页上写着："No Silver Bullet - 老张"。

没有银弹。

他翻开书，书签停在第 16 章，标题就是 "No Silver Bullet"。布鲁克斯写道："没有任何技术或管理上的进展，能够独立地许诺十年内使生产率、可靠性或简洁性获得数量级上的提高。"

这话写于 1986 年。

将近 40 年过去了，AI 出现了。这是银弹吗？

李晓明合上书，打开笔记本。屏幕亮起，VS Code 自动恢复了上次的 session。他新建了一个文件：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-yaml">2025.01.21 凌晨随想今天修了一个 bug，AI 生成的。准确说，是 AI 不理解上下文生成的 bug。但问题是，如果有一天 AI 能理解所有上下文呢？张威说要拥抱变化。陈宇说要创造价值。刘洋说要学会使用。林静说要早点睡觉。他们都对，但他们都不理解。不理解什么？不理解一个 33 岁的程序员，在深夜修复 AI 的 bug 时，那种荒谬感。就像一个修理机器人的机器人。最终，谁修理谁？</code></section>
```

他保存了文件，然后删除了。

卧室的门虚掩着，他轻轻推开。林静侧躺着，月光透过窗帘的缝隙，在她脸上投下柔和的光影。床头柜上，她的手机充着电，旁边是一摞学生的作业本。

李晓明脱下外套，尽量不发出声音。

"回来了？" 林静迷糊地说。

"吵醒你了。"

"没事。" 她翻了个身，"今天学校也在讨论 AI。说以后要用 AI 助教，能自动批改作业。"

"嗯。"

"你说，" 林静睁开眼，"AI 能理解一个孩子为什么把 ' 爸爸 ' 写成 ' 爹爹 ' 吗？能理解他可能是想念在外打工的父亲吗？"

李晓明躺下来，盯着天花板："不能。"

"那就好。" 林静靠过来，把头枕在他肩上，"那我就不会失业。你也不会。"

"为什么？"

"因为你们都在做 AI 不懂的事。你懂代码背后的为什么，就像我懂孩子背后的为什么。"

李晓明没有说话。他想告诉林静，在技术的世界里，"为什么" 正在变得越来越不重要。重要的是 "怎么做"，而这恰恰是 AI 最擅长的。

但他什么都没说，只是抱紧了妻子。

窗外，北京的夜空看不到星星。但偶尔会有飞机闪烁的灯光划过，去往不知什么地方。

## 4

1 月 21 日，周二，9:00 AM。

会议室里，投影仪投射出一条陡峭上升的曲线。横轴是时间，纵轴是 "人效"。这个词是张威从硅谷带回来的，据说是衡量工程师产出的最新标准。

"各位，"张威站在投影前，黑色的Polo衫，卡其色的休闲裤，标准的技术管理层着装，"Q1已经过去三周了。我们的AI工具采用率只有32%。"

他点了一下翻页器，新的slide出现：一个柱状图，每个人的名字下面对应着一个百分比。刘洋，89%。李晓明，12%。

"刘洋做得很好，"张威说，"上周用Cursor重构的订单模块，代码量减少60%，可读性提升40%。这就是AI的力量。"

李晓明想说点什么，但忍住了。他知道那个重构后的模块有多少隐患，但现在不是争论的时候。

"李晓明，" 张威的目光转向他，"你的使用率为什么这么低？"

"我在用，" 李晓明说，"但不是所有场景都适合。"

"比如？"

"比如昨晚的支付模块 bug。"

会议室安静了几秒。

"那是个例，"张威说，"整体而言，AI能覆盖80%的开发场景。剩下的20%，才需要人工介入。问题是，我们需要那么多人来处理20%的场景吗？"

这个问题像一记重锤，砸在每个人心上。

"我举个例子，" 张威打开了一个新的 slide，"支付公司 Klarna 上个月宣布，他们的 AI 助手处理了三分之二的客服对话，相当于 700 名全职员工的工作量，预计每年能提升 4000 万美元的利润。这就是趋势。"

"但客服和开发不一样..." 有人小声说。

"本质是一样的，" 张威打断他，"都是信息处理。区别只是复杂度。而 AI 的能力边界，每天都在扩大。"

他又翻了一页："所以，Q1的KPI很明确。每个人的AI工具使用率必须达到60%以上。达不到的..."

他没有说完，但所有人都懂。

"还有问题吗？"

没人说话。

"那就这样。散会。"

人们陆续离开会议室。李晓明收拾着笔记本，张威走了过来。

"晓明，留一下。"

等其他人都走了，张威关上门。

"我知道你是个好工程师，" 他说，"技术扎实，经验丰富。但这些不够了。"

"什么意思？"

"意思是，你需要进化。不是作为程序员进化，而是作为... 怎么说呢，AI 时代的新物种。"

"新物种？"

"对。不是写代码的人，而是指挥 AI 写代码的人。不是 debug 的人，而是设计系统让 bug 不可能出现的人。"

李晓明看着这个 31 岁的上司。斯坦福的硕士，三年时间从 P6 升到总监，期权价值超过 500 万。他代表着新一代的技术管理者：不迷信技术，只看重效率。

"给你个建议，" 张威说，"陈宇的公司你知道吧？他们在做 AI 编程助手。你可以考虑一下。不是让你跳槽，是让你理解这个行业的方向。"

"你知道陈宇？"

"当然。我还是他们的天使投资人之一。"张威笑了笑，"这个赛道很有前景。当然，风险也很大。90%的创业公司会死掉，但活下来的那10%，会改变整个行业。"

李晓明没有回答。

"Think about it，" 张威拍了拍他的肩膀，用英语说，"The future is already here, it's just not evenly distributed."

未来已经到来，只是分布不均。

威廉・吉布森的名言。李晓明当然知道。

问题是，在这个未来里，他站在哪一边？

会议结束后，李晓明回到工位。

刘洋已经在那里等着了，手里拿着一杯瑞幸咖啡。

"李哥，这是给你的。生椰拿铁，你最喜欢的。"

李晓明接过咖啡："谢谢。坐吧。"

他打开 IDE，调出昨晚的支付模块代码。

"你看这里，" 他指着屏幕，"知道为什么 Copilot 会生成有 bug 的代码吗？"

刘洋摇头。

"因为它学习的是 GitHub 上的开源代码。而大部分开源项目，都是单机场景。分布式事务、并发控制、最终一致性，这些企业级的复杂场景，开源项目很少涉及。"

"所以 AI 不理解？"

"不是不理解，是没见过。就像一个只在动物园里见过老虎的人，不会知道野生老虎有多危险。"

刘洋若有所思。

"但这不是重点，" 李晓明继续，"重点是，你也没见过。你用 AI 的时候，怎么判断它是对是错？"

"我... 我会运行测试。"

"如果测试用例也是 AI 生成的呢？"

刘洋沉默了。

"我不是反对 AI，" 李晓明说，"我是担心我们失去判断的能力。当所有人都依赖 AI 时，谁来判断 AI？"

"那... 我该怎么办？"

李晓明想了想："两条路。第一，深入学习基础知识。分布式系统、数据库原理、操作系统、编译原理。这些东西，AI 可以帮你写代码，但不能帮你理解原理。"

"第二呢？"

"第二，学会提问。不是问 AI 怎么实现，而是问为什么这样实现。不是问 what，而是问 why。"

刘洋点点头，在笔记本上记着什么。

"李哥，" 他突然说，"其实我挺羡慕你的。"

"羡慕我？" 李晓明有些意外。

"嗯。你有自己的判断，知道什么是对的。而我... 我只是个工具的使用者。"

这话让李晓明一时不知道该说什么。

他想起自己 25 岁的时候，也是这样一个工具的使用者。只不过那时的工具是 Eclipse、Maven、Spring。他花了多少年，才从使用者变成理解者？

而现在的年轻人，还有这样的时间吗？

"李哥，" 刘洋又问，"你觉得我们这一代程序员，还有未来吗？"

李晓明看着这个年轻人，看到了八年前的自己。

"有，" 他说，"但不是你想象的那种未来。"

窗外，北京的天空灰蒙蒙的。PM2.5 指数 183，中度污染。

但生活还要继续。代码还要写。bug 还要修。

至于未来？

谁知道呢。

## 第二章 学习曲线

## 1

2025 年 3 月 15 日，周六下午 2 点。

公司培训室里，空调开到了 26 度，有种让人昏昏欲睡的温暖。投影幕布上，一个穿着黑色连帽衫的培训师正在演示 Cursor 的最新功能。他叫 Kevin，据说是从字节跳动挖来的 AI 布道师，时薪 3000。

"看好了，"Kevin 的手指在触控板上优雅地滑动，"我要用 Cursor 在三分钟内完成一个完整的 RESTful API。"

屏幕上，紫色的光标开始闪烁。Kevin 输入了一行注释：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-sql">// Create a user management system with CRUD operations using Express.js and Prisma</code></section>
```

然后按下 Tab 键。

魔法发生了。

代码像瀑布一样倾泻而下。Model 定义、路由配置、数据库连接、错误处理、甚至单元测试，一气呵成。整个过程，Kevin 只按了不到十次 Tab。

"2 分 47 秒，" 他看了眼 Apple Watch，"包括数据库迁移脚本。"

培训室里响起稀稀拉拉的掌声。刘洋鼓得最用力，眼神里充满崇拜。

李晓明坐在最后一排，手里的笔记本上只写了一个词：Context。

"有问题吗？"Kevin 问。

李晓明举起手："这个系统如何处理并发用户注册时的邮箱唯一性校验？"

Kevin 愣了一下："Cursor 会自动添加数据库唯一索引。"

"我是说，在分布式环境下，两个请求同时通过了应用层校验，都准备插入数据库，这时候怎么办？"

"数据库会报错，其中一个会失败。"

"那失败的请求怎么处理？用户体验如何保证？重试策略是什么？"

Kevin 的笑容有些僵硬："这些... 可以让 AI 继续优化。"

"怎么优化？" 李晓明追问，"AI 理解什么是 ' 好的用户体验 ' 吗？"

培训室陷入短暂的沉默。

张威咳嗽了一声："李晓明提的是好问题。Kevin，请继续。"

接下来的一个小时，Kevin 演示了更多 "魔法"：

-   用 Claude 生成整个微服务架构图
    
-   用 Copilot 重构遗留代码
    
-   用 ChatGPT 写技术文档 每一个演示都很流畅，每一个结果都很完美。完美得不真实。
    

休息时间，李晓明走到茶水间。刘洋跟了过来。

"李哥，你为什么总是挑刺？"

李晓明倒了杯咖啡："我不是挑刺，是提醒。"

"提醒什么？"

"提醒你们，魔法是有代价的。"

刘洋不解："什么代价？"

李晓明指了指窗外。北京的春天，柳絮漫天飞舞，像某种失控的代码。

"看到那些柳絮了吗？" 他说，"很美，对吧？但对过敏的人来说，那是灾难。AI 工具也一样，对会用的人是助力，对不会用的人..."

他没有说完。

"是什么？"

"是替代品。"

## 2

下午 4 点，培训进入实战环节。

"现在，我需要一个志愿者，"Kevin 说，"来演示如何用 AI 解决实际问题。"

刘洋立即举手，但 Kevin 的目光越过他，停在李晓明身上。

"李先生，既然你这么有想法，不如你来？"

李晓明站起身，走到台前。

"题目是什么？"

Kevin 打开了一个 GitHub Issue："一个真实的生产问题。某电商平台的订单系统，高峰期出现数据不一致。订单表显示已支付，但支付表没有记录。"

这是个经典的分布式事务问题。李晓明很熟悉。

"请用 AI 工具解决。"Kevin 补充，"你有十分钟。"

李晓明坐到演示电脑前，打开 Cursor。但他没有立即开始写代码，而是打开了一个新文件：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-graphql">## Problem Analysis1. &nbsp;**Symptoms**: Order marked as paid, but no payment record.2. &nbsp;**Possible Causes**:&nbsp; &nbsp; * &nbsp; Network timeout between order service and payment service.&nbsp; &nbsp; * &nbsp; Payment service crash after processing payment but before confirming to order service.&nbsp; &nbsp; * &nbsp; Message queue delivery failure (if&nbsp;using async communication).&nbsp; &nbsp; * &nbsp; Lack of atomicity across service boundaries.3. &nbsp;**Key Question**: How to ensure atomicity or eventual consistency between two separate services?</code></section>
```

"你在做什么？"Kevin 问，"为什么不让 AI 直接生成解决方案？"

"因为 AI 不会问 ' 为什么 '。" 李晓明说。

他继续写：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-graphql">## Context Gathering &amp; Solution DesignBefore generating code, we need to define the non-functional requirements and design constraints.1. &nbsp;**Consistency Model**: Eventual consistency is acceptable, but financial data must be accurate.2. &nbsp;**Chosen Pattern**: Saga pattern with Transactional Outbox&nbsp;for&nbsp;reliable event publishing.3. &nbsp;**Key Mechanisms**:&nbsp; &nbsp; * &nbsp; Idempotent consumers to handle message redelivery.&nbsp; &nbsp; * &nbsp; Compensating transactions to roll back actions&nbsp;in&nbsp;case&nbsp;of failure.&nbsp; &nbsp; * &nbsp; Monitoring and alerting&nbsp;for&nbsp;failed Sagas.</code></section>
```

台下的人开始窃窃私语。这不是他们期待的 "AI 魔法秀"。

"时间过去五分钟了。"Kevin 提醒。

李晓明这才开始与 Cursor 对话：

"Based on the Saga and Transactional Outbox pattern defined above, generate a TypeScript implementation for the Order service. The service should: 1) Create an order in 'PENDING' state. 2) Atomically save the order and an 'OrderCreated' event to its local database using a transactional outbox. 3) A separate process will publish this event. The Payment service will consume this event idempotently."

这一次，Cursor 生成的代码完全不同。不再是简单的 API 调用，而是一个深思熟虑的分布式事务解决方案。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-yaml">// Generated code&nbsp;for&nbsp;the Order Serviceclass OrderSagaCoordinator {&nbsp; constructor(private db: PrismaClient, private outbox: Outbox) {}&nbsp; async createOrder(orderData: any): Promise&lt;Order&gt; {&nbsp; &nbsp;&nbsp;return&nbsp;this.db.$transaction(async (tx) =&gt; {&nbsp; &nbsp; &nbsp; // Step 1: Create the order&nbsp;in&nbsp;PENDING state&nbsp; &nbsp; &nbsp; const order = await tx.order.create({&nbsp; &nbsp; &nbsp; &nbsp; data: { ...orderData, status:&nbsp;'PENDING'&nbsp;},&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp; // Step 2: Write the event to the outbox within the same transaction&nbsp; &nbsp; &nbsp; await this.outbox.add(tx, {&nbsp; &nbsp; &nbsp; &nbsp;&nbsp;type:&nbsp;'OrderCreated',&nbsp; &nbsp; &nbsp; &nbsp; payload: { orderId: order.id, amount: order.amount },&nbsp; &nbsp; &nbsp; });&nbsp; &nbsp; &nbsp;&nbsp;return&nbsp;order;&nbsp; &nbsp; });&nbsp; }}</code></section>
```

"这，"Kevin 有些惊讶，"这确实是个不错的方案。"

"不是 AI 不错，" 李晓明说，"是提问不错。AI 是镜子，映射的是使用者的思维深度。如果你只会问 ' 怎么做 '，它就只会给你代码。如果你问 ' 为什么 ' 和 ' 什么情况下 '，它才会给你解决方案。"

他转向台下："这就是我想说的。AI 工具很强大，但它不能替代思考。它是放大器，放大的是你的能力 —— 如果你的能力是零，放大后还是零。"

培训室里很安静。

刘洋若有所思。张威面无表情。其他人各有心思。

"时间到。"Kevin 说，语气有些尴尬。

李晓明回到座位。他旁边的同事小声说："牛逼，怼得漂亮。"

但李晓明没有觉得得意。他只是觉得累。

不是身体的累，是心理的累。要不断证明自己的价值，要不断对抗潮流，要不断提醒别人思考。

这样的日子，还要持续多久？

## 3

晚上 7 点，五道口，一家叫 "代码与咖啡" 的咖啡馆。

这是程序员的聚集地。墙上贴满了各种技术大会的海报，书架上摆着 O’Reilly 的动物书，背景音乐是 Lofi Hip Hop，据说能提高编程效率。

李晓明坐在角落，对面是陈宇。

"听说你今天在培训上大出风头？" 陈宇笑着说。

"谁告诉你的？"

"刘洋。他在朋友圈发了一条：' 今天见识了什么叫降维打击。'"

李晓明摇摇头："我不是想打击谁，只是..."

"只是看不惯那些 AI 传教士？"

"算是吧。"

陈宇要了两杯咖啡，埃塞俄比亚的耶加雪菲，他最喜欢的豆子。

"其实 Kevin 说的没错，" 陈宇说，"AI 确实是未来。问题是，什么样的未来。"

他打开笔记本，屏幕上是他们公司的产品：AIde（AI + IDE 的双关语）。

"看，这是我们的理念。不是 AI replacing developers，而是 AI empowering developers。"

界面很简洁，像是 Cursor 和 VSCode 的混血儿。但仔细看，有些不同。

"这是什么？" 李晓明指着一个叫 "Wisdom Mode" 的按钮。

"这是我们的核心功能。" 陈宇有些得意，"不只是生成代码，而是解释为什么这样写。每一行代码都有理由，每个设计决策都有依据。"

他演示了一个例子。当 AI 生成一个算法时，旁边会出现一个小窗口：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-sql">Why this approach?- Time Complexity: O(n&nbsp;log&nbsp;n) vs naive O(n²)- Space Trade-off: Uses extra O(n) memory&nbsp;for&nbsp;10x speed- Real-world analogy: Like sorting mail by zip code first- Common pitfalls: Watch out&nbsp;for&nbsp;integer&nbsp;overflow when n &gt; 10^6- Alternative: If memory is limited, consider heap-based approach</code></section>
```

"有意思。" 李晓明承认。

"这还不是全部。" 陈宇切换到另一个界面，"看这个，'Mentor Mode'。"

屏幕上出现一个对话框：

"I notice you're implementing a cache. Have you considered:

1.  Cache invalidation strategy?
    
2.  Memory limits?
    
3.  Concurrent access patterns?
    

Would you like me to explain any of these concepts?"

"它不是替代你思考，" 陈宇说，"而是引导你思考。就像一个永远在线的高级工程师。"

李晓明盯着屏幕，陷入沉思。

"怎么样？" 陈宇问，"有兴趣加入吗？"

"你们有多少人？"

"15 个。7 个工程师，3 个产品，2 个设计，2 个运营，还有我。"

"用户呢？"

"5000 个付费用户。月收入 30 万。离盈利还差一点，但投资人很看好。"

"为什么需要我？"

陈宇放下咖啡杯，认真地看着他："因为你是我认识的人里，最懂 ' 为什么 ' 的人。我们需要的不是写代码的人，而是理解代码背后意义的人。"

李晓明没有立即回答。他看着窗外，五道口的夜晚，霓虹闪烁，年轻人来来往往。这里是中国互联网的起点之一，见证了无数创业梦想的诞生与破灭。

"我考虑考虑。" 最后他说。

"没问题。" 陈宇笑了，"不过提醒你一句，张威可能撑不了多久了。"

"什么意思？"

"你没听说吗？公司在谈新一轮融资，条件之一就是'优化'30%的研发人员。张威的KPI里，有一条就是完成这个目标。"

李晓明的心沉了下去。

"Q2，最迟 Q3，" 陈宇说，"裁员潮就会来。到时候，不是你选择未来，而是未来选择你。"

## 4

晚上 9 点，李晓明到家。

意外的是，林静还没睡，正在餐桌前批改作业。旁边放着她的 iPad，屏幕上是一个叫 "AI 助教" 的应用。

"这是什么？" 李晓明问。

"学校新配的，" 林静说，"说是能自动批改客观题，还能分析学生的错误模式。"

"好用吗？"

"挺神奇的。你看，" 她展示给李晓明看，"它能识别出这个学生总是把 ' 的地得 ' 用错，还给出了针对性的练习建议。"

李晓明看着屏幕上的分析报告，确实很详细。错误类型、频率、可能原因、改进建议，一应俱全。

"但是，" 林静放下 iPad，"它看不出这个孩子为什么最近总是写错字。"

"为什么？"

"因为他爸爸上个月出车祸了，妈妈每天以泪洗面。孩子心不在焉，怎么可能写好字？"

林静拿起红笔，在作业本上写下评语："小明，老师知道你最近很难过。如果需要聊聊，随时来找我。作业可以慢慢改，但要记住，生活总会好起来的。"

李晓明看着妻子，突然想起下午自己说的话：AI 是镜子，映射的是使用者的思维深度。

但林静映射的，不是思维深度，是情感深度。

"你们程序员，" 林静说，"总觉得效率最重要。但有时候，慢一点，笨一点，反而更 human。"

"Human？"

"人性。有温度。" 她合上作业本，"就像你们的代码，如果都让 AI 写，是不是每个程序都会长得一模一样？"

李晓明想了想："可能会。"

"那多无聊啊。" 林静站起身，走到厨房，"我煮了银耳汤，要不要喝？"

"要。"

李晓明坐在餐桌前，打开笔记本。屏幕上，Cursor 的图标在 dock 栏里静静地躺着。旁边是 VS Code、IntelliJ IDEA、Sublime Text... 这些陪伴了他多年的工具。

他打开终端，输入了一行命令：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-shell">$&nbsp;history&nbsp;| grep&nbsp;"git commit"&nbsp;| wc -l12847</code></section>
```

12847 次提交。九年的程序生涯。

每一次提交，都是一个小小的创造。每一行代码，都有他的思考。

这些，AI 能理解吗？

他关上电脑，端起银耳汤。

甜的，温的，有林静的味道。

这就够了。

至少今晚，这就够了。

## 第三章 至暗时刻

## 1

2025 年 5 月 28 日，周三上午 9 点。

会议室里的空气像凝固的水泥。

李晓明坐在靠墙的位置，能清楚地看到每个人的表情。刘洋不停地转着手里的笔，咔哒咔哒的声音在安静的空间里格外刺耳。产品经理赵姐低着头看手机，大概在给孩子的老师发消息。运维的小王靠在椅子上，眼睛盯着天花板的通风口。

张威走进来，手里拿着一个黑色的文件夹。他今天穿了西装，这很反常。平时他都是 Polo 衫配牛仔裤，硅谷风。西装意味着正式，正式意味着坏消息。

"我长话短说。"张威没有坐下，站在投影仪前，"公司新一轮融资的条件之一，是优化组织结构，提升人效。具体到我们部门，需要减员20%。"

20%。技术部35个人，意味着7个人要走。

"名单已经确定。" 张威打开文件夹，"基于 Q1 和 Q2 的绩效评估，以及 AI 工具使用率等多维度考量。"

他翻开第一页。PPT 上出现两个名字：

王磊。赵晓东。

李晓明认识他们。王磊坐在他斜对面，35 岁，去年刚在五环外买了房，首付掏空了双方父母的积蓄。上个月他还在群里晒 B 超照片，说老婆怀孕了，预产期是 11 月。

赵晓东，38 岁，技术专家，但不愿意带团队。两个孩子，大的在上国际学校，一年学费 20 万。小的刚上幼儿园，也是私立的。他常说的一句话是："给孩子最好的教育，是父母的责任。"

"今天下午 HR 会安排离职面谈，" 张威继续说，声音像机器，"补偿是 N+1，这个月底前完成交接。"

王磊的脸瞬间变白。他张了张嘴，似乎想说什么，但最终什么都没说。

赵晓东倒是很平静，平静得可怕。他慢慢合上笔记本，站起身，对张威点了点头，然后走出了会议室。背影笔直，像个战败但不投降的将军。

"其余的调整会在 Q3 继续，" 张威说，"希望大家提高效率，拥抱变化。散会。"

人们陆续离开，没人说话。走廊里只有脚步声，像某种沉闷的鼓点。

李晓明回到工位，看到王磊正在收拾东西。一个马克杯，上面印着 "World's Best Dad"。几本技术书，《Java 并发编程实战》《深入理解 JVM》。一盆多肉，已经有些蔫了。还有那张 B 超照片，用透明胶带贴在显示器边框上。

"老王..." 李晓明想安慰点什么。

王磊抬起头，眼睛红红的："晓明，我上周刚把所有积蓄拿去交了装修首付。"

李晓明沉默了。

"你说讽刺不？"王磊苦笑，"我的AI使用率是32%，刚好在及格线下。如果我多用一点，是不是就能留下？"

"这不是你的错。"

"我知道。" 王磊把 B 超照片小心地放进钱包，"但账单不会管这是谁的错。房贷不会，奶粉钱也不会。"

下午，李晓明经过会议室，看到王磊在里面签离职文件。HR 是个年轻姑娘，机械地念着标准话术："公司感谢您的贡献... 这是竞业协议... 六个月内不得加入竞争对手..."

王磊一言不发地签字，笔尖在纸上沙沙作响。

晚上 7 点，王磊离开了。工位清空了，像从来没有人坐过。只有显示器边框上透明胶带的痕迹，证明这里曾经有过一个父亲的期待。

## 2

当天晚上 10 点，李晓明正准备睡觉，手机突然响了。

"生产环境出大事了！" 是运维小王，声音带着恐慌，"支付系统全线崩溃！"

李晓明一个激灵："什么时候？"

"15 分钟前开始，现在损失已经超过 200 万！"

他套上外套就往外跑。林静迷糊地问："怎么了？"

"公司系统崩了，我得去处理。"

"小心点。"

深夜的北京，路上车很少。李晓明打车直奔公司。10 点 45 分，他冲进作战室。

场面一片混乱。三个运维守在不同的屏幕前，疯狂地敲击键盘。张威来了，头发乱糟糟的，显然也是从床上爬起来的。刘洋坐在角落里，脸色苍白得像纸。

"什么情况？" 李晓明问。

"支付回调大量超时，" 运维主管说，"数据库连接池爆了，Redis 全是脏数据。"

"最近的变更是什么？"

所有人的目光都转向刘洋。

"是... 是我下午上线的代码，" 刘洋声音发抖，"用 Claude 优化的支付模块，本地测试都通过了..."

李晓明走到刘洋的工位，打开代码库。最新的commit：`feat: AI优化支付流程，性能提升300%`。

他快速浏览代码，脑子里的告警响了起来。

"这里，" 他指着屏幕，"你把同步调用改成了异步，但没有考虑幂等性。"

"还有这里，" 他切换到另一个文件，"事务隔离级别用错了，在高并发下会产生幻读。"

"最要命的是这个，" 他打开配置文件，"连接池配置完全是错的。Claude 给你的是单机配置，但我们是分布式部署！"

刘洋的脸更白了："我... 我让 Claude 分析了我们的架构图..."

"架构图不等于运行时环境！" 李晓明没时间解释更多，"让开。"

他接管了键盘。

手指在键盘上飞舞，像钢琴家在演奏最熟悉的曲子。他的大脑高速运转，多年的经验在这一刻全部调动起来。

首先，回滚？不行，已经有脏数据了，直接回滚会造成更大的不一致。

其次，修复数据？太慢，每秒钟都在损失。

那就只有一个办法：热修复。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-graphql"># Step 1: Build and push the hotfix image to the container registrydocker build -t registry.example.com/payment:hotfix-20250528 .docker push registry.example.com/payment:hotfix-20250528# Step 2: Update the deployment to use the new imagekubectl&nbsp;set&nbsp;image deployment/payment-service payment=registry.example.com/payment:hotfix-20250528# Step 3: Monitor the rollout statuskubectl rollout status deployment/payment-service</code></section>
```

他一边操作一边解释："先止血，再治病。必须先推送镜像，否则 K8s 节点拉不到新版本。"

23:17，系统恢复了60%。

23:45，恢复80%。

00:10，完全恢复。

作战室里安静了下来，只有键盘声和服务器风扇的嗡鸣。

"损失统计出来了，" 财务在电话里说，"287 万。"

张威的脸色很难看。他看了看刘洋，又看了看李晓明。

"晓明，" 他说，"还是你靠谱。但问题是，我们不能每次都靠你来救火。"

"我明白。"

"刘洋，" 张威转向那个年轻人，"明天来我办公室。"

刘洋低着头，手在微微发抖。

清晨 5 点，李晓明走出公司大楼。东方已经泛白，早起的鸟儿开始叫了。他很累，不只是身体的累，更是心理的累。

他救了系统，但救不了刘洋。

这个年轻人太依赖 AI 了，依赖到失去了基本的判断力。当 AI 给出答案时，他不会问 "为什么"，只会问 "怎么用"。

这是谁的错？是刘洋的错？是 AI 的错？还是这个急功近利的时代的错？

## 3

5 月 29 日晚上 10 点，李晓明刚处理完善后事宜，手机又响了。

是陈宇。

"老李，" 声音很沉重，"我有个坏消息。"

李晓明心里一沉："怎么了？"

"资金链断了。投资人撤资了。"

"为什么？"

"他们说 AI 编程助手赛道太拥挤，大厂都在做，我们没有竞争优势。" 陈宇苦笑，"你知道最讽刺的是什么吗？"

"什么？"

"他们用ChatGPT分析了我们的商业计划书，得出的结论是：成功概率不足15%。"

AI 判了 AI 创业公司的死刑。这确实够讽刺的。

"你打算怎么办？"

"还能怎么办？遣散团队，把剩下的钱还给投资人。" 陈宇停顿了一下，"老李，我之前说的 offer，作废了。对不起。"

"没事，我理解。"

"理解个屁！" 陈宇突然爆发，"我他妈的不理解！我们明明有好产品，有用户，有口碑，为什么就是活不下去？"

李晓明没有回答。因为他也不理解。

挂了电话，他坐在书房里发呆。桌上的《人月神话》还翻开着，旁边是空了的咖啡杯。

林静推门进来："还不睡？"

"陈宇的公司倒了。"

"哦。" 林静坐到他旁边，"其实我今天也有个消息要告诉你。"

"什么？"

"爸爸的体检报告出来了，肺部有阴影，医生建议进一步检查。"

李晓明的心又沉了一截："严重吗？"

"不知道。但如果需要手术的话..." 她停顿了一下，"医生说要准备 30 万。"

30 万。

李晓明打开手机银行。余额：15 万 3 千。其中 10 万是下个月要还的房贷。

"别担心，" 他握住林静的手，"会有办法的。"

"嗯。" 林静靠在他肩上，"其实我不担心钱，我担心你。你最近压力太大了。"

压力。

35 岁的焦虑，AI 的威胁，同事被裁，系统崩溃，朋友创业失败，岳父生病...

压力像潮水，一波接一波，快要把他淹没了。

## 4

5 月 30 日凌晨 4 点。

李晓明睡不着，走到阳台上。

北京的夜空永远看不见星星，只有远处楼宇的航标灯在闪烁。他从冲锋衣口袋里摸出一包烟，是两年前的，一直没抽完。

点上一根。

烟雾缭绕，像他混乱的思绪。

他想起九年前，刚毕业时的自己。那时候觉得代码能改变世界，觉得技术能解决一切问题。现在呢？技术本身成了问题。

手机震动了一下。知乎推送：《被 AI 取代的程序员，后来都去哪了？》

他点进去，最高赞的回答是：

" 讲个真事。我前同事，37 岁，大厂 P7，去年被优化了。找了三个月工作，要么是薪资砍半，要么是要求必须熟练使用 AI 工具。最后他去送外卖了。

他说，至少送外卖的时候，他知道自己在创造价值 —— 把热乎乎的饭送到饥饿的人手里。而不是像以前那样，写一堆可能第二天就被 AI 重写的代码。"

李晓明关掉知乎，打开了 ChatGPT。

他输入："你会取代程序员吗？"

ChatGPT 回答：" 我不会 ' 取代 ' 程序员，而是 augment（增强）程序员的能力。编程的本质不仅是写代码，更是理解问题、设计解决方案、做出权衡决策。这些需要人类的创造力、判断力和同理心。

我可以帮助处理重复性任务，生成样板代码，提供建议，但我无法：

-   理解业务的真实需求
    
-   做出伦理和价值判断
    
-   处理模糊和矛盾的需求
    
-   创造真正创新的解决方案 未来的程序员，不是写代码的人，而是设计系统、定义问题、指导 AI 的人。"
    

李晓明盯着这段话，陷入沉思。

"那么，" 他继续输入，"一个 33 岁的程序员，如何转型？"

ChatGPT 回答："33 岁正是转型的黄金时期。你有足够的经验理解技术的本质，又有足够的精力学习新东西。建议：

1.  从执行者转向设计者
    
2.  从编码者转向架构师
    
3.  从个人贡献者转向团队赋能者
    
4.  从技术专家转向领域专家
    

记住：AI 时代需要的不是 'AI vs 人类 '，而是 'AI + 人类 '。找到你与 AI 协作的最佳模式。"

与 AI 协作。

这个词让李晓明想起了什么。

他打开笔记本，新建了一个文档：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-bash">人机协作开发模式 v0.1核心理念：- 人类定义 What（做什么）和 Why（为什么）- AI 执行 How（怎么做）- 人类验证 Whether（是否正确）角色分工：- 人类：系统设计师、质量守门员、价值定义者- AI：代码生成器、模式识别器、效率加速器实施步骤：1. 人类设计系统架构和核心抽象2. AI 生成具体实现代码3. 人类 review 和修正关键逻辑4. AI 处理测试和文档5. 人类做最终决策和部署价值主张：不是用 AI 取代人，而是用&nbsp;"人 + AI"&nbsp;取代&nbsp;"只会用 AI 的人"</code></section>
```

他盯着屏幕，第一次感觉到了某种可能性。

不是对抗，不是逃避，而是共生。

太阳出来了，阳光洒在阳台上。

新的一天开始了。

或许，也是新的开始。

## 第四章 进化之路

## 1

2025 年 7 月 30 日，周二上午 10 点。

李晓明站在张威办公室门口，深吸了一口气。

过去两个月，他几乎没怎么睡好。白天处理岳父的医疗事宜，晚上研究人机协作模式。林静说他瘦了五斤，眼窝都凹进去了。

但他从未如此清醒。

敲门。

"进。"

张威正在看三个显示器，左边是代码，中间是数据报表，右边是飞书。典型的技术管理者工作状态。

"有事？" 张威没有抬头。

"我有个提案。" 李晓明说。

这句话让张威停下了手里的工作。他转过椅子，第一次认真地看着李晓明。

"说。"

李晓明打开笔记本，连上会议室的显示器。屏幕上出现四个大字：人机协作。

"这是什么？" 张威皱眉。

"这是我们的未来。" 李晓明说，"不是 AI 取代人类，也不是人类对抗 AI，而是人类与 AI 的协作。"

他切换到下一页。一个金字塔图形出现在屏幕上。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-markdown">价值定义／＼／Why＼／ ＼／ What&nbsp;if&nbsp;＼／ ＼／ What ＼／ ＼／ How ＼／________________＼执行实现</code></section>
```

"金字塔顶端是人类，" 李晓明解释，"负责定义价值、理解为什么、预见风险。底层是 AI，负责执行、实现、优化。"

"理论谁都会说，" 张威说，"关键是怎么落地。"

李晓明切换到实例页面："还记得两个月前的支付系统崩溃吗？"

张威的脸色变了变："当然记得。287 万的损失。"

"问题的根源是什么？"

"刘洋用 AI 生成的代码有 bug。"

"不，" 李晓明摇头，"根源是我们让 AI 做了它不该做的事 —— 理解业务上下文。同时，我们让人做了人不擅长的事 —— 大量重复编码。"

他打开一个架构图："这是我设计的新模式。"

屏幕上，系统被分成了四层：

1.  **意图层（Intent）**：人类定义要解决什么问题
    
2.  **设计层（Design）**：人类设计系统架构和核心抽象
    
3.  **实现层（Implementation）**：AI生成具体代码
    
4.  **验证层（Verification）**：人类验证关键路径，AI处理常规测试
    

"举个例子，" 李晓明打开 IDE，"如果我们要实现一个新的支付功能。"

他开始演示：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-ruby">// 人类定义意图（Intent）/**&nbsp;* @intent&nbsp;* 需求：实现一个支付拆分功能&nbsp;* 场景：用户使用多种支付方式（余额+银行卡+优惠券）完成一笔订单&nbsp;* 约束：&nbsp;* 1. 原子性：要么全部成功，要么全部失败&nbsp;* 2. 幂等性：重复请求不会重复扣款&nbsp;* 3. 可追溯：每一步都要有日志&nbsp;*/// 人类设计核心抽象（Design）abstract class PaymentSplitter {&nbsp; /**&nbsp; &nbsp;* @design&nbsp; &nbsp;* 核心设计决策：&nbsp; &nbsp;* 1. 使用Saga模式处理分布式事务&nbsp; &nbsp;* 2. 每个支付方式是一个独立的Saga步骤&nbsp; &nbsp;* 3. 失败时反向补偿&nbsp; &nbsp;*/&nbsp; public abstract splitPayment(order: Order, methods: PaymentMethod[]): Promise&lt;void&gt;;&nbsp; // AI should generate the implementation&nbsp;for&nbsp;concrete classes...}</code></section>
```

"看到了吗？" 李晓明说，"人类负责 ' 什么 ' 和 ' 为什么 '，AI 负责 ' 怎么做 '。人类是建筑师，AI 是建筑工人。"

张威若有所思："继续。"

李晓明切换到另一个页面："更重要的是，这种模式下，我们可以建立 ' 防护栏 '。"

```
<section><span leaf=""><br></span></section><section><code id="code-lang-perl">// 防护栏示例@AIGuardRail({&nbsp; preConditions: [(amount) =&gt; amount &gt; 0 &amp;&amp; amount &lt; 100000, //amount 必须在合理范围内(userId) =&gt; userService.exists (userId), //user_id 必须存在且状态正常&nbsp; ],&nbsp; postConditions: [(result) =&gt; result.status ===&nbsp;'SUCCESS', // 必须有明确的成功状态&nbsp; ],&nbsp; invariants: ['no_concurrent_payment_for_same_user'&nbsp;// 不允许并发处理同一用户的支付&nbsp; ]})async&nbsp;function&nbsp;processPayment(amount: number, userId: string): Promise&lt;PaymentResult&gt; {&nbsp; // AI-generated code will be placed here and automatically checked&nbsp; // against the guardrail rules before and after execution.}</code></section>
```

"这样，即使 AI 生成了有问题的代码，也会被防护栏拦截。" 李晓明说。

张威站起来，走到窗前，看着外面的 CBD。过了一会儿，他转过身：

"你需要什么？"

"一个小团队，两个月时间，做一个 POC（概念验证）。"

"如果失败了呢？"

李晓明想了想："那至少我们试过了。总比坐以待毙强。"

张威笑了，这是李晓明第一次看到他真心的笑容。

"好，批准了。你可以选两个人。"

"刘洋，" 李晓明说，"还有新来的实习生小陈。"

"刘洋？" 张威很意外，"他差点被开除。"

"正因为他犯过错，所以他知道 AI 的局限。而且，他需要一个机会。"

## 2

8 月 1 日，新项目启动。

李晓明给团队取了个名字：HAC（Human-AI Collaboration）。办公室在 12 楼的一个小会议室，临时改造的。三张桌子，三台电脑，一块白板。

刘洋第一天来的时候，带了一箱红牛。

"李哥，" 他说，"谢谢你。"

"谢什么？"

"给我机会。" 刘洋的眼睛有些红，"自从上次事故后，没人愿意跟我合作了。"

"那是因为他们不理解失败的价值，" 李晓明说，"你知道爱迪生发明灯泡失败了多少次吗？"

"一千次？"

"他说：' 我没有失败，我只是发现了一千种不能做灯泡的方法。' 你的失败，让我们发现了 AI 不能做什么。这很宝贵。"

实习生小陈是个 00 后，清华计算机系的，很聪明但也很安静。他负责构建测试框架。

第一周，他们设计系统架构。

白板上画满了图：

```
<section><span leaf=""><br></span></section><section><code id="code-lang-java">Human Layer (Architects)↓ Intent &amp; ConstraintsDesign Layer (Patterns)↓ Abstractions &amp; InterfacesAI Layer (Generators)↓ Code &amp; TestsValidation Layer (Guards)↓ Safety &amp; QualityProduction Layer (Deployment)</code></section>
```

第二周，开始编码。

不，准确地说，是开始 "编排"。他们写的不是代码，而是 "意图描述" 和 "约束规则"。

```
<section><span leaf=""><br></span></section><section><code id="code-lang-vbnet">intent:&nbsp; name: UserAuthenticationServicepurpose: 安全地验证用户身份constraints:- 密码必须加密存储（bcrypt, min_rounds: 10）- 登录失败超过 5 次锁定账户- Session 必须有超时机制（30 分钟）- 所有认证操作必须记录日志quality_gates:&nbsp; - code_coverage:&nbsp;"&gt;= 80%"&nbsp; - security_scan:&nbsp;"no critical issues"&nbsp; - performance:&nbsp;"&lt; 100ms response time"ai_instructions:&nbsp; model:&nbsp;"gpt-4"&nbsp; style:&nbsp;"clean, defensive, well-commented"&nbsp; patterns: ["singleton",&nbsp;"factory",&nbsp;"observer"]</code></section>
```

基于这些描述，AI 生成了完整的认证服务。但这不是结束，而是开始。

"看这里，" 刘洋指着屏幕，"AI 生成的加密方法有问题。它用的是 SHA256，不是 bcrypt。"

"为什么？" 小陈问。

李晓明看了一眼："因为 GitHub 上大部分老代码都用 SHA256。AI 学的是历史，不是最佳实践。"

他们手动修正了这个问题，并把它加入了 "防护栏" 规则库。

第三周，系统初具规模。

他们开发了一个完整的电商后台，包括用户管理、商品管理、订单处理、支付集成。总共 15000 行代码，其中：

-   人类写的：2000 行（核心抽象和约束）
    
-   AI 生成的：13000 行（具体实现）
    
-   生成时间：4 小时
    
-   调试修正时间：2 天 "效率提升了 10 倍，" 刘洋兴奋地说。
    

"不只是效率，"李晓明指着测试报告，"看质量。零严重bug，测试覆盖率92%。"

但真正的考验还在后面。

## 3

8 月 15 日，周三下午 2 点。

CEO 办公室。

这是李晓明第二次来这里。第一次是入职的时候，九年前。那时候 CEO 还很年轻，公司只有 50 个人。现在，CEO 头发花白了，公司有 3000 人。

会议室里除了 CEO，还有 CTO、CFO，以及几个 VP。张威也在，坐在角落里。

"我听说你有个有意思的项目。"CEO 说。

李晓明打开笔记本："Human-AI Collaboration，人机协作开发模式。"

"又是 AI，"CFO 皱眉，"我们已经在 AI 工具上投入了几百万，效果并不理想。"

"因为我们用错了方式，" 李晓明说，"我们试图用 AI 替代人，而不是增强人。"

他开始演示。

屏幕分成两部分。左边是传统的纯 AI 开发，右边是 HAC 模式。

"同样的需求：开发一个实时数据分析系统。"

两边同时开始。

左边：开发者输入 prompt，AI 生成代码，看起来很完美。

右边：开发者先设计架构，定义约束，然后让 AI 生成实现。

"现在，我们加入一个变化。" 李晓明说，"数据源突然增加了 10 倍。"

左边的系统崩溃了。内存溢出，数据库连接池爆满。

右边的系统自动降级。因为在设计时就考虑了弹性伸缩。

"再来一个变化，" 李晓明继续，"新的隐私法规要求删除某些敏感数据。"

左边：需要人工寻找所有相关代码并修改，耗时 3 小时。

右边：修改一个约束规则，AI 自动重新生成合规代码，耗时 5 分钟。

"这就是区别，" 李晓明总结，"左边是 'AI 做事，人类修 bug'。右边是 ' 人类设计，AI 执行，系统自愈 '。"

CEO 若有所思："成本呢？"

CFO拿出计算器："按照演示的效率，人力成本可以降低40%，但质量提升60%。投资回报率...很可观。"

"风险呢？"CTO 问。

"主要风险是思维转变，" 李晓明坦诚地说，"工程师需要从 ' 写代码的人 ' 转变为 ' 设计系统的人 '。这需要培训和时间。"

"如果我们不做这个转变呢？"CEO 问。

张威站起来："那我们就会被那些做了转变的公司淘汰。"

会议室安静了。

CEO 看着窗外，CBD 的天际线在夕阳下闪闪发光。每一栋楼里，都有公司在做相同的抉择：拥抱变化，还是固守传统。

"李晓明，"CEO 转过身，"你愿意负责这个转型吗？"

李晓明愣了一下："我？"

"对。新职位：AI 协作架构师。负责在全公司推广这个模式。"

"我..." 李晓明看了看张威。

张威点点头："这是你应得的。"

## 4

8 月 20 日，周四晚上 7 点。

李晓明准时关上电脑。这是三个月来，他第一次不加班。

走出公司大楼，夕阳正好。他没有叫车，而是选择走路回家。

路过那家 "代码与咖啡" 咖啡馆，他看到陈宇坐在里面，面前放着笔记本，正在写什么。

李晓明推门进去。

"老李！" 陈宇很惊喜，"好久不见。"

"在忙什么？"

"写总结。创业失败的 100 个教训。" 陈宇自嘲地笑，"下次创业可以避免 99 个。"

"还有下次？"

"当然。失败是成功之母嘛。" 陈宇合上笔记本，"听说你升职了？"

"算是吧。"

"AI 协作架构师，" 陈宇念着这个 title，"很酷。"

"不只是 title。" 李晓明说，"是一种新的可能。"

他给陈宇讲了 HAC 项目，讲了人机协作的理念，讲了未来的规划。

"你知道吗，" 陈宇听完后说，"这正是我当初想做的。只是我太理想化了，想一步到位。而你，找到了渐进的路径。"

"要不要一起？" 李晓明问，"公司在招人。"

陈宇摇摇头："谢谢，但我想再试试创业。这次，我要做教育。"

"教育？"

"对，教人们如何与 AI 协作。不是技术培训，而是思维培训。"

两个老友碰了碰咖啡杯。

晚上 8 点，李晓明到家。

林静正在准备晚餐，肚子已经明显隆起。预产期是 12 月，还有四个月。

"今天怎么这么早？" 她笑着问。

"以后都会这么早。" 李晓明说。

他从包里拿出一个文件夹："新的 offer，签了。"

林静接过来看："月薪...45000？"

"AI 协作架构师的市场价。" 李晓明说，"而且没有 996，正常上下班。"

"太好了！" 林静抱住他，"我们可以换个大点的房子了。"

"不急，" 李晓明说，"我看中了公司附近的一个小区。不是学区房，但环境很好，离公司近，我可以每天回家吃晚饭。"

"学区房不重要吗？"

"重要。但更重要的是，" 李晓明摸了摸林静的肚子，"孩子能每天见到爸爸。"

林静的眼睛湿润了。

晚餐很简单。西红柿鸡蛋面，李晓明最喜欢的。他们聊着未来，聊着孩子的名字，聊着新家的装修。

"对了，" 林静突然说，"今天学校 AI 助教系统升级了，加入了情感识别功能。"

"有用吗？"

"还是不行。它能识别出孩子在哭，但不知道是因为想妈妈，还是因为被同学欺负。"

"这就是 AI 的局限。" 李晓明说。

"所以老师不会被取代？"

"不会。就像程序员不会被取代。改变的只是我们的角色。"

林静想了想："从教书变成育人？"

"从写代码变成设计未来。"

他们相视而笑。

窗外，北京的夜景璀璨。这个城市从不睡觉，总有人在奋斗，总有人在焦虑，总有人在迷茫。

但李晓明不再焦虑了。

他找到了自己的位置，不是在 AI 之下，不是在 AI 之上，而是在 AI 身边。

这就够了。

## 尾声

三个月后，2025 年 11 月 15 日。

李晓明站在公司年会的舞台上，台下坐着 500 多名工程师。

"大家好，我是李晓明。九个月前，我是一个焦虑的 33 岁程序员，担心被 AI 取代。今天，我想分享一个故事。"

他打开 PPT，第一页是一张照片：一个骑手和一匹马。

"这是人类和马的关系。几千年来，马帮助人类运输、耕作、战斗。汽车出现后，马没有消失，而是角色改变了。它们成为了运动、娱乐、治疗的伙伴。"

他切换到下一页：一个程序员和一个 AI 助手的图标。

"这就是我们和 AI 的未来。不是替代，而是进化。不是人 vs 机器，而是人 with 机器。"

"有人问我，程序员的未来在哪里？"

"我的答案是：未来不在于你会写多少代码，而在于你能创造多少价值。不在于你懂多少技术，而在于你理解多少人性。"

"AI 很强大，它能写代码，能优化算法，能处理数据。但它不能理解一个用户为什么在凌晨 3 点还在使用我们的产品。它不能感受一个创业者的梦想。它不能体会一个父亲想给孩子更好生活的心情。"

"这些，只有人能理解。"

"所以，不要恐惧 AI，拥抱它。让它成为你的伙伴，而不是对手。"

"记住，在 AI 时代，最稀缺的不是会编程的人，而是懂得如何让技术服务于人的人。"

"谢谢大家。"

掌声雷动。

台下，刘洋用力鼓掌，眼睛有些湿润。

张威站在后排，露出欣慰的笑容。

而在医院里，林静正在待产室。她看着手机直播，对肚子里的宝宝说："看，这是爸爸。他很厉害对不对？"

一个月后，李晓明的儿子出生了。

取名李知新。

知是知道的知，新是创新的新。

在这个 AI 的时代，愿他既知道过去，又能创造未来。

就像他的父亲一样。

说明

这篇小说，由本人指导提示 AI 生成，之后用另外的 AI 进行审核润色输出。提示词与流程设计为本人原创。用户初始输入的主题为：

2025年，AI冲击下，一个33岁程序员

**点赞 +「在看」，转发**给你身边有需要的朋友。收不到推送?那是因为你只**订阅**，却没有**加星标**。

欢迎订阅我的小报童付费专栏，每月更新不少于3篇文章。订阅一整年价格优惠。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果有问题咨询，或者希望加入社群和热爱钻研的小伙伴们一起讨论，**订阅知识星球**吧。不仅包括小报童的推送内容，还可以自由发帖与提问。之前已经积累下的帖子和问答，就有数百篇。足够你好好翻一阵子。知识星球支持72小时内无条件退款，所以你可以放心尝试。

![Image](data:image/svg+xml,%3C%3Fxml version='1.0' encoding='UTF-8'%3F%3E%3Csvg width='1px' height='1px' viewBox='0 0 1 1' version='1.1' xmlns='http://www.w3.org/2000/svg' xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg stroke='none' stroke-width='1' fill='none' fill-rule='evenodd' fill-opacity='0'%3E%3Cg transform='translate(-249.000000, -126.000000)' fill='%23FFFFFF'%3E%3Crect x='249' y='126' width='1' height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

若文中部分链接可能无法正常显示与跳转，可能是因为微信公众平台的外链限制。如需访问，请点击文末「阅读原文」链接，查看链接齐备的版本。

## 延伸阅读

-   [看着星友 3 分钟抢光 AI 调研工具内测名额，我突然明白了什么叫生产力革命](https://mp.weixin.qq.com/s?__biz=MzIyODI1MzYyNA==&mid=2653549207&idx=1&sn=800de3fbdd105bbc82d8e25b6967ada8&scene=21#wechat_redirect)
    
-   [ChatGPT 时代，我的新书《智慧共生》上市了](https://mp.weixin.qq.com/s?__biz=MzIyODI1MzYyNA==&mid=2653544335&idx=1&sn=a630c816baae41e54e8558154e694b90&scene=21#wechat_redirect)
    
-   [人工智能绘图应用 DALLE 2 开始公开测试了](https://mp.weixin.qq.com/s?__biz=MzIyODI1MzYyNA==&mid=2653543253&idx=1&sn=b3605fcb37fb914d6fc7e96fd21396cb&scene=21#wechat_redirect)
    
-   [AI 真要成精了？ChatGPT 上手体验](https://mp.weixin.qq.com/s?__biz=MzIyODI1MzYyNA==&mid=2653543718&idx=1&sn=aa4f795124b7ca2036148551e553278d&scene=21#wechat_redirect)
    
-   [如何用 AI 给科研提速？超长对话记忆 Kimi Chat 体验](https://mp.weixin.qq.com/s?__biz=MzIyODI1MzYyNA==&mid=2653545366&idx=1&sn=3424e31694fa6f2e688f649727beda5f&scene=21#wechat_redirect)