<template>
  <div class="hello">
    <button v-on:click="start(true)">Play</button>
    <video id="localVideo" ref="localVideo" autoplay muted style="width:40%;"></video>
    <video autoplay style="width:40%;" ref="remoteStream-1"></video>
    <video autoplay style="width:40%;" ref="remoteStream-2"></video>
    <video autoplay style="width:40%;" ref="remoteStream-3"></video>
    <video autoplay style="width:40%;" ref="remoteStream-4"></video>
    <video autoplay style="width:40%;" ref="remoteStream-5"></video>
    <!-- <div v-for="(stream, idx) in remoteStreams" v-bind:key="idx">
      <video autoplay style="width:40%;" :ref="`remoteStream-${idx}`" :srcObject="stream"></video>
    </div>-->
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
      localStream: {},
      nextStream: 1,
      nextRemoteStream: {},
      peerConnections: []
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
      const peerConnection = new RTCPeerConnection(this.peerConnectionConfig);
      peerConnection.onicecandidate = this.gotIceCandidate;
      peerConnection.ontrack = this.addRemoteStream;
      peerConnection.onnegotiationneeded = data => {
        console.log("in negotiaton needed");
        console.log(data);
      };
      peerConnection.addStream(this.localStream);

      if (isCaller) {
        const offer = await peerConnection.createOffer();
        await peerConnection.setLocalDescription(offer);
        this.serverConnection.send(
          JSON.stringify({
            sdp: peerConnection.localDescription,
            uuid: this.uuid
          })
        );
      }

      this.peerConnections.push(peerConnection);
    },

    async handleServerMessage(message) {
      this.peerConnections.map(async peerConnection => {
        if (!peerConnection) {
          this.start(false);
        }

        const signal = JSON.parse(message.data);

        // Ignore messages from ourself
        if (signal.uuid == this.uuid) {
          return;
        }

        if (signal.sdp) {
          await peerConnection.setRemoteDescription(
            new RTCSessionDescription(signal.sdp)
          );

          if (signal.sdp.type == "offer") {
            console.log("received offer signal");
            const answer = await peerConnection.createAnswer();
            await peerConnection.setLocalDescription(answer);

            this.serverConnection.send(
              JSON.stringify({
                sdp: peerConnection.localDescription,
                uuid: this.uuid
              })
            );
          }
        } else if (signal.ice) {
          console.log("received ICE candidate");
          await peerConnection.addIceCandidate(new RTCIceCandidate(signal.ice));
        }
      });
    },

    gotIceCandidate(event) {
      if (event.candidate != null) {
        this.serverConnection.send(
          JSON.stringify({ ice: event.candidate, uuid: this.uuid })
        );
      }
    },

    addRemoteStream(event) {
      console.log("got remote stream");
      // console.log(event);
      this.nextRemoteStream = event.streams[0];
      this.$refs[`remoteStream-${this.nextStream}`].srcObject =
        event.streams[0];
      this.nextStream++;
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
