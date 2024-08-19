<template>
  <div>
    <div class="w-full flex justify-center items-center flex-col px-12 pt-4">
      <p class="text-white text-2xl font-bold mb-2">
        Local Video Compressor - No Upload Required!
      </p>
      <p class="text-white text-lg">
        Unlike most online compressors that upload videos to a server, this tool
        processes everything locally, ensuring quick and private compression
        even with a slow internet connection.
      </p>
    </div>
    <div v-show="showVideoEditing">
      <div v-show="!isProcessing" class="flex flex-col justify-center items-center p-8">
        <video ref="playerRef" class="video-js cursor-pointer" width="500" @click="togglePlayPause"></video>

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

        <div class="w-full h-16 rounded-sm bg-gray-700 mt-4 relative" ref="timeline">
          <div class="w-2 h-full absolute bg-red-200 cursor-move z-10 opacity-50" :style="{ left: `${leftHandleX}px` }"
            ref="leftHandle" />
          <div class="w-2 h-full absolute bg-red-200 cursor-move z-10 opacity-50" :style="{ left: `${rightHandleX}px` }"
            ref="rightHandle" />

          <div class="absolute h-full bg-blue-200 opacity-50 cursor-move" :style="{
            left: `${inbetweenBoxLeft}px`,
            width: `${inbetweenBoxWidth}px`,
          }" ref="inbetweenRect" />

          <div class="absolute w-0.5 h-full bg-red-600" :style="{ left: `${redLineX}%` }" />


        </div>

        <div class="flex items-center mt-4">
          <label class="ml-4 mr-2">Target Size (MB):</label>
          <input type="number" v-model="targetSizeMB" class="input input-bordered" min="1" step="1" />
        </div>

        <button class="btn btn-primary mt-4" @click="() => processVideo()">
          Compress
        </button>

        <button class="btn btn-secondary mt-4" @click="reset">
          Choose another file
        </button>
      </div>
      <div v-show="isProcessing" class="flex flex-col justify-center items-center p-8">
        <p class="text-2xl">Processing...</p>
        <p class="text-lg">{{ `Attempt ${iteration + 1}: Trying resolution ${resolution
          }p and bitrate ${bitrate} kbps` }}</p>

        <progress class="progress progress-primary w-96 mt-3" :value="processProgress" max="100"></progress>
      </div>
    </div>

    <div v-show="!showVideoEditing" class="flex flex-col justify-center items-center p-8">
      <input accept=".mkv, .mp4" type="file" ref="fileInput" class="file-input file-input-bordered w-full max-w-xs"
        @change="onFileChanged" />
      <div class="mt-4 w-full max-w-lg border-2 border-dashed border-gray-500 p-8 text-center rounded-md cursor-pointer"
        @dragover.prevent @drop.prevent="handleDrop" @click="handleClick">
        <p class="text-gray-500">
          Drag and drop video file here, or click to select files
        </p>
      </div>

      <div class="mt-4 flex flex-col justify-center items-center" v-if="mkvDetected">
        <p class="text-lg text-red-600">Stupid Mkv file detected, wait a bit...</p>
        <progress class="progress w-56 progress-primary mt-4" />
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

const targetSizeMB = ref(25);
const resolution = ref("720");
const bitrate = ref(200);
const videoSize = ref(0);
const selectedStartTime = ref(0);
const selectedDuration = ref(0);
const isPlaying = ref(true);

const iteration = ref(0);

const mkvDetected = ref(false)
const baseURL = 'https://unpkg.com/@ffmpeg/core-mt@0.12.6/dist/esm'

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
  if (!showVideoEditing.value) return;
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

async function handleFiles(file: File) {
  // Check if the file is an MKV
  if (file.type === "video/x-matroska" || file.name.endsWith(".mkv")) {
    mkvDetected.value = true;
    // Convert MKV to MP4 using FFmpeg
    const ffmpeg = new FFmpeg();

    const reader = new FileReader();
    reader.readAsArrayBuffer(file);
    reader.onload = async function (event) {
      const videoData = new Uint8Array(event.target!.result as ArrayBuffer);

      ffmpeg.on("log", (y: any) => {
        console.log(y.message);
      });

      await ffmpeg.load({
        coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, "text/javascript"),
        wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, "application/wasm"),
        workerURL: await toBlobURL(
          `${baseURL}/ffmpeg-core.worker.js`,
          "text/javascript"
        ),
      });

      await ffmpeg.writeFile("input.mkv", videoData);

      await ffmpeg.exec([
        "-i",
        "input.mkv",
        "-c",
        "copy", // Copy streams without re-encoding
        "output.mp4",
      ]);

      const data = (await ffmpeg.readFile("output.mp4")) as any;
      const videoBlob = new Blob([data.buffer], { type: "video/mp4" });
      const mp4File = new File([videoBlob], "converted.mp4", {
        type: "video/mp4",
      });

      // Proceed with the converted MP4 file
      processVideoFile(mp4File);
    };
  } else {
    // If the file is not MKV, proceed directly
    processVideoFile(file);
  }
}

function processVideoFile(file: File) {
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

async function compressVideo(ffmpeg: any) {
  const resolutionOption: any = {
    "64": "scale=128:-1",   // 64p
    "144": "scale=256:-1",  // 144p
    "240": "scale=426:-1",  // 240p
    "360": "scale=640:-1",  // 360p
    "480": "scale=854:-1",  // 480p
    "720": "scale=1280:-1", // 720p
  };


  const ffmpegArgs = [
    "-ss",
    selectedStartTime.value.toString(),
    "-i",
    "input.mp4",
    "-vf",
    resolutionOption[resolution.value],
    "-t",
    selectedDuration.value.toString(),
    "-crf",
    "32", // You can also try lower values like 28 to further reduce size
    "-preset",
    "ultrafast", // `slow` or `medium` will give better compression at the cost of speed
    "-threads",
    "4",
    "-f",
    "mp4",
    "output.mp4",
  ];



  console.log('executing')
  await ffmpeg.exec(ffmpegArgs);
}

async function processVideo(maxIterations = 5) {
  isProcessing.value = true;

  iteration.value = 0;
  let fileSize = videoSize.value;
  const ffmpeg = new FFmpeg();
  ffmpeg.on("log", (y: any) => {
    console.log(y)
    const timeMatch = y.message.match(/time=(\d{2}):(\d{2}):(\d{2}\.\d{2})/);

    if (timeMatch) {
      const hours = parseInt(timeMatch[1], 10);
      const minutes = parseInt(timeMatch[2], 10);
      const seconds = parseFloat(timeMatch[3]);

      // Convert the time to total seconds
      const totalTimeInSeconds = hours * 3600 + minutes * 60 + seconds;
      processProgress.value = (totalTimeInSeconds / selectedDuration.value) * 100;
    }
  });

  await ffmpeg.load({
    coreURL: await toBlobURL(`${baseURL}/ffmpeg-core.js`, "text/javascript"),
    wasmURL: await toBlobURL(`${baseURL}/ffmpeg-core.wasm`, "application/wasm"),
    workerURL: await toBlobURL(
      `${baseURL}/ffmpeg-core.worker.js`,
      "text/javascript"
    ),
  });

  const reader = new FileReader();
  reader.readAsArrayBuffer(videoFile.value);
  reader.onload = async function (event) {
    const videoData = new Uint8Array(event.target!.result as ArrayBuffer);
    let data: any
    await ffmpeg.writeFile("input.mp4", videoData);

    while (iteration.value < maxIterations) {
      // Compress the video with the current resolution and bitrate
      await compressVideo(ffmpeg);


      data = (await ffmpeg.readFile("output.mp4")) as any;
      fileSize = await data.length;

      console.log(
        `Output file size: ${(fileSize / (1024 * 1024)).toFixed(2)} MB`
      );

      // Check if the file size is within the acceptable range of the target size
      if (fileSize <= targetSizeMB.value * 1024 * 1024) {
        console.log("Target size achieved!");
        break;
      }

      // If the file size is too large, reduce bitrate and resolution
      bitrate.value *= 0.8;

      const resolutions = ["720", "480", "360", "240", "144", "64"];
      const currentIndex = resolutions.indexOf(resolution.value);

      resolution.value =
        currentIndex < resolutions.length - 1
          ? resolutions[currentIndex + 1]
          : "240";

      iteration.value++;
    }

    // Read the processed video from FFmpeg's virtual filesystem
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
  }
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
  mkvDetected.value = false;

}
</script>

<style scoped></style>
