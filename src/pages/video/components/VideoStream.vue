<template>
<div id="videos">
  <div class="remote-video-container">
    <video id="remote-video" autoplay="true" v-show="showRemote"></video>
    <remote-controls :socket="socket" :socketReady="socketReady" :inCall="inCall" :verbose="verbose"></remote-controls>
  </div>

  <div class="local-video-container">
    <video id="local-video" autoplay="true" v-show="showLocal" muted></video>
    <local-controls :socket="socket" :toggle="toggleLocalVideo"></local-controls>
  </div>
</div>
</template>


<script>

  import TextBox from './TextBox.vue';
  import RemoteControls from './RemoteControls.vue';
  import LocalControls from './LocalControls.vue';
  import WebRTC from '../../../services/webrtc.js';
  import recordrtc from 'recordrtc';
  import axios from 'axios';
  import MetricsGatherer from '../../../services/metricsGatherer.js';

  export default {

    props: [
    'socket',
    'socketReady',
    'roomJoined',
    'verbose'
    ],

    data: function() {
      return {
        localVideo: null,           // Video element displaying local stream
        remoteVideo: null,          // Video element displaying remote stream
        rtc: null,                  // WebRTC instance
        gatherer: null,              // Video/audio stream from counterpart
        showRemote: true,
        showLocal: true,
        inCall: false
      };
    },

    watch: {
      socketReady: function() {
        if(this.socketReady) {
          this.rtc = new WebRTC(
            this.localVideo,
            this.remoteVideo,
            this.socket,
            this.verbose
          );
          this.rtc.start()
          .then(this.registerListeners);
          this.gatherer = new MetricsGatherer(this.socket, localStorage.id_token);
        }
      }
    },

    methods: {

      toggleLocalVideo: function() {
        this.showLocal = !this.showLocal;
      },
      // ========================================================================
      // == Begin RecordRTC =====================================================
      // ========================================================================
      postAudioClip: function (recorder) {
        let rawAudio = recorder.getBlob();
        let audioFile = new File([rawAudio], 'snippet.wav', {type: 'audio/wav'});
        let fromLang_toLang = `${this.$root.$data.nativeLang}_${this.$root.$data.foreignLang}`;

        let formData = new FormData();
        formData.append('file', audioFile);

        axios.post(`/api/transcribe/${fromLang_toLang}`, formData)
        .then(response => {
          let message = {id: Date.now(), text: response.data};
          this.socket.translateText(message);
          console.log('You sent:', message);
        })
        .catch(error => {
          console.log('Error getting transcript from server');
          console.log(error);
        });
      },

      recordSnippet: function() {
        console.log('record');
        var recorder = recordrtc(this.rtc.localVideoStream, {
          type: 'audio',
          recorderType: recordrtc.StereoAudioRecorder,
          numberOfAudioChannels: 1
        });

        recorder.setRecordingDuration(3000, () => {
          setTimeout(this.recordSnippet, 0);
          setTimeout(() => {
            this.postAudioClip(recorder);
          }, 0);
        });

        recorder.startRecording();
      },

      // ========================================================================
      // == End RecordRTC =======================================================
      // ========================================================================

      // Convenience method for logging debugging messages
      log: function() {
        this.verbose && console.log.apply(console, ['VideoStream:', ...arguments]);
      },

      // Attach necessary listeners to socket and to window
      // Send 'Ready' status to parent component on completion
      registerListeners: function() {

        this.log('Registering listeners...');

        // Occurs if we are the first client in the room
        this.socket.on('created room', (room) => {
          this.log(`Created room "${room}", we are the Caller`);
          this.rtc.isCaller = true;
          this.log(`Waiting for callee to join...`);
        });

        // Occurs if we are not the first client in the room
        this.socket.on('joined room', (room) => {
          this.log(`Joined room "${room}", we are the Callee`);
          this.inCall = true;
          this.rtc.isCaller = false;
        });

        // Occurs when another party joins our room
        this.socket.on('callee joined', (room) => {
          this.log(`Callee has joined room "${room}"`);
          // It is now safe to initiate call
          this.inCall = true;
          this.rtc.sendOffer();
        });

        this.socket.on('relay', (data) => {
          if (data.type === 'ICECandidate') {   // ICE candidate received from counterpart
            this.rtc.handleReceiveIceCandidate(data);
          } else if (data.type === 'Offer') {   // Offer received from Caller
            this.rtc.handleReceiveOffer(data.sdp);
            this.gatherer.startCallWatcher(this.$root.$data.nativeLang, this.$root.$data.foreignLang);
          } else if (data.type === 'Answer') {  // Answer received from Callee
            this.rtc.handleReceiveAnswer(data.sdp);
            this.gatherer.startCallWatcher(this.$root.$data.nativeLang, this.$root.$data.foreignLang);
          } else if (data.type === 'Bye') {     // Hangup received from counterpart
            this.log('Partner has hung up');
            this.inCall = false;
            this.gatherer.sendCallData();
            this.rtc.handleRemoveStream();
          }
        });

        this.socket.on('control', (message) => {
          this.log('control received: ', message);
          if (message === 'cam') {
            this.showRemote = !this.showRemote;
          }

          if (message === 'mic') {
            if (this.remoteVideo.muted) {
              this.remoteVideo.muted = false;
            } else {
              this.remoteVideo.muted = true;
            }
          }
        });

        window.addEventListener('beforeunload', () => {
          this.gatherer.sendCallData();
          this.rtc.hangup();
        });

        // Signal parent component that we are ready to receive messages
        this.$emit('Ready', 'VideoStream');
      }
    },

    components: {
      TextBox,
      RemoteControls,
      LocalControls
    },

    mounted: function() {
      // Initialize references to HTML5 video elements
      this.log('Mounted');
      this.localVideo = document.getElementById('local-video');
      this.remoteVideo = document.getElementById('remote-video');
      window.localVideo = this.localVideo;
      setTimeout(this.recordSnippet, 1000);
    },

    beforeDestroy: function() {
      // this.recorder.stopRecording(function () {
      //   console.log('Stopped recording.');
      // });
      this.gatherer && this.gatherer.sendCallData();
      this.rtc && this.rtc.hangup();
    }
  }
</script>


<style>


  #videos {
    display: grid;
    grid-gap: 10px;

    grid-template-columns: repeat(8, [col] auto ) ;
    grid-template-rows: repeat(8, [row] auto  );
  }

  #videos > * {
    color: #fff;
    z-index:10;
  }

  #remote-video {
    /*background: rgba(0,0,0,0.6);*/
    /*width: 100%;*/
    /*height: 100%;*/
    /*box-shadow: 2px 2px 20px 2px #111;*/
    /*grid-column: col 1 / span  8;*/
    /*grid-row: row 1 / span 8;*/
  }

  #local-video {
    /*background: rgba(0,0,0,0.6);*/
    /*grid-column: col 8;*/
    /*grid-row: row 8;*/
    width: auto;
    height: 100%;
    /*box-shadow: 2px 2px 20px 2px #111;*/
    /*z-index:11;*/
    /*margin-bottom: 20px;*/
  }

  .remote-video-container {
    background: rgba(0,0,0,0.6);
    width: 100%;
    height: 100%;
    box-shadow: 2px 2px 20px 2px #111;
    grid-column: col 1 / span  8;
    grid-row: row 1 / span 8;
  }

  .local-video-container {
    background: rgba(0,0,0,0.6);
    grid-column: col 8;
    grid-row: row 8;
    width: auto;
    height: 150px;
    box-shadow: 2px 2px 20px 2px #111;
    z-index:11;
    /*margin-bottom: 20px;*/
  }

  #recorder {
    color: black;
  }


</style>
