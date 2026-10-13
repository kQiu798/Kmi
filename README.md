# Kmi

核心功能(实现得相当完整)
1.  双模型切换 —— Flash / Pro 通过两个按钮切 MODEL_MAP,modelType 持久化到 localStorage。
2.  深度思考 —— thinking: {type} + reasoning_effort 三档(Low / High / Max),思考过程用可折叠块实时流式展示(ChatView.vue:82-92)。
3.  联网搜索 —— Tavily 搜索 -> 结果作为独立 user 消息注入(ChatView.vue:1210-1219),刻意不塞进 system prompt 以规避 DeepSeek 内容过滤,有注释说明,是有意为之的设计。
4.  文件上传 —— 图片走 image_url 多模态,文本文件内联;图片超 2048px 用 canvas 压缩;15MB / 5 个文件限制。
5.  消息版本管理 —— 重新生成时保留历史版本,支持左右箭头切换(versions[] + currentVersion)。
6.  热点推荐 —— Tavily 搜当天热点 -> Flash 模型精简到 25 字以内 -> 缓存 24 小时。
7.  互斥开关 —— 深度思考 / 联网搜索 / 猫娘强化三者互斥。
8.  响应式 —— 600 / 480 / 360px 三档断点,移动端隐藏头像、操作按钮常显。
