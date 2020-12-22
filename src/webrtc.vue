<template>
  <div class="video-list" > 
      <div v-for="item in videoList"
          v-bind:video="item"
          v-bind:key="item.id"
          v-bind:id="'item-' + item.id"
          class="video-item"
          @click="triggerFullScreen(item.id)">
        <video autoplay playsinline ref="videos" :height="cameraHeight" :muted="item.muted" :id="item.id">
        </video>
        <div class="overlay">
          <h3>{{ displayedName }}</h3>
        </div>
      </div>
  </div>
</template>

<script>
  import RTCMultiConnection from 'rtcmulticonnection';
  require('adapterjs');
  export default {
    name: 'vue-webrtc',
    components: {
      RTCMultiConnection
    },
    data() {
      return {
        rtcmConnection: null,
        localVideo: null,
        videoList: [],
        canvas: null
      };
    },
    props: {
      displayedName: {
        type: String,
        default: ''
      },
      roomId: {
        type: String,
        default: 'public-room'
      },
      socketURL: {
        type: String,
        default: 'https://rtcmulticonnection.herokuapp.com:443/'
      },
      cameraHeight: {
        type: [Number, String],
        default: 160
      },
      autoplay: {
        type: Boolean,
        default: true
      },
      screenshotFormat: {
        type: String,
        default: 'image/jpeg'
      },
      enableAudio: {
        type: Boolean,
        default: true
      },
      enableVideo: {
        type: Boolean,
        default: true
      },
      enableLogs: {
        type: Boolean,
        default: false
      },
      stunServer: {
        type: String,
        default: null
      },
      turnServer: {
        type: String,
        default: null
      }
    },
    watch: {
    },
    mounted() {
      var that = this;
      this.rtcmConnection = new RTCMultiConnection();
      this.rtcmConnection.socketURL = this.socketURL;
      this.rtcmConnection.autoCreateMediaElement = false;
      this.rtcmConnection.enableLogs = this.enableLogs;
      this.rtcmConnection.session = {
        audio: this.enableAudio,
        video: this.enableVideo
      };
      this.rtcmConnection.sdpConstraints.mandatory = {
        OfferToReceiveAudio: this.enableAudio,
        OfferToReceiveVideo: this.enableVideo
      };
      if ((this.stunServer !== null) || (this.turnServer !== null)) {
        this.rtcmConnection.iceServers = []; // clear all defaults
      }
      if (this.stunServer !== null) {
        this.rtcmConnection.iceServers.push({
          urls: this.stunServer
        });
      }
      if (this.turnServer !== null) {
        var parse = this.turnServer.split('%');
        var username = parse[0].split('@')[0];
        var password = parse[0].split('@')[1];
        var turn = parse[1];
        this.rtcmConnection.iceServers.push({
          urls: turn,
          credential: password,
          username: username
        });
      }
      this.rtcmConnection.onstream = function (stream) {
        let found = that.videoList.find(video => {
          return video.id === stream.streamid
        })
        if (found === undefined) {
          let video = {
            id: stream.streamid,
            muted: stream.type === 'local'
          };

          that.videoList.push(video);

          if (stream.type === 'local') {
            that.localVideo = video;
            that.localVideo.up = true;
          }
        }

        setTimeout(function(){ 
          for (var i = 0, len = that.$refs.videos.length; i < len; i++) {
            if (that.$refs.videos[i].id === stream.streamid)
            {
              that.$refs.videos[i].srcObject = stream.stream;
              break;
            }
          }
        }, 1000);
        
        that.$emit('joined-room', stream.streamid);
      };
      this.rtcmConnection.onstreamended = function (stream) {
        var newList = [];
        that.videoList.forEach(function (item) {
          if (item.id !== stream.streamid) {
            newList.push(item);
          }
        });
        that.videoList = newList;
        that.$emit('left-room', stream.streamid);
      };

      this.rtcmConnection.onmute = function(e) {
        that.$emit('mute-video', e);
      };

      this.rtcmConnection.onunmute = function(e) {
        that.$emit('unmute-video', e);
      };

    },
    methods: {
      triggerFullScreen(streamId) {
        let videoObj = document.getElementById(streamId)
        if (videoObj.requestFullscreen) {
          videoObj.requestFullscreen();
        }
        else if (videoObj.msRequestFullscreen) {
          videoObj.msRequestFullscreen();
        }
        else if (videoObj.mozRequestFullScreen) {
          videoObj.mozRequestFullScreen();
        }
        else if (videoObj.webkitRequestFullscreen) {
          videoObj.webkitRequestFullscreen();
        } else {
          console.log("Fullscreen API is not supported");
        }
        videoObj.removeAttribute('controls');
      },
      join() {
         var that = this;
         this.rtcmConnection.openOrJoin(this.roomId, function (isRoomExist, roomid) {
            if (isRoomExist === false && that.rtcmConnection.isInitiator === true) {
              that.$emit('opened-room', roomid);
            }
          });
        /* Disable media when not active before join */
          if (!this.enableAudio) {
            this.changeMicroState();
          }
          if (!this.enableVideo) {
            this.changeVideoState();
          }
      },
      leave() {
        this.rtcmConnection.attachStreams.forEach(function (localStream) {
          localStream.stop();
        });
        this.videoList = [];
      },
      changeVideoState() {
        var that = this;
        this.rtcmConnection.attachStreams.forEach(function (localStream) {
          if (that.localVideo.up) {
            localStream.mute('video', false);
          } else {
            localStream.unmute('video', false);
            // To correct unmute bug on audio track
            localStream.unmute('audio', false);
          }
          that.localVideo.up = !that.localVideo.up;
        });
      },
      changeMicroState() {
        this.rtcmConnection.attachStreams.forEach(function (localStream) {
          localStream.getAudioTracks()[0].enabled = !localStream.getAudioTracks()[0].enabled;
        });
      },
      capture() {
        return this.getCanvas().toDataURL(this.screenshotFormat);
      },
      getCanvas() {
        let video = this.getCurrentVideo();
        if (video !== null && !this.ctx) {
          let canvas = document.createElement('canvas');
          canvas.height = video.clientHeight;
          canvas.width = video.clientWidth;
          this.canvas = canvas;
          this.ctx = canvas.getContext('2d');
        }
        const { ctx, canvas } = this;
        ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
        return canvas;
      },
      getCurrentVideo() {
        if (this.localVideo === null) {
          return null;
        }
        for (var i = 0, len = this.$refs.videos.length; i < len; i++) {
          if (this.$refs.videos[i].id === this.localVideo.id)
            return this.$refs.videos[i];
        }
        return null;
      },
      shareScreen() {
        var that = this;
        if (navigator.getDisplayMedia || navigator.mediaDevices.getDisplayMedia) {
          function addStreamStopListener(stream, callback) {
            var streamEndedEvent = 'ended';
            if ('oninactive' in stream) {
                streamEndedEvent = 'inactive';
            }
            stream.addEventListener(streamEndedEvent, function() {
                callback();
                callback = function() {};
            }, false);
          }

          function onGettingSteam(stream) {
            that.rtcmConnection.addStream(stream);
            that.$emit('share-started', stream.streamid);

            addStreamStopListener(stream, function() {
              that.rtcmConnection.removeStream(stream.streamid);
              that.$emit('share-stopped', stream.streamid);
            });
          }

          function getDisplayMediaError(error) {
            console.log('Media error: ' + JSON.stringify(error));
          }

          if (navigator.mediaDevices.getDisplayMedia) {
            navigator.mediaDevices.getDisplayMedia({video: true, audio: false}).then(stream => {
              onGettingSteam(stream);
            }, getDisplayMediaError).catch(getDisplayMediaError);
          }
          else if (navigator.getDisplayMedia) {
            navigator.getDisplayMedia({video: true}).then(stream => {
              onGettingSteam(stream);
            }, getDisplayMediaError).catch(getDisplayMediaError);
          }
        }
      }
    }
  };
</script>
<style scoped>
  .video-list {
    background: black;
    height: auto;
    height: 100%;
    width: 100%;
  }

  video {
      width:100%;
      height:100%;
    }

  /* one item */
  .video-item:first-child:nth-last-child(1) {
      width: 100%;
      height: 100%;
  }

  /* two items */
  .video-item:first-child:nth-last-child(2),
  .video-item:first-child:nth-last-child(2) ~ .video-item {
      width: 100%;
      height: 50%;
  }

  /* three and four items */
  .video-item:first-child:nth-last-child(3),
  .video-item:first-child:nth-last-child(3) ~ .video-item,
  .video-item:first-child:nth-last-child(4),
  .video-item:first-child:nth-last-child(4) ~ .video-item {
      width: 50%;
      height: 50%;
  }


  /* five and six items */
  .video-item:first-child:nth-last-child(5),
  .video-item:first-child:nth-last-child(5) ~ .video-item,
  .video-item:first-child:nth-last-child(6),
  .video-item:first-child:nth-last-child(6) ~ .video-item  {
      width: 50%;
      height: 33.3%;
  }

  /* 7 to 9 */
  .video-item:first-child:nth-last-child(n+7),
  .video-item:first-child:nth-last-child(n+7) ~ .video-item  {
      width: 33%;
      height: 33.3%;
  }

  /* 10 to 12 */
  .video-item:first-child:nth-last-child(n+10),
  .video-item:first-child:nth-last-child(n+10) ~ .video-item  {
      width: 25%;
      height: 33.3%;
  }

  /* 12 to 16 */
  .video-item:first-child:nth-last-child(n+12),
  .video-item:first-child:nth-last-child(n+12) ~ .video-item  {
      width: 25%;
      height: 25%;
  }

  /* 16 to 25 */
  .video-item:first-child:nth-last-child(n+17),
  .video-item:first-child:nth-last-child(n+17) ~ .video-item  {
      width: 20%;
      height: 20%;
  }

  /* 26 to 30 */
  .video-item:first-child:nth-last-child(n+26),
  .video-item:first-child:nth-last-child(n+26) ~ .video-item  {
      width: 16.6%;
      height: 20%;
  }

  /* 31 to 36 */
  .video-item:first-child:nth-last-child(n+31),
  .video-item:first-child:nth-last-child(n+31) ~ .video-item  {
      width: 16.6%;
      height: 16.6%;
  }

  /* Remove controls from full screen video */
  video::-webkit-media-controls {
    display:none !important;
  }

  .video-item {
    background: black;
    display: inline-block;
    cursor: pointer;
    position: relative;
  }

  .video-item .overlay
  {
      position: absolute;
      top: 15px;
      left: 10px;
      width: 100%;
      z-index: -1 /* Hide name on video */
  }

  .video-item .overlay h3
  {
      font-size: 160%;
      color: #fff;
      font-weight: bold;
  }

  .video-item:hover {
    border: 3px solid;
    border-color: red;
  }

  /* Show name on hover */
  .video-item:hover > .overlay{
    z-index: 2;
  }

</style>
