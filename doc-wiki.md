### 课堂页直播录音相关逻辑
1. 涉及的api
- wx.startRecord()
- wx.stopRecord()
- wx.onVoiceRecordEnd()
- wx.uploadVoice()
2. 录音简单逻辑（连续录音）
- 录音开始，事件绑定，触发startRecord
- 由AudioRecorder监听当前录音状态，如果当前录音消息时长超过58秒（微信api规定最长录音时间是60秒），由WeixinApi的onVoiceRecordEnd监听录音时长，自动断开
- 这时候需要标记当前录音文件status为force_stop, 拿到localId，随后，AudioRecorder会把当前录音标记为发送状态，这时候交由UploadQueue队列中，并开启下一次录音进程
- 录音完成的文件的逻辑在UploadQueue里面进行，这时候会进行两个步骤，一是把当前录音文件加入课堂消息列表中，二是调用wx.uploadVoice接口上传当前录音文件到微信服务器，获取serverId
- 在课堂消息流组件ClassroomMessageList里面，主要是处理wx.uploadVoice成功回调的录音文件，会监听标记为send状态的消息，然后调用/message/send的后端接口传给后端(这里也会把标记为failed的消息记录到本地local，然后下次进入课堂页，会遍历失败消息，重新请求/message/send的接口传过去)
3. 注意点
- onVoiceRecordEnd: 录音时间超过一分钟没有停止的时候，中途录音中断都会执行 complete 回调
- uploadVoice: UploadQueue组件是处理录音上传逻辑，录音上传进程就是拿localId换取serverId的过程，在调用wx.uploadVoice的过程有小概率出现错误，需要做最长时间（这里是10秒）等待，超过十秒后需要对当前录音做失败标记，不能卡在一条录音上；如果出现新进入的语音，然后上一条录音还没处理完毕，需要标记queued进入队列排队等待。