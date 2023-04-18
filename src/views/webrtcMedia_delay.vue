<template>
  <div class="box">
    <el-card>
      <el-button @click="joinRoom">connect</el-button>
      <p>peer: {{ peerName }}</p>
      <video v-show="peerName == 'push'" ref="local_video" autoplay controls src="test.mp4"></video>
      <video v-show="peerName == 'pull'" ref="remote_video" autoplay controls></video>
    </el-card>
  </div>
</template>
<script>
export default {
  data() {
    return {
      peerName: '',
      loading: false,
      roomName: "",
    };
  },
  methods: {
    // 加入房间
    joinRoom() {
      this.$socketEmit("join private room", {
        room: 'delay test',
      }).then((res) => {
        this.$notify({
          message: res.message,
          type: res.success ? "success" : "warning",
        });
        this.loading = true;
        if (res.start) {
          this.startLive();
        }
      });
    },


    // 开始链接
    async startLive(offerSdp) {
      if (!offerSdp) {
        let stream = this.$refs.local_video.captureStream();
        // 2.呼叫者创建一个RTCPeerConnection 并调用 RTCPeerConnection.addTrack()
        stream.getTracks().forEach((track) => {
          this.peer.addTrack(track, stream);
        });
      }

      if (!offerSdp) {
        this.peerName = 'push'
        // 3.呼叫者调用 RTCPeerConnection.createOffer() 来创建一个提议(offer).
        const offer = await this.peer.createOffer();
        // 4.呼叫者调用 RTCPeerConnection.setLocalDescription() (en-US) 将提议(Offer) 设置为本地描述 (即，连接的本地描述).
        await this.peer.setLocalDescription(offer);
        // 呼叫者通过信令服务器将提议(offer)传递至 本次呼叫的预期的接受者.
        this.$socketEmit("webrtc", { offer });
      } else {
        this.peerName = 'pull'
        // 7.接受者收到了提议(offer) 并调用 RTCPeerConnection.setRemoteDescription() 将其记录为远程描述 (也就是连接的另一端的描述).
        await this.peer.setRemoteDescription(offerSdp);
        // 9.接受者通过 RTCPeerConnection.createAnswer() (en-US) 创建一个应答。
        const answer = await this.peer.createAnswer();
        // 11.接受者通过信令服务器将应答传递到呼叫者.
        this.$socketEmit("webrtc", { answer });
        // 10.接受者调用 RTCPeerConnection.setLocalDescription() (en-US) 将应答(answer)   设置为本地描述. 此时，接受者已经获知连接双方的配置了.
        await this.peer.setLocalDescription(answer);
      }
    },

    // 初始化RTCPeerConnection
    init() {
      this.peer = new RTCPeerConnection();
      this.$socketOn("webrtc", (res) => {
        try {
          if (res.offer) {
            this.startLive(res.offer);
          }
          if (res.answer) {
            // 12.呼叫者接受到应答.
            // 13.呼叫者调用 RTCPeerConnection.setRemoteDescription() 将应答设定为远程描述. 如此，呼叫者已经获知连接双方的配置了.
            this.peer.setRemoteDescription(res.answer);
          }
          if (res.candidate) {
            // 除了交换关于媒体的信息(上面提到的Offer / Answer和SDP )中，对等体必须交换关于网络连接的信息。（交换ICE候选）
            this.peer.addIceCandidate(res.candidate);
          }
        } catch (error) {
          this.$notify.error("创建链接失败");
        }
      });

      // 除了交换关于媒体的信息(上面提到的Offer / Answer和SDP )中，对等体必须交换关于网络连接的信息。（交换ICE候选）
      this.peer.onicecandidate = (e) => {
        if (e.candidate) {
          this.$socketEmit("webrtc", { candidate: e.candidate });
        }
      };

      // 每个Peer建立一个track事件的响应程序，这个事件会在远程Peer添加一个track到其stream上时被触发。
      this.peer.ontrack = (e) => {
        if (e && e.streams) {
          this.$refs.remote_video.srcObject = e.streams[0];
          this.loading = false;
        }
      };
    },
  },
  created() {
    this.init();
  },
};
</script>
<style  lang="scss" scoped>
.box {
  width: 100vw;
  height: 100vh;
  padding: 10px;
  box-sizing: border-box;

  .el-card {
    box-sizing: border-box;
    padding: 30px;
    width: 100%;
    height: 100%;
    overflow-y: auto;
  }

  video {
    width: 100%;
    height: auto;
  }
}
</style>