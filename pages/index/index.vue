<template>
	<view>
		<view class="uni-padding-wrap uni-common-mt">
			<view class="uni-btn-v">
				<button type="primary" :disabled="!canConnect" @click="connect">{{!isConnected ?'connnect':'disconnect' }}</button>
			</view>
			<view class="text-box" scroll-y="true">
				<text>{{text}}</text>
			</view>
		    <view class="uni-form-item uni-column">
		        <!-- <view class="title"><text class="uni-form-item__title">聊天内容输入框</text></view> -->
		        <input class="uni-input" placeholder-style="color:#F76260" placeholder="请输入聊天内容" :value="inputContentValue" @input="clearInput" />
				<!-- <view class="uni-input-wrapper"> -->
		            <!-- <input class="uni-input" placeholder="" :value="inputContentValue" @input="clearInput" /> -->
		            <!-- <text class="uni-icon" v-if="showClearIcon" @click="clearIcon">&#xe434;</text> -->
		        <!-- </view> -->
		    </view> 
			<view class="uni-btn-v">
				<button type="primary" :disabled="!canChat" @click="chat">send</button>
				<!-- <button type="warn" :disabled="!canTalk" @click="talk">talk</button> -->
			</view>
		</view>
	</view>
</template>
<script>
	export default {
		data() {
			return {
				title: 'text',
				texts: [
				],
				text: '',
				canConnect: true,
				canChat: false,
				canTalk: false,
				extraLine: [],
				isConnected: false,
				inputContentValue: '',
                showClearIcon: false,				
			}
		},
		methods: {
			//连接websocket
			connectWebSocket() {
				let that = this;
				console.log('调用连接websocket')
				this.socketTask = uni.connectSocket({
						url: 'ws://124.70.89.172:9527',
						success(res) {
							console.log("WebSocket连接成功");
							uni.showToast({
								title: 'WebSocket连接成功',
								icon: "none"
							});
							that.isConnected = true;
							// this.canConnect = false;
						},
						fail(err) {
							console.log("报错", err);
						}
					},
				);
				this.socketTask.onOpen(function(res) {
					console.log('WebSocket连接已打开！');
					that.isConnected = true;
					// that.canConnect = false;
					let obj = {};
					obj.request_id = that.guid();
					obj.format = "utf-8";
					obj.timestamp = 1620358489;
					obj.version = 1;
					obj.command = "connect";
					let data = {};
					data.uuid = "123456";
					obj.data = data;
					//console.log(obj);
					let content = JSON.stringify(obj);
					//console.log(content);
					that.sendWebSocketMessage(content);
				})
				this.socketTask.onMessage(function(res) {
					console.log('收到服务器内容：' + res.data);
					that.handlerMsg(JSON.parse(res.data)) //这里是对获取到的数据进行操作
				});
				this.socketTask.onError(function(res) {
					console.log('WebSocket连接打开失败，请检查！');
					console.log(res);
					that.isConnected = false
					// that.canConnect = true;
					//进入重新连接
					// that.reconnectWebSocket();
				})
				// // 监听连接关闭 -
				this.socketTask.onClose((e) => {
					console.log('WebSocket连接关闭！');
					clearInterval(that.timer)
					that.timer = ''
					if (!that.isClose) {
						// that.reconnectWebSocket()
					}
				})
				console.log(this.socketTask)
			},
			
			//进入重新连接
			reconnectWebSocket() {
				console.log('进入断线重连');
				// this.socketTask.close();
				this.socketTask = null;
				this.connectWebSocket()
			},

			//发送消息
			sendWebSocketMessage(msg) {
				console.log('发送信息')
				console.log(msg)
				return new Promise((reslove, reject) => {
					this.socketTask.send({
						data: msg,
						success(res) {
							console.log('发送成功')
							reslove(res)
						},
						fail(res) {
							console.log('发送失败')
							console.log(res)
							reject(res)
						}
					});
				})
			},
						
			//心跳
			heart() {
				let that = this;
				clearInterval(this.timer);
				this.timer = '';
				let msg = {
					"type": "heartbeat",
				}
				this.timer = setInterval(() => {
					 that.sendSocketMessage(JSON.stringify(msg)).then(res => {
						console.log('心跳成功')
					}).catch(res => {
						console.log('发送失败')
					console.log((res))
					 })
				}, 200000)
			},
			
			// 主动关闭socket连接
			closeWebSocket() {
				this.socketTask.close({
					success() {
						uni.showToast({
							title: 'WebSocket关闭成功',
							icon: "none"
						});
					}
				});
			},
				
			//处理接收消息
			handlerMsg(payload) {
				//console.log('进入处理接收消息');
				console.log(payload);
				let request_id = payload.request_id;
				let data = payload.data;
				let command = data.command;
				if (command == 'connect') {
					this.extraLine.pop();
					this.extraLine.push('连接中...连接成功');
					this.text = this.extraLine.join('\n');
					this.canChat = true;
					this.canTalk = true;
				}
				else if (command == 'chat') {
					let type = data.type;
					if (type == 'text') {
						// 显示文本
						let content = data.content;
						this.extraLine.push('机器人：' + content);
						this.text = this.extraLine.join('\n');
						this.texts.push('AI:' + content);
					} else if (type == 'audio') {
						// 播放语音
						let content = data.content;
						const innerAudioContext = uni.createInnerAudioContext();
						innerAudioContext.autoplay = true;
						// innerAudioContext.src = 'https://bjetxgzv.cdn.bspapp.com/VKCEYUGU-hello-uniapp/2cc220e0-c27a-11ea-9dfb-6da8e309e0d8.mp3';
						innerAudioContext.src = content;
						innerAudioContext.onPlay(() => {
						  console.log('开始播放');
						});
						innerAudioContext.onError((res) => {
						  console.log(res.errMsg);
						  console.log(res.errCode);
						});
					}
				}
			},
			
			connect: function(e) {
				if (this.isConnected == false) {
					this.extraLine.push('连接中...');
					this.text = this.extraLine.join('\n');
					this.connectWebSocket();
				}
				else {
					this.isConnected = false;
					// this.canConnect = true;
					this.extraLine.push('断开连接');
					this.text = this.extraLine.join('\n');
					this.closeWebSocket();
				}
			},
			
			composeContent: function(e) {
				this.texts.push('Human:' + this.inputContentValue);
				let text = '';
				for (let item of this.texts) {
				    text += item;
					text += '\n';
				}
				console.log('text=' + text);
				return text;
			},
			
			chat: function(e) {
				console.log(this.inputContentValue);
				this.extraLine.push('我：' + this.inputContentValue);
				this.text = this.extraLine.join('\n');
				let obj = {};
				obj.request_id = this.guid();
				obj.format = "utf-8";
				obj.timestamp = 1620358489;
				obj.version = 1;
				obj.command = "chat";
				let data = {};
				data.direction = "human_to_ai";
				data.type = "text";
				data.language = "zh_CN";
				data.content = this.composeContent();
				data.addition = "";																
				obj.data = data;
				//console.log(obj);
				let content = JSON.stringify(obj);
				//console.log(content);
				this.sendWebSocketMessage(content);
				this.inputContentValue = null;
			},
			
			talk: function(e) {
				console.log(this.inputContentValue);
				this.extraLine.push('我：' + this.inputContentValue);
				this.text = this.extraLine.join('\n');
				let obj = {};
				obj.request_id = this.guid();
				obj.format = "utf-8";
				obj.timestamp = 1620358489;
				obj.version = 1;
				obj.command = "chat";
				let data = {};
				data.direction = "human_to_robot";
				data.type = "text";
				data.language = "zh_CN";
				data.content = this.inputContentValue;
				data.addition = "";																
				obj.data = data;
				//console.log(obj);
				let content = JSON.stringify(obj);
				//console.log(content);
				this.sendWebSocketMessage(content);
			},
			
			clearInput: function(event) {
			    this.inputContentValue = event.detail.value;
			    if (event.detail.value.length > 0) {
			        this.showClearIcon = true;
			    } else {
			        this.showClearIcon = false;
			    }
			},
			clearIcon: function() {
			    this.inputContentValue = '';
			    this.showClearIcon = false;
			},
		
			S4() {
				return (((1 + Math.random()) * 0x10000) | 0).toString(16).substring(1);
			},
			guid() {
				return (this.S4() + this.S4() + "-" + this.S4() + "-" + this.S4() + "-" + this.S4() + "-" + this.S4() + this.S4() +
					this.S4());
			},
			async genImgCode() {
				let uuid = "cms" + this.guid();
				this.codeImg = this.$serverUrl + '/captcha.jpg?uuid=' + uuid;
			}
		}
	}
</script>

<style>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
	}
	
	.logo {
		height: 200rpx;
		width: 200rpx;
		margin-top: 200rpx;
		margin-left: auto;
		margin-right: auto;
		margin-bottom: 50rpx;
	}
	
	.text-area {
		display: flex;
		justify-content: center;
	}
	
	.title {
		font-size: 36rpx;
		color: #8f8f94;
	}
	.text-box {
		/* margin-bottom: 40rpx; */
		/* padding: 40rpx 0; */
		padding-top: 0px;
		padding-bottom: 0px;
		display: flex;
		min-height: 300rpx;
		background-color: #FFFFFF;
		justify-content: flex-start;
		align-items: top;
		text-align: left;
		font-size: 30rpx;
		color: #353535;
		line-height: 1.8;
	}
	
	.flex-item {
		width: 33.3%;
		height: 200rpx;
		text-align: center;
		line-height: 200rpx;
	}

	.flex-item-V {
		width: 100%;
		height: 150rpx;
		text-align: center;
		line-height: 150rpx;
	}

	.text {
		margin: 15rpx 10rpx;
		padding: 0 20rpx;
		background-color: #ebebeb;
		height: 70rpx;
		line-height: 70rpx;
		text-align: center;
		color: #777;
		font-size: 26rpx;
	}

	.desc {
		/* text-indent: 40rpx; */
	}
	.flex-pc {
		display: flex;
		justify-content: center;
	}
</style>
