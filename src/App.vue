<template>
  <div>
    <div v-show="showVideoEditing">
      <div
        v-show="!isProcessing"
        class="flex flex-col justify-center items-center p-8"
      >
        <video
          ref="playerRef"
          class="video-js cursor-pointer"
          width="500"
          @click="togglePlayPause"
          muted
        ></video>

        <div class="flex items-center mt-4">
          <button class="btn btn-primary" @click="togglePlayPause">
            {{ isPlaying ? "Pause" : "Play " }}
          </button>
          <p class="ml-4">Total Duration: {{ formatTime(duration) }}</p>
          <p class="ml-4">Start Time: {{ formatTime(selectedStartTime) }}</p>

          <p class="ml-4">
            Selected Duration: {{ Math.round(selectedDuration) }} seconds
          </p>
          <p class="ml-4">Video Size: {{ videoSize }} MB</p>
        </div>

        <div
          class="w-full h-16 rounded-sm bg-gray-700 mt-4 relative"
          ref="timeline"
        >
          <div
            class="w-2 h-full absolute bg-red-200 cursor-move z-10 opacity-50"
            :style="{ left: `${leftHandleX}px` }"
            ref="leftHandle"
          />
          <div
            class="w-2 h-full absolute bg-red-200 cursor-move z-10 opacity-50"
            :style="{ left: `${rightHandleX}px` }"
            ref="rightHandle"
          />

          <div
            class="absolute h-full bg-blue-200 opacity-50 cursor-move"
            :style="{
              left: `${inbetweenBoxLeft}px`,
              width: `${inbetweenBoxWidth}px`,
            }"
            ref="inbetweenRect"
          />

          <div
            class="absolute w-0.5 h-full bg-red-600"
            :style="{ left: `${redLineX}%` }"
          />

          <!-- <div class="absolute w-full bottom-0">
          <p
            v-for="index in timestepCount + 1"
            :key="index"
            class="absolute mr-3 select-none"
            :style="{ left: `${(index - 1) * timestepCount}%` }"
          >
            {{ Math.round(duration * ((index - 1) / timestepCount)) }}
          </p>
        </div> -->
        </div>

        <div class="flex items-center mt-4">
          <label class="mr-2">Resolution:</label>
          <select v-model="resolution" class="select select-bordered">
            <option value="240">240p</option>
            <option value="360">360p</option>
            <option value="480">480p</option>
            <option value="720">720p</option>
          </select>

          <label class="ml-4 mr-2">Bitrate:</label>
          <input
            type="number"
            v-model="bitrate"
            class="input input-bordered"
            min="50"
            max="1000"
            step="50"
          />
          kbps
        </div>

        <button class="btn btn-primary mt-4" @click="processVideo">
          Compress
        </button>

        <button class="btn btn-secondary mt-4" @click="reset">
          Choose another file
        </button>
      </div>
      <div
        v-show="isProcessing"
        class="flex flex-col justify-center items-center p-8"
      >
        <p class="text-xl">Processing...</p>
        <progress
          class="progress progress-primary w-96 mt-3"
          :value="processProgress"
          max="100"
        ></progress>
      </div>
    </div>

    <div
      v-show="!showVideoEditing"
      class="flex flex-col justify-center items-center p-8"
    >
      <input
        accept="video/*"
        type="file"
        ref="fileInput"
        class="file-input file-input-bordered w-full max-w-xs"
        @change="onFileChanged"
      />
      <div
        class="mt-4 w-full max-w-lg border-2 border-dashed border-gray-500 p-8 text-center rounded-md cursor-pointer"
        @dragover.prevent
        @drop.prevent="handleDrop"
        @click="handleClick"
      >
        <p class="text-gray-500">
          Drag and drop video file here, or click to select files
        </p>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, nextTick } from "vue";
import videojs from "video.js";
import "video.js/dist/video-js.min.css";
import { useDraggable } from "@vueuse/core";
import { FFmpeg } from "@ffmpeg/ffmpeg";
import { toBlobURL } from "@ffmpeg/util";

const leftHandle = ref<HTMLElement | null>(null);
const rightHandle = ref<HTMLElement | null>(null);
const inbetweenRect = ref<HTMLElement | null>(null);
const timeline = ref<HTMLElement | undefined>();

const leftHandleX = ref(0);
const rightHandleX = ref(1000);
const inbetweenBoxLeft = ref(0);
const inbetweenBoxWidth = ref(100);
const redLineX = ref(0);

const videoFile = ref();
const offsetBetweenHandles = 5;

const isProcessing = ref(false);
const processProgress = ref(0);

const resolution = ref("360");
const bitrate = ref(200);
const videoSize = ref(0);
const selectedStartTime = ref(0);
const selectedDuration = ref(0);
const isPlaying = ref(true);

useDraggable(leftHandle, {
  containerElement: timeline,
  onMove: (position: any) => {
    leftHandleX.value = Math.min(
      position.x,
      rightHandleX.value - offsetBetweenHandles
    );
    calcInBetweenBoxPos();
    player.currentTime(
      (leftHandleX.value / timeline.value!.getBoundingClientRect().width) *
        player.duration()
    );
    calcSelectedStartTimeAndDuration();
  },
});

useDraggable(rightHandle, {
  containerElement: timeline,
  onMove: (position: any) => {
    rightHandleX.value = Math.max(
      position.x,
      leftHandleX.value + offsetBetweenHandles
    );
    calcInBetweenBoxPos();
    player.currentTime(
      (rightHandleX.value / timeline.value!.getBoundingClientRect().width) *
        player.duration()
    );
    calcSelectedStartTimeAndDuration();
  },
});

useDraggable(inbetweenRect, {
  containerElement: timeline,
  onMove: (position: any) => {
    const newLeftHandleX = Math.min(
      position.x,
      timeline.value!.getBoundingClientRect().width - inbetweenBoxWidth.value
    );
    const newRightHandleX =
      newLeftHandleX + inbetweenBoxWidth.value - rightHandle.value!.offsetWidth;

    if (
      newLeftHandleX >= 0 &&
      newRightHandleX <= timeline.value!.getBoundingClientRect().width
    ) {
      leftHandleX.value = newLeftHandleX;
      rightHandleX.value = newRightHandleX;
      calcInBetweenBoxPos();
      calcSelectedStartTimeAndDuration();
      player.currentTime(
        (leftHandleX.value / timeline.value!.getBoundingClientRect().width) *
          player.duration()
      );
    }
  },
});

const playerRef = ref();
let player: any;
const fileInput = ref();
const duration = ref(1);

// const timestepCount = 10;

const showVideoEditing = ref(false);

onMounted(() => {
  player = videojs(playerRef.value, {});
  player.on("loadedmetadata", () => {
    duration.value = player.duration();
    videoSize.value = Math.round(videoFile.value.size / (1024 * 1024)); // Convert size to MB
    calcSelectedStartTimeAndDuration();
  });
  player.on("timeupdate", () => {
    const timeInVideo = player.currentTime() / player.duration();
    if (
      timeInVideo >
      rightHandleX.value / timeline.value!.getBoundingClientRect().width
    )
      player.currentTime(
        (leftHandleX.value / timeline.value!.getBoundingClientRect().width) *
          player.duration()
      );
    else redLineX.value = timeInVideo * 100;
  });
  window.addEventListener("keydown", handleKeydown);
});

function handleClick() {
  fileInput.value.click();
}

function calcSelectedStartTimeAndDuration() {
  selectedStartTime.value =
    (leftHandleX.value / timeline.value!.getBoundingClientRect().width) *
    player.duration();
  selectedDuration.value =
    ((rightHandleX.value - leftHandleX.value) /
      timeline.value!.getBoundingClientRect().width) *
    player.duration();
}

function calcInBetweenBoxPos() {
  inbetweenBoxLeft.value = leftHandleX.value;
  inbetweenBoxWidth.value =
    rightHandleX.value -
    leftHandleX.value +
    rightHandle.value!.getBoundingClientRect().width;
}

function handleKeydown(event: KeyboardEvent) {
  if (!showVideoEditing.value) return
  if (event.code === "Space") {
    event.preventDefault();
    togglePlayPause();
  } else if (event.code === "Enter") {
    event.preventDefault();
    processVideo();
  }
}

function handleDrop(event: DragEvent) {
  const dt = event.dataTransfer;
  if (dt && dt.files.length > 0) {
    handleFiles(dt.files[0]);
  }
}

function onFileChanged(e: Event) {
  const target = e.target as HTMLInputElement;
  if (target.files && target.files.length > 0) {
    handleFiles(target.files[0]);
  }
}

function handleFiles(file: File) {
  videoFile.value = file;
  const url = URL.createObjectURL(file);
  player.src({ type: file.type, src: url });
  showVideoEditing.value = true;
  nextTick().then(() => {
    rightHandleX.value = timeline.value!.getBoundingClientRect().width;
    calcInBetweenBoxPos();
  });

  player.play();
  duration.value = player.duration();
}

function togglePlayPause() {
  if (player.paused()) {
    player.play();
  } else {
    player.pause();
  }
  isPlaying.value = !player.paused();
}

async function processVideo() {
  isProcessing.value = true;
  const baseURL = "https://unpkg.com/@ffmpeg/core@0.12.6/dist/esm";
  if (!videoFile.value) return;

  const start =
    (leftHandleX.value / timeline.value!.getBoundingClientRect().width) *
    player.duration();
  const end =
    (rightHandleX.value / timeline.value!.getBoundingClientRect().width) *
    player.duration();
  const duration = end - start;

  const reader = new FileReader();
  reader.readAsArrayBuffer(videoFile.value);
  reader.onload = async function (event) {
    const videoData = new Uint8Array(event.target!.result as ArrayBuffer);

    const ffmpeg = new FFmpeg();

    ffmpeg.on("log", (y: any) => {
      const timeMatch = y.message.match(/time=(\d{2}):(\d{2}):(\d{2}\.\d{2})/);

      if (timeMatch) {
        const hours = parseInt(timeMatch[1], 10);
        const minutes = parseInt(timeMatch[2], 10);
        const seconds = parseFloat(timeMatch[3]);

        // Convert the time to total seconds
        const totalTimeInSeconds = hours * 3600 + minutes * 60 + seconds;
        processProgress.value = (totalTimeInSeconds / duration) * 100;
      }
    });

    await ffmpeg.load({
      coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, "text/javascript"),
      wasmURL: await toBlobURL(
        `${baseURL}/ffmpeg-core.wasm`,
        "application/wasm"
      ),
      workerURL: await toBlobURL(
        `${baseURL}/ffmpeg-core.worker.js`,
        "text/javascript"
      ),
    });

    // Write the input video to FFmpeg's virtual filesystem
    await ffmpeg.writeFile("input.mp4", videoData);

    const resolutionOption: any = {
      "240": "scale=426:-1",
      "360": "scale=640:-1",
      "480": "scale=854:-1",
      "720": "scale=1280:-1",
    };

    // Execute FFmpeg commands to process the video
    const ffmpegArgs = [
      "-ss",
      start.toString(), // Start time
      "-i",
      "input.mp4",
      "-vf",
      resolutionOption[resolution.value], // Rescale the video
      "-t",
      duration.toString(), // Duration
      "-b:v",
      `${bitrate.value}k`, // User-specified bitrate
      "-preset",
      "ultrafast", // Faster encoding preset
      "-crf",
      "24", // Constant rate factor (lower quality for faster encoding)
      "-f",
      "mp4", // Output format
      "output.mp4",
    ];

    await ffmpeg.exec(ffmpegArgs);

    // Read the processed video from FFmpeg's virtual filesystem
    const data = (await ffmpeg.readFile("output.mp4")) as any;
    const videoBlob = new Blob([data.buffer], { type: "video/mp4" });
    const videoUrl = URL.createObjectURL(videoBlob);

    // Download the processed video
    const a = document.createElement("a");
    a.href = videoUrl;
    a.download = "output.mp4";
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    isProcessing.value = false;
    processProgress.value = 0;
  };
}

function formatTime(time: number) {
  const hours = Math.floor(time / 3600);
  const minutes = Math.floor((time % 3600) / 60);
  const seconds = Math.floor(time % 60);
  return `${hours.toString().padStart(2, "0")}:${minutes
    .toString()
    .padStart(2, "0")}:${seconds.toString().padStart(2, "0")}`;
}

function reset() {
  showVideoEditing.value = false;
  leftHandleX.value = 0;
  fileInput.value.value = null;
}
</script>

<style scoped></style>
