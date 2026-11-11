# 2026-11-10：<用atomic+dp跑crossword任务>

## 做了什么

今天跑了用了recovery数据训练的dp模型做crossword任务，做MARS，用gemini和astra都跑了。

## 观察到了什么

recovery数据可能会把模型训练坏，采集recovery的时候不能主动示范错误动作，要从错误动作结束阶段做起，示范正确的动作。不然在deploy的时候机器人会主动做错误的动作。



vlm当作checker会有和policy执行和molmopoint打点的频率互斥问题。比如每当机械臂做一个place动作的时候，动作做到一半，还没做完，vlm checker就打断进行check,导致机械臂未回到home状态，遮挡了目标方块，vlm check三次后没有找到point便自动退出程序。



dp对相机图像的要求很高，我们的外部相机放置有点歪，往右侧倾斜，所以当目标物体在右侧的时候，point的像素点位置准确特别重要，如果偏移了一点对于policy来说，成功率就会下降很多。并且我们目前的方法对于腕部相机的使用不高，这可能是训练的问题，我们把腕部相机的point embedding向量长度增长到512了（之前是128）dp的效果还是不好，腕部相机对于policy影响依然远不如external camera.并且dp训练后有很强的动作惯性，或者是overfitting,稍微动一下方格摆放位置效果可能差别很大；泛化性不好，训练是用的四个字母，deploy加一个字母就放不了了，第五个字母还是会当作第四个字母放。



gemini总是断触真是服了，astra好很多，并且astra会replan,会调整phase,gemini基本不会replan.

## 我的理解

哪些判断有证据，哪些只是猜测；假设是否得到支持，还有什么无法解释。

