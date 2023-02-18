<template>
  <div class="record-playback" v-if="!isRecording">
    <div v-if="recordingData.length == 0">
      <img
        src="@/assets/icons8-microphone-60.png"
        @click="addRecording"
        @contextmenu.prevent
        class="add-record"
      />
    </div>
    <div class="playback" v-else>
      <img
        src="@/assets/icons8-microphone-60.png"
        @click="recordAudio"
        @contextmenu.prevent
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
        @contextmenu.prevent
        class="bin-button"
      />
    </div>
  </div>
  <div class="recording" v-else>
    <div>
      <img
        src="@/assets/icons8-pause-squared-48.png"
        @click="pauseRecording"
        @contextmenu.prevent
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
        placeholder="Voice transcript"
        readonly
      />
      <div>
        Language:
        <select v-model="chosenLanguage" @change="changeLanguage">
          <option
            v-for="language in languages"
            :key="language"
            :value="language.value"
          >
            {{ language.display_name }}
          </option>
        </select>
      </div>
      <br /><img
        src="@/assets/icons8-edit-48.png"
        @click="edit"
        @contextmenu.prevent
        class="edit-button"
      />
    </div>
    <div v-else>
      <textarea
        class="transcript"
        v-model.trim="editableTranscript"
        placeholder="Voice transcript"
        ref="editTranscript"
      />
      <br /><img
        src="@/assets/icons8-checkmark-48.png"
        @click="updateTranscript"
        @contextmenu.prevent
        class="edit-button"
      /><img
        src="@/assets/icons8-cancel-48.png"
        @click="cancelEdit"
        @contextmenu.prevent
        class="edit-button"
      />
    </div>
    <div class="vidnote">
      <video :src="vidNoteFileUrl" controls></video>
    </div>
    <div>
      Voice effect:
      <select v-model="chosenVoiceEffect" @change="changeVoiceEffect">
        <option
          v-for="effect in voiceEffects"
          :key="effect"
          :value="effect.value"
        >
          {{ effect.display_name }}
        </option>
      </select>
    </div>
    <br />
    <div v-if="shareable">
      <img
        src="@/assets/icons8-send-48.png"
        @click="send"
        @contextmenu.prevent
        class="send-button"
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
      vidNoteFileUrl: null,
      vidNoteFile: null,
      shareFiles: null,
      shareable: false,
      chosenVoiceEffect: "",
      voiceEffects: [{ value: "", display_name: "" }],
      chosenLanguage: "en",
      languages: [],
    };
  },
  watch: {
    vidNoteFileUrl() {
      this.shareFiles = {
        files: [this.vidNoteFile],
      };
      this.shareable =
        typeof navigator.share === "function" &&
        navigator.canShare(this.shareFiles);
    },
  },
  methods: {
    changeLanguage: function () {
      this.transcribe();
    },
    changeVoiceEffect: function () {
      this.makeVidNote(
        this.recordingFile,
        this.transcript,
        this.chosenVoiceEffect
      );
    },
    send: function () {
      navigator.share(this.shareFiles);
    },
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
      this.chosenVoiceEffect = "";
    },
    makeVidNote: function (audioFile, transcript, effect) {
      const formData = new FormData();
      formData.append("audio", audioFile);
      formData.append("transcript", transcript);
      formData.append("effect", effect);
      axios
        .post(process.env.VUE_APP_BACKEND_URL + "/vidnote/", formData, {
          headers: {
            "Content-Type": "multipart/form-data",
          },
          responseType: "blob",
        })
        .then((response) => {
          this.vidNoteFile = new File([response.data], "result.mp4", {
            type: "video/mp4",
          });
          if (this.vidNoteFileUrl) {
            URL.revokeObjectURL(this.vidNoteFileUrl);
          }
          this.vidNoteFileUrl = URL.createObjectURL(this.vidNoteFile);
        })
        .catch(function (error) {
          console.log(error);
        });
    },
    transcribe: function () {
      const formData = new FormData();
      formData.append("audio", this.recordingFile);
      formData.append("language", this.chosenLanguage);
      axios
        .post(process.env.VUE_APP_BACKEND_URL + "/transcribe/", formData, {
          headers: {
            "Content-Type": "multipart/form-data",
          },
          responseType: "json",
        })
        .then((response) => {
          this.transcript = response.data.transcript;
          this.makeVidNote(
            this.recordingFile,
            this.transcript,
            this.chosenVoiceEffect
          );
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
      this.makeVidNote(
        this.recordingFile,
        this.transcript,
        this.chosenVoiceEffect
      );
    },
  },
  created() {
    axios
      .options(process.env.VUE_APP_BACKEND_URL + "/vidnote/", {
        responseType: "json",
      })
      .then((response) => {
        this.voiceEffects.push(...response.data.actions.POST.effect.choices);
      })
      .catch(function (error) {
        console.log(error);
      });
    axios
      .options(process.env.VUE_APP_BACKEND_URL + "/transcribe/", {
        responseType: "json",
      })
      .then((response) => {
        this.languages.push(...response.data.actions.POST.language.choices);
      })
      .catch(function (error) {
        console.log(error);
      });
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
.send-button {
  cursor: pointer;
  transition: 0.2s;
}
.send-button:hover {
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
@media (orientation: portrait) {
  .transcript {
    width: 100%;
    resize: none;
    height: 120px;
  }
}
@media (orientation: landscape) {
  .transcript {
    width: 25%;
    resize: none;
    height: 120px;
  }
}
.vidnote {
  transform: scale(0.8);
  display: flex;
  justify-content: center;
}
textarea {
  font-size: 18px;
}
</style>