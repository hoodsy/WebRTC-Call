<template>
  <div class="hello">
    <button v-on:click="start(true)">Play</button>
    <video id="localVideo" ref="localVideo" autoplay muted style="width:40%;"></video>
    <video autoplay style="width:40%;" ref="remoteVideo"></video>
  </div>
</template>

<script>
export default {
  name: "HelloWorld",

  data() {
    return {
      peerConnectionConfig: {
        iceServers: [
          { urls: "stun:stun.stunprotocol.org:3478" },
          { urls: "stun:stun.l.google.com:19302" }
        ]
      },
      localStream: {}
    };
  },

  mounted() {
    this.serverConnection = new WebSocket(
      "wss://9fb46aab.ngrok.io"
      // "wss://localhost:8443"
      // "wss://localhost:8443" + window.location.hostname + ":8443"
    );
    this.serverConnection.onmessage = this.handleServerMessage;
    this.uuid = this.createUUID();
    this.playLocal();
  },

  methods: {
    async playLocal() {
      const stream = await navigator.mediaDevices.getUserMedia({
        video: true,
        audio: true
      });
      this.$refs.localVideo.srcObject = stream;
      this.localStream = stream;
    },

    async start(isCaller) {
      this.peerConnection = new RTCPeerConnection(this.peerConnectionConfig);
      this.peerConnection.onicecandidate = this.gotIceCandidate;
      this.peerConnection.ontrack = this.gotRemoteStream;
      this.peerConnection.addStream(this.localStream);

      if (isCaller) {
        this.peerConnection.createOffer().then(this.createdDescription);
      }
    },

    async handleServerMessage(message) {
      if (!this.peerConnection) {
        this.start(false);
      }

      const signal = JSON.parse(message.data);

      // Ignore messages from ourself
      if (signal.uuid == this.uuid) {
        return;
      }

      if (signal.sdp) {
        await this.peerConnection.setRemoteDescription(
          new RTCSessionDescription(signal.sdp)
        );

        if (signal.sdp.type == "offer") {
          console.log("received offer signal");
          const answer = await this.peerConnection.createAnswer();
          await this.peerConnection.setLocalDescription(answer);
          this.serverConnection.send(
            JSON.stringify({
              sdp: this.peerConnection.localDescription,
              uuid: this.uuid
            })
          );
        }
      } else if (signal.ice) {
        console.log("received ICE candidate");
        await this.peerConnection.addIceCandidate(
          new RTCIceCandidate(signal.ice)
        );
      }
    },

    gotIceCandidate(event) {
      if (event.candidate != null) {
        this.serverConnection.send(
          JSON.stringify({ ice: event.candidate, uuid: this.uuid })
        );
      }
    },

    gotRemoteStream(event) {
      console.log("got remote stream");
      this.$refs.remoteVideo.srcObject = event.streams[0];
    },

    async createdDescription(description) {
      console.log("got description");

      await this.peerConnection.setLocalDescription(description);
      this.serverConnection.send(
        JSON.stringify({
          sdp: this.peerConnection.localDescription,
          uuid: this.uuid
        })
      );
    },

    createUUID() {
      function s4() {
        return Math.floor((1 + Math.random()) * 0x10000)
          .toString(16)
          .substring(1);
      }

      return (
        s4() +
        s4() +
        "-" +
        s4() +
        "-" +
        s4() +
        "-" +
        s4() +
        "-" +
        s4() +
        s4() +
        s4()
      );
    }
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
