<template>
  <div class="mobile-chat-wrapper">
    <div class="chat-history" ref="chatContentB">
      <div class="chat-content"  ref="chatContentA" >
        <div class="clear clearfix scorll" ref="chatContent">
          <div class="chat-wrapper clear clearfix" v-for="(item, index) in chatList" :key="item.id">
            <div class="chat-friend" v-if="item.uid !== '1001'">
              <div class="info-time">
                <img :src="item.headImg" alt="" />
                <span>{{ item.name }}</span>
                <span>{{ item.time }}</span>
              </div>
              <div class="chat-text" v-if="item.chatType == 0">
                <template>
                  <div :class="isSend && index === chatList.length - 1 ? 'flash_cursor_parent':''" :id="item.uid">{{ item.msg }}<i class="flash_cursor"></i></div>
                  
                </template>
                
              </div>
              <div class="chat-img" v-if="item.chatType == 1">
                <img :src="item.msg" alt="表情" v-if="item.extend.imgType == 1" style="width: 100px; height: 100px" />
                <el-image :src="item.msg" :preview-src-list="srcImgList" v-else> </el-image>
              </div>
              <div class="chat-img" v-if="item.chatType == 2">
                <div class="word-file">
                  <FileCard :fileType="item.extend.fileType" :file="item.msg"></FileCard>
                </div>
              </div>
            </div>
            <div class="chat-me" v-else>
              <div class="info-time">
                <span>{{ item.name }}</span>
                <span>{{ item.time }}</span>
                <img :src="item.headImg" alt="" />
              </div>
              <div class="chat-text" v-if="item.chatType == 0">
                {{ item.msg }}
              </div>
              <div class="chat-img" v-if="item.chatType == 1">
                <img :src="item.msg" alt="表情" v-if="item.extend.imgType == 1" style="width: 100px; height: 100px" />
                <el-image style="max-width: 300px; border-radius: 10px" :src="item.msg" :preview-src-list="srcImgList" v-else> </el-image>
              </div>
              <div class="chat-img" v-if="item.chatType == 2">
                <div class="word-file">
                  <FileCard :fileType="item.extend.fileType" :file="item.msg"></FileCard>
                </div>
              </div>
            </div>
          </div>
        </div>	
        
      </div>
    </div>
    <div class="chat-input-wrapper">
      <el-input v-model="inputMsg" autofocus />
      <img class="send-icon" src="@/assets/img/emoji/rocket.png" alt="" @click="sendText" />
    </div>
  </div>
</template>

<script>
import { animation } from '@/util/util'
import {marked} from 'marked'
import hljs from "highlight.js";

import 'highlight.js/styles/intellij-light.css';

export default {
  name: 'MobileChat',
  data() {
    return {
      websock: null,
      chatList: [
        {
          headImg: require('@/assets/img/head_portrait1.png'),
          name: 'ChatGPT',
          time: new Date().toLocaleTimeString(),
          msg: ' ChatGPT为您服务',
          chatType: 0,
          uid: '1002'
        }
      ],
      inputMsg: '',
      isSend: false,
      text:'',
    }
  },
  created() {
    this.initWebSocket()
  },
  destroyed() {
    this.websock.close() //离开路由之后断开websocket连接
  },
  methods: {
    initWebSocket() {
      //初始化weosocket
      let uid = window.localStorage.getItem("uid");
      if (uid == null || uid == '' || uid == 'null') {
        uid = this.uuid();
      }
      // 设置本地存储
      window.localStorage.setItem("uid", uid);
      const wsuri = `${process.env.VUE_APP_BASE_WS}/weixin/websocket/${uid}`
      this.websock = new WebSocket(wsuri)
      this.websock.onmessage = this.websocketonmessage
      this.websock.onopen = this.websocketonopen
      this.websock.onerror = this.websocketonerror
      this.websock.onclose = this.websocketclose
    },
    websocketonopen() {
      //此处不做任何处理
      
    },
    websocketonerror(e) {
      console.log(e);
      //连接建立失败重连
      this.initWebSocket()
    },
    websocketonmessage(e) { 
      
      if(e.data == '[DONE]'){ 
        this.text = '';
        this.isSend = false
        return
      } 
      
      const redata = JSON.parse(e.data)

      if (redata.content == null || redata.content == 'null') {
        return;
      } 
      const msgUuid =  window.localStorage.getItem("msgUuid")
      this.text = this.text + redata.content;
      this.setText(this.text, msgUuid)
    },
    websocketsend(Data) {
      //数据发送
      this.websock.send(Data)
      let msgUuid = this.uuid();
      window.localStorage.setItem("msgUuid",msgUuid)
        //数据接收
        let chatGPT = {
          headImg: require('@/assets/img/head_portrait1.png'),
          name: 'ChatGPT',
          time: new Date().toLocaleTimeString(),
          msg: "",
          chatType: 0, //信息类型，0文字，1图片
          uid: msgUuid //uid
        }
        console.log(msgUuid);
        this.sendMsg(chatGPT)
    },
    websocketclose(e) {
      //关闭
      console.log('断开连接', e)
    },
    sendText() {
      if(this.isSend){
        this.$message.warning("正在回复请稍后")
      }
      if (!this.isSend&&this.inputMsg) {
        let chatMsg = {
          headImg: require('@/assets/img/head_portrait.jpg'),
          name: 'YOU',
          time: new Date().toLocaleTimeString(),
          msg: this.inputMsg,
          chatType: 0, //信息类型，0文字，1图片
          uid: '1001' //uid
        }
        this.sendMsg(chatMsg)
        this.inputMsg = ''
        this.loading = true
        this.isSend = true
        this.websocketsend(chatMsg.msg)
      } else {
        // this.$message({
        //   message: '消息不能为空哦~',
        //   type: 'warning'
        // })
      }
    },
    //发送信息
    sendMsg(msgList) {
      this.chatList.push(msgList)
      this.scrollBottom()
    },
    //获取窗口高度并滚动至最底层
    scrollBottom() {
      this.$nextTick(() => {
        const scrollDom = this.$refs.chatContentB
        console.log(scrollDom.scrollHeight);
        // animation(scrollDom, scrollDom.scrollHeight - scrollDom.offsetHeight)
        
        scrollDom.scrollTo({
	        top:scrollDom.scrollHeight, //滚动距离
	        // behavior: 'auto',  // 滚动条立即滚动
	        behavior: 'smooth' //平滑滚动
	      })
      })
    },
    debounceSearch(keyWords) {
	    timer = setTimeout(function() {
	        getSuggestList(keyWords);
	    }, 500);
	  },
    uuid() {
      var s = [];
      var hexDigits = "0123456789abcdef";
      for (var i = 0; i < 36; i++) {
          s[i] = hexDigits.substr(Math.floor(Math.random() * 0x10), 1);
      }
      s[14] = "4"; // bits 12-15 of the time_hi_and_version field to 0010
      s[19] = hexDigits.substr((s[19] & 0x3) | 0x8, 1); // bits 6-7 of the clock_seq_hi_and_reserved to 01
      s[8] = s[13] = s[18] = s[23] = "-";

      var uuid = s.join("");
      return uuid;
    },
    setText(text, uuid_str) {
      let content = document.getElementById(uuid_str);
       marked.setOptions({
          renderer: new marked.Renderer(),
          highlight: function(code, lang) {
            // const hljs = require('highlight.js');
            // const language = hljs.getLanguage(lang) ? lang : 'plaintext';
            // return hljs.highlight(code, { language }).value;
            return hljs.highlightAuto(code).value;
          },
          langPrefix: 'hljs language-', // highlight.js css expects a top-level 'hljs' class.
          // renderer: rendererMD,
          gfm: true,//默认为true。 允许 Git Hub标准的markdown.
          tables: true,//默认为true。 允许支持表格语法。该选项要求 gfm 为true。
          breaks: false,//默认为false。 允许回车换行。该选项要求 gfm 为true。
          pedantic: false,//默认为false。 尽可能地兼容 markdown.pl的晦涩部分。不纠正原始模型任何的不良行为和错误。
          sanitize: false,//对输出进行过滤（清理）
          smartLists: true,
          smartypants: false//使用更为时髦的标点，比如在引用语法中加入破折号。
      });
      
      content.innerHTML = ` <div class="markdown-body">${marked.parse(text + '<i class="flash_cursor"></i>')}</div>`;
      this.scrollBottom()
    }
  }
}
</script>

<style lang="scss">
@import "@/assets/css/markdown.css";
@import "@/assets/css/my-markdown.css";
.mobile-chat-wrapper {
  display: flex;
  flex-direction: column;
  overflow: hidden;
  height: 100vh;
  position: relative;
  // background-color: #272a37;
  background-color: #ededed;
  font-weight: 500;
  .chat-history {
    flex: 1 1 0;
    overflow-y: auto;
  }

  .chat-input-wrapper {
    display: flex;
    align-items: center;
    padding: 8px 16px 8px 8px;
    position: absolute;
    left: 0;
    right: 0;
    bottom: 0;
    padding-bottom: 25px;
    // background-color: #272a37;
    background-color: #f6f6f6;
    .send-icon {
      height: 40px;
      margin-left: 16px;
    }
  }

  .chat-content {
    width: 100%;
    //height: 85vh;
    overflow-y: scroll;
    padding: 20px;
    box-sizing: border-box;
    margin-bottom: 15%;
    &::-webkit-scrollbar {
      width: 0;
      /* Safari,Chrome 隐藏滚动条 */
      height: 0;
      /* Safari,Chrome 隐藏滚动条 */
      display: none;
      /* 移动端、pad 上Safari，Chrome，隐藏滚动条 */
    }

    .chat-wrapper {
      position: relative;
      word-break: break-all;
      display: block;
      .chat-friend {
        width: 100%;
        float: left;
        margin-bottom: 20px;
        display: flex;
        flex-direction: column;
        justify-content: flex-start;
        align-items: flex-start;

        .chat-text {
          max-width: 90%;
          padding: 20px;
          border-radius: 20px 20px 20px 5px;
          // background-color: rgb(56, 60, 75);
          background-color: #fff;
          // color: #fff;
          color: #181818;

          // &:hover {
          //   background-color: rgb(39, 42, 55);
          // }

          // pre {
          //   // white-space: break-spaces;
          // }
        }

        .chat-img {
          img {
            width: 100px;
            height: 100px;
          }
        }

        .info-time {
          margin: 10px 0;
          color: #181818;
          font-size: 14px;

          img {
            width: 30px;
            height: 30px;
            border-radius: 50%;
            vertical-align: middle;
            margin-right: 10px;
          }

          span:last-child {
            color: rgb(101, 104, 115);
            margin-left: 10px;
            vertical-align: middle;
          }
        }
      }

      .chat-me {
        width: 100%;
        float: right;
        margin-bottom: 20px;
        position: relative;
        display: flex;
        flex-direction: column;
        justify-content: flex-end;
        align-items: flex-end;

        .chat-text {
          float: right;
          max-width: 90%;
          padding: 20px;
          border-radius: 20px 20px 5px 20px;
          // background-color: rgb(29, 144, 245);
          background-color: #a9e979;
          // color: #fff;
          color: #181818;

          &:hover {
            background-color: rgb(26, 129, 219);
          }
        }

        .chat-img {
          img {
            max-width: 300px;
            max-height: 200px;
            border-radius: 10px;
          }
        }

        .info-time {
          margin: 10px 0;
          // color: #fff;
          color: #181818;
          font-size: 14px;
          display: flex;
          justify-content: flex-end;

          img {
            width: 30px;
            height: 30px;
            border-radius: 50%;
            vertical-align: middle;
            margin-left: 10px;
          }

          span {
            line-height: 30px;
          }

          span:first-child {
            color: rgb(101, 104, 115);
            margin-right: 10px;
            vertical-align: middle;
          }
        }
      }
    }
  }
  .flash_cursor_parent .flash_cursor {
    width: 10px;
    height: 30px;
    display: inline-block;
    background: #d6e3f5;
    opacity: 1;
    animation: glow 800ms ease-out infinite alternate;
  }
  @keyframes glow {
    0% {
      opacity: 1;
    }

    25% {
      opacity: 0.5;
    }

    50% {
      opacity: 0;
    }

    75% {
      opacity: 0.5;
    }

    100% {
      opacity: 1;
    }
  }

  .clearfix:after { content: ""; display:block; height: 0; clear: both; visibility: hidden;}
				.chearfix {*zoom: 1;}

			.clearfix:after {
				content: "";
				display: block;
				height: 0;
				visibility: hidden;
				clear: both;
			}
			.clearfix {
				*zoom: 1; /* //这是ie6，7专用的 */
			}
}
</style>
