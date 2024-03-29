<template>
  <div class="container">
    <h3>Object recognition with Tensorflow</h3>
    <div class="mb-3">
      <div v-if="!isStreaming">
        <button class="btn btn-light" @click="openCamera">Open Camera</button>
      </div>
      <div v-else class="">
        <button @click="stopStreaming" class="btn btn-light">Stop Streaming</button>
        <button @click="snapshot" class="btn btn-light">Snapshot</button>
        <button @click="detect" class="btn btn-light">Detect Object</button>
      </div>
    </div>
    <div class="image-container mb-3">
      <div class="border">
        <video ref="videoRef" autoplay="true" id="videoElement" />
      </div>
      <div class="border">
        <img
          src="@/assets/tf_logo_social.png"
          ref="imgRef"
          id="imageElement"
        />
      </div>
    </div>
    <table class="table w-50 m-auto" v-if="result.length > 0">
      <thead>
        <tr>
          <th scope="col">Object</th>
          <th scope="col">Score</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="obj in result" :key="obj.class">
          <td>{{obj.class}}</td>
          <td>{{obj.score}}</td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
import { ref, onMounted, defineComponent } from 'vue'
import '@tensorflow/tfjs-backend-cpu'
import '@tensorflow/tfjs-backend-webgl'
import * as cocoSsd from '@tensorflow-models/coco-ssd'
import { capitalCase } from 'change-case'
const video = document.getElementById('videoElement')
const img = document.getElementById('imageElement')
export default defineComponent({
  name: 'App',
  setup() {
    const imgRef = ref(img)
    const videoRef = ref(video)
    const isStreaming = ref(false)
    const isLoading = ref(true)
    const result = ref([])
    let model = undefined

    onMounted(() => {
      getModel()
    })

    async function getModel() {
      model = await cocoSsd.load()
      isLoading.value = false
    }

    async function detect() {
      const img = imgRef.value
      const predictions = await model.detect(img)
      result.value = predictions
      if(!result.value.length) result.value.push({class:"No objects detected, please try again"}) 
      else result.value = result.value.map((x) => ({...x,class: capitalCase(x.class)}))

      const canvas = document.createElement('canvas')
      const ctx = canvas.getContext('2d')

      ctx.canvas.width = img.clientWidth;
      ctx.canvas.height = img.clientHeight;
      
      ctx.drawImage(img, 0, 0, img.clientWidth, img.clientHeight)
      
      ctx.font = "15px Arial";
      result.value?.forEach((x)=>{
        
        
        ctx.lineWidth = 3;
        drawStroked(ctx,`${capitalCase(x.class)} (${Math.round(x.score * 100)}%)` , x.bbox[0]+5, x.bbox[1]+20);
        ctx.strokeStyle = "#04D9FF";
        ctx.strokeRect(
        ...x.bbox
      );
      })
      const data = canvas.toDataURL('image/png')
      imgRef.value.setAttribute('src', data)
    }

    function drawStroked(ctx, text, x, y) {
        ctx.font = '15px Arial';
        ctx.strokeStyle = '#20639B';
        // ctx.lineWidth = 3;
        ctx.strokeText(text, x, y);
        ctx.fillStyle = "#FFFFFF";
        ctx.fillText(text, x, y);
    }

    async function openCamera() {
      if (navigator.mediaDevices.getUserMedia) {
        navigator.mediaDevices.getUserMedia({ video: { facingMode: "user"} }).then((stream) => {
          isStreaming.value = true
          videoRef.value.srcObject = stream
        })
      }
    }

    function stopStreaming() {
      const stream = videoRef.value.srcObject
      const tracks = stream.getTracks()
      tracks.map((track) => track.stop())
      isStreaming.value = false
    }

    function snapshot() {
      const canvas = document.createElement('canvas')
      const ctx = canvas.getContext('2d')

      ctx.drawImage(videoRef.value, 0, 0, canvas.width, canvas.height)
      const data = canvas.toDataURL('image/png')
      imgRef.value.setAttribute('src', data)
      result.value = []
    }

    return {
      imgRef,
      detect,
      result,
      isStreaming,
      videoRef,
      openCamera,
      stopStreaming,
      snapshot
    }
  }
})
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  margin-top: 50px;
}
.image-container {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 2rem;
  #imageElement {
    width: 100%;
    height: 100%;
  }
  #videoElement {
    width: 100%;
    height: 100%;
    transform: rotateY(180deg);
    -webkit-transform: rotateY(180deg); /* Safari and Chrome */
    -moz-transform: rotateY(180deg); /* Firefox */
  }
}
@media (max-width: 400px) {
    .image-container {
      grid-template-columns: 1fr;
      grid-auto-rows: 1fr;
    }
  }
</style>
