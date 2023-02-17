<template>
  <div class="record-playback" v-if="!isRecording">
    <div v-if="recordingData.length == 0">
      <img
        src="@/assets/icons8-microphone-60.png"
        @click="addRecording"
        class="add-record"
      />
    </div>
    <div class="playback" v-else>
      <img
        src="@/assets/icons8-checkmark-50.png"
        @click="approveRecorded"
        class="approve-recorded"
      />
      <img
        src="@/assets/icons8-microphone-60.png"
        @click="recordAudio"
        class="record-button"
      />
      <audio
        controls
        :src="recordedAudioUrl"
        controlsList="nodownload nofullscreen noremoteplayback noplaybackrate"
      ></audio>
      <img
        src="@/assets/icons8-bin-48.png"
        @click="deleteRecorded"
        class="bin-button"
      />
    </div>
  </div>
  <div class="recording" v-else>
    <div>
      <img
        src="@/assets/icons8-pause-squared-48.png"
        @click="pauseRecording"
        class="pause-button"
      />
    </div>
  </div>
  <br />
  <div v-if="recordingData.length != 0 && !isRecording && transcript !== null">
    <div v-if="!editTranscript">
      <textarea
        class="transcript"
        v-model.trim="transcript"
        placeholder="Transcript"
        readonly
      /><br /><img
        src="@/assets/icons8-edit-24.png"
        @click="edit"
        class="edit-button"
      />
    </div>
    <div v-else>
      <textarea
        class="transcript"
        v-model.trim="editableTranscript"
        placeholder="Transcript"
        ref="editTranscript"
      />
      <br /><img
        src="@/assets/icons8-checkmark-48.png"
        @click="updateTranscript"
        class="edit-button"
      /><img
        src="@/assets/icons8-cancel-48.png"
        @click="cancelEdit"
        class="edit-button"
      />
    </div>
  </div>
</template>
<script>
import axios from "axios";
export default {
  name: "VoiceRecorder",
  data() {
    return {
      audio: null,
      isRecording: false,
      recorder: null,
      recordingData: [],
      recordedAudioUrl: null,
      recordingFile: null,
      transcript: null,
      editableTranscript: null,
      editTranscript: false,
    };
  },
  methods: {
    pauseRecording: function () {
      if (this.recorder) {
        this.recorder.pause();
        this.recordingFile = new Blob(this.recordingData, {
          type: "audio/ogg; codecs=opus",
        });
        if (this.recordedAudioUrl) {
          window.URL.revokeObjectURL(this.recordedAudioUrl);
        }
        this.recordedAudioUrl = window.URL.createObjectURL(this.recordingFile);
        this.transcribe();
      }
    },
    addRecording: function () {
      navigator.permissions.query({ name: "microphone" }).then((permission) => {
        if (permission.state === "granted") {
          if (!this.audio) {
            this.audio = navigator.mediaDevices.getUserMedia({ audio: true });
          }
          this.recordAudio();
        } else {
          this.audio = navigator.mediaDevices.getUserMedia({ audio: true });
        }
      });
    },
    recordAudio: function () {
      if (!this.recorder || this.recorder.state == "inactive") {
        this.audio.then((stream) => {
          this.recorder = new MediaRecorder(stream);
          this.recorder.onstart = () => {
            this.isRecording = true;
          };
          this.recorder.onresume = () => {
            this.isRecording = true;
          };
          this.recorder.onpause = () => {
            this.isRecording = false;
          };
          this.recorder.ondataavailable = (event) => {
            this.recordingData.push(event.data);
          };
          this.recorder.start(0); //0 for as little audio buffering as possible so recording starts immediately
        });
      } else {
        this.recorder.resume();
      }
    },
    deleteRecorded: function () {
      this.recorder.ondataavailable = () => {};
      this.recorder.stop();
      this.recordingFile = null;
      this.recordingData = [];
      this.transcript = null;
    },
    approveRecorded: function () {
      const formData = new FormData();
      formData.append("audio", this.recordingFile);
      formData.append("transcript", "hello world");
      axios
        .post(process.env.VUE_APP_BACKEND_URL + "/vidnote/", formData, {
          headers: {
            "Content-Type": "multipart/form-data",
          },
          responseType: "blob",
        })
        .then((response) => {
          console.log(response);
          const outputFile = new File([response.data], "result.mp4", {
            type: "video/mp4",
          });
          const outputFileUrl = URL.createObjectURL(outputFile);
          const audio = new Audio(outputFileUrl);
          audio.play();
          const shareData = {
            files: [outputFile],
          };
          console.log(navigator.canShare(shareData));
          console.log(typeof navigator.share === "function");
          navigator.share(shareData);
          const link = document.createElement("a");
          link.href = outputFileUrl;
          link.download = "result";
          link.click();
        })
        .catch(function (error) {
          console.log(error);
        });
    },
    transcribe: function () {
      const formData = new FormData();
      formData.append("audio", this.recordingFile);
      axios
        .post(process.env.VUE_APP_BACKEND_URL + "/transcribe/", formData, {
          headers: {
            "Content-Type": "multipart/form-data",
          },
          responseType: "json",
        })
        .then((response) => {
          console.log(response);
          this.transcript = response.data.transcript;
        })
        .catch(function (error) {
          console.log(error);
        });
    },
    edit: function () {
      this.editTranscript = true;
      this.editableTranscript = this.transcript;
      this.$nextTick(() => {
        this.$refs.editTranscript.select();
      });
    },
    cancelEdit: function () {
      this.editTranscript = false;
      this.editableTranscript = this.transcript;
    },
    updateTranscript: function () {
      this.editTranscript = false;
      this.transcript = this.editableTranscript;
    },
  },
};
</script>
<style scoped>
.add-record {
  padding: 6px 10px;
  cursor: pointer;
  transition: 0.2s;
}
.add-record:hover {
  transform: scale(1.1);
}
.recording {
  display: flex;
  justify-content: center;
}
.record-button {
  cursor: pointer;
  transition: 0.2s;
}
.record-button:hover {
  transform: scale(1.1);
}
.pause-button {
  cursor: pointer;
  transition: 0.2s;
}
.pause-button:hover {
  transform: scale(1.1);
}
.bin-button {
  cursor: pointer;
  transition: 0.2s;
}
.bin-button:hover {
  transform: scale(1.1);
}
.approve-recorded {
  cursor: pointer;
  transition: 0.2s;
}
.approve-recorded:hover {
  transform: scale(1.1);
}
.record-playback {
  display: flex;
  justify-content: center;
}
.playback {
  transform: scale(0.9);
  display: flex;
  justify-content: center;
}
.edit-button {
  padding: 6px 10px;
  border-radius: 50%;
  cursor: pointer;
}
.edit-button:hover {
  background: #e0e0e0;
}
.transcript {
  width: 30%;
  resize: none;
  height: 120px;
}
</style>