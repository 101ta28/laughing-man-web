<template>
  <v-container>
    <div v-if="!hasUserMedia" class="text-center">
      <v-alert type="error">
        お使いのブラウザでは getUserMedia() がサポートされていません。
      </v-alert>
    </div>
    <div v-else>
      <v-btn color="primary" @click="enableCam" v-if="!cameraEnabled">
        カメラを有効にする
      </v-btn>
      <div v-show="cameraEnabled" style="position: relative;">
        <video id="webcam" style="width: 100%; max-width: 640px; height: auto;" autoplay playsinline></video>
        <div id="liveView" style="position: absolute; top: 0; left: 0; width: 100%; height: 100%;">
          <!-- バウンディングボックスをここに表示 -->
          <div v-for="(prediction, index) in predictions" :key="index" class="highlighter" :style="getStyle(prediction)">
          </div>
        </div>
      </div>
      <v-btn color="success" @click="saveImage" v-if="cameraEnabled">
        Save Image
      </v-btn>
    </div>
  </v-container>
  <v-footer app>
    <v-spacer />
    <span>X:
      <a href="https://twitter.com/101ta28" target="_blank" rel="noopener noreferrer">
        @101ta28
      </a>
    </span>
  </v-footer>
</template>

<script setup>
import { FaceDetector, FilesetResolver } from '@mediapipe/tasks-vision';
import { onMounted, reactive, ref } from 'vue';

const cameraEnabled = ref(false);
const hasUserMedia = ref(!!navigator.mediaDevices?.getUserMedia);
const predictions = reactive([]);

let faceDetector;

async function initializeFaceDetector() {
  const vision = await FilesetResolver.forVisionTasks('https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@latest/wasm');
  faceDetector = await FaceDetector.createFromOptions(vision, {
    baseOptions: {
      modelAssetPath: 'https://storage.googleapis.com/mediapipe-models/face_detector/blaze_face_short_range/float16/1/blaze_face_short_range.tflite',
      delegate: 'GPU',
    },
    runningMode: 'VIDEO'
  });
}

async function enableCam() {
  if (!faceDetector) {
    alert("Face Detector is still loading. Please try again later.");
    return;
  }
  try {
    const stream = await navigator.mediaDevices.getUserMedia({ video: true });
    const videoElement = document.querySelector('#webcam');
    videoElement.srcObject = stream;
    cameraEnabled.value = true;
    predictWebcam();
  } catch (err) {
    console.error(err);
    alert("Webcam cannot be accessed.");
  }
}

async function predictWebcam() {
  if (!faceDetector || !cameraEnabled.value) {
    return;
  }

  const videoElement = document.getElementById('webcam');
  let startTimeMs = performance.now();

  if (videoElement.readyState !== videoElement.HAVE_ENOUGH_DATA) {
    requestAnimationFrame(predictWebcam);
    return;
  }

  try {
    const results = await faceDetector.detectForVideo(videoElement, startTimeMs);
    // ここでスプレッド構文を使用する前に、results.detections を確認します
    if (results && results.detections) {
      predictions.splice(0, predictions.length, ...results.detections);
    }
  } catch (error) {
    console.error(error);
  }

  requestAnimationFrame(predictWebcam);
}


const coverImageUrl = new URL('../assets/laughing_man_noise.gif', import.meta.url).href;

function getStyle(prediction) {
  const videoElement = document.getElementById('webcam');
  const scaleX = videoElement.offsetWidth / videoElement.videoWidth;
  const scaleY = videoElement.offsetHeight / videoElement.videoHeight;

  const padding = 150; // バウンディングボックスをどれだけ大きくするか
  const additionalTopOffset = 20; // 画像を上に移動させる追加のオフセット量

  const left = (prediction.boundingBox.originX - padding / 2) * scaleX;
  // 画像を上に少し移動させるために additionalTopOffset を減算
  const top = (prediction.boundingBox.originY - padding / 2 - additionalTopOffset) * scaleY;
  const width = (prediction.boundingBox.width + padding) * scaleX;
  const height = (prediction.boundingBox.height + padding) * scaleY;

  return {
    position: 'absolute',
    left: `${left}px`,
    top: `${top}px`,
    width: `${width}px`,
    height: `${height}px`,
    backgroundImage: `url(${coverImageUrl})`,
    backgroundSize: 'cover', // 画像がバウンディングボックスを覆うように調整
    backgroundPosition: 'center',
  };
}

async function saveImage() {
  const videoElement = document.getElementById('webcam');
  const canvas = document.createElement('canvas');
  canvas.width = videoElement.videoWidth;
  canvas.height = videoElement.videoHeight;
  const ctx = canvas.getContext('2d');

  // ビデオの現在のフレームをキャンバスに描画
  ctx.drawImage(videoElement, 0, 0, canvas.width, canvas.height);

  // 各バウンディングボックスに対応する画像を非同期にロード
  const imageLoadPromises = predictions.map(prediction => {
    return new Promise(resolve => {
      const scaleX = canvas.width / videoElement.offsetWidth;
      const scaleY = canvas.height / videoElement.offsetHeight;
      const padding = 150;
      const additionalTopOffset = 20;
      const left = (prediction.boundingBox.originX - padding / 2) * scaleX;
      const top = (prediction.boundingBox.originY - padding / 2 - additionalTopOffset) * scaleY;
      const width = (prediction.boundingBox.width + padding) * scaleX;
      const height = (prediction.boundingBox.height + padding) * scaleY;

      const img = new Image();
      img.src = coverImageUrl;
      img.onload = () => {
        ctx.drawImage(img, left, top, width, height);
        resolve();
      };
    });
  });

  // すべての画像のロードが完了したら、キャンバスの内容を保存
  await Promise.all(imageLoadPromises).then(() => {
    const imageURL = canvas.toDataURL('image/png');
    const link = document.createElement('a');
    link.href = imageURL;
    link.download = 'captured-image.png';
    link.click();
  });
}



onMounted(() => {
  initializeFaceDetector();
});
</script>

<style>
.highlighter {
  box-sizing: border-box;
}
</style>
