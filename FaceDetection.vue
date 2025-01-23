<template>
  <section id="demos wd-100">
    <button id="webcamButton" class="mdc-button mdc-button--raised" @click="enableCam">
      <span class="mdc-button__ripple"></span>
      <span class="mdc-button__label">ENABLE WEBCAM</span>
    </button>
    <div id="liveView" class="videoView">
      <!-- <div id="detectionGuide"  v-if="isGuideVisible"></div> -->
      <video id="webcam" autoplay playsinline></video>
    </div>
  </section>
  <div id="photoSection">
    <h2>Photo Captured</h2>
    <div id="photoContainer"></div>
  </div>
</template>

<script>
import {
  ObjectDetector,
  FilesetResolver,
  Detection,
  ObjectDetectionResult
} from "@mediapipe/tasks-vision";

export default {
  data() {
    return {
      objectDetector: null,
      runningMode: "IMAGE",
      lastVideoTime: -1,
      children: [],
      isGuideVisible: false,
      images: [
        { id: 1, src: "https://assets.codepen.io/9177687/coupledog.jpeg" },
        { id: 2, src: "https://raw.githubusercontent.com/Zabdi0819/test_model/main/7.jpeg" },
      ], // Replace with actual images
    };
  },
  methods: {
    async initializeObjectDetector() {
      const vision = await FilesetResolver.forVisionTasks(
        "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.2/wasm"
      );
      this.objectDetector = await ObjectDetector.createFromOptions(vision, {
        baseOptions: {
          modelAssetPath: `https://raw.githubusercontent.com/Zabdi0819/test_model/main/model_ine_v2.tflite`,
          delegate: "GPU",
        },
        scoreThreshold: 0.87,
        runningMode: this.runningMode,
        maxResults: 1,
      });
      document.getElementById("demos").classList.remove("invisible");
    },
    async handleClick(event) {
      const container = event.target.parentNode;
      this.clearHighlighters(container);

      if (!this.objectDetector) {
        alert("Object Detector is still loading. Please try again.");
        return;
      }

      if (this.runningMode === "VIDEO") {
        this.runningMode = "IMAGE";
        await this.objectDetector.setOptions({ runningMode: "IMAGE" });
      }

      const ratio = event.target.height / event.target.naturalHeight;
      const detections = await this.objectDetector.detect(event.target);
      this.displayImageDetections(detections, event.target, ratio);
    },
    displayImageDetections(result, resultElement, ratio) {
      for (let detection of result.detections) {
        const p = document.createElement("p");
        p.setAttribute("class", "info");
        p.innerText =
          detection.categories[0].categoryName +
          " - with " +
          Math.round(parseFloat(detection.categories[0].score) * 100) +
          "% confidence.";
        p.style = `
          left: ${detection.boundingBox.originX * ratio}px;
          top: ${detection.boundingBox.originY * ratio}px;
          width: ${detection.boundingBox.width * ratio - 10}px;
        `;
        const highlighter = document.createElement("div");
        highlighter.setAttribute("class", "highlighter");
        highlighter.style = `
          left: ${detection.boundingBox.originX * ratio}px;
          top: ${detection.boundingBox.originY * ratio}px;
          width: ${detection.boundingBox.width * ratio}px;
          height: ${detection.boundingBox.height * ratio}px;
        `;
        resultElement.parentNode.appendChild(highlighter);
        resultElement.parentNode.appendChild(p);
      }
    },
    clearHighlighters(container) {
      const highlighters = container.getElementsByClassName("highlighter");
      while (highlighters[0]) {
        highlighters[0].parentNode.removeChild(highlighters[0]);
      }
      const infos = container.getElementsByClassName("info");
      while (infos[0]) {
        infos[0].parentNode.removeChild(infos[0]);
      }
    },
    async enableCam() {
      if (!this.objectDetector) {
        console.log("Wait! objectDetector not loaded yet.");
        return;
      }
      const enableWebcamButton = document.getElementById("webcamButton");
      enableWebcamButton.classList.add("removed");

      const constraints = { video: true };
      try {
        const stream = await navigator.mediaDevices.getUserMedia(constraints);
        const video = document.getElementById("webcam");
        video.srcObject = stream;
        video.addEventListener("loadeddata", this.predictWebcam);
      } catch (err) {
        console.error(err);
      }
    },
    async predictWebcam() {
      const video = document.getElementById("webcam");
      if (this.runningMode === "IMAGE") {
        this.runningMode = "VIDEO";
        await this.objectDetector.setOptions({ runningMode: "VIDEO" });
      }

      const startTimeMs = performance.now();
      if (video.currentTime !== this.lastVideoTime) {
        this.lastVideoTime = video.currentTime;

        const detections = await this.objectDetector.detectForVideo(
          video,
          startTimeMs
        );
        this.displayVideoDetections(detections);
        if (detections.detections.length > 0) {
          const detectedObject = detections.detections[0].categories[0].categoryName;
          this.handleDetection(detectedObject, video);
        } else {
          this.clearDetectionTimer();
        }
      }
      
      window.requestAnimationFrame(this.predictWebcam);
    },
    handleDetection(detectedObject, video) {
      if (this.detectionTimer && this.lastDetectedObject === detectedObject) {
        this.isGuideVisible = true;

        const elapsedTime = performance.now() - this.detectionStartTime;
        if (elapsedTime >= 2000) {
          this.capturePhoto(video);
          this.clearDetectionTimer();
        }
      } else {
        this.lastDetectedObject = detectedObject;
        this.detectionStartTime = performance.now();
        if (!this.detectionTimer) {
          this.detectionTimer = setTimeout(() => {}, 2000);
        }
      }
    },
    clearDetectionTimer() {
      clearTimeout(this.detectionTimer);
      this.detectionTimer = null;
      this.lastDetectedObject = null;
      this.detectionStartTime = null;
    },
    capturePhoto(video) {
      const canvas = document.createElement("canvas");
      const context = canvas.getContext("2d");
      canvas.width = video.videoWidth;
      canvas.height = video.videoHeight;

      context.drawImage(video, 0, 0, canvas.width, canvas.height);

      const photoURL = canvas.toDataURL("image/jpeg");

      this.showCapturedPhoto(photoURL);
    },
    showCapturedPhoto(photoURL) {
      const photoContainer = document.getElementById("photoContainer");
      const img = document.createElement("img");
      img.src = photoURL;
      img.alt = "Captured photo";
      img.style.width = "80%";
      img.style.marginTop = "20px";

      photoContainer.innerHTML = ""; 
      photoContainer.appendChild(img);
    },
    displayVideoDetections(result) {
      const liveView = document.getElementById("liveView");

      // Elimina los elementos previos
      for (let child of this.children) {
        liveView.removeChild(child);
      }
      this.children.splice(0);

      const video = document.getElementById("webcam");
      const ratio = video.videoHeight / video.videoHeight;

      // Dibuja los resultados de las detecciones
      for (let detection of result.detections) {
        const p = document.createElement("p");
        p.setAttribute("class", "info");
        p.innerText =
          detection.categories[0].categoryName +
          " - with " +
          Math.round(parseFloat(detection.categories[0].score) * 100) +
          "% confidence.";
        p.style = `
          left: ${detection.boundingBox.originX * ratio}px;
          top: ${detection.boundingBox.originY * ratio - 20}px;
          width: ${detection.boundingBox.width * ratio - 10}px;
          position: absolute;
          color: white;
          background-color: rgba(0, 0, 0, 0.5);
          padding: 2px;
          font-size: 12px;
        `;
        const highlighter = document.createElement("div");
        highlighter.setAttribute("class", "highlighter");
        highlighter.style = `
          left: ${detection.boundingBox.originX * ratio}px;
          top: ${detection.boundingBox.originY * ratio}px;
          width: ${detection.boundingBox.width * ratio}px;
          height: ${detection.boundingBox.height * ratio}px;
          position: absolute;
          border: 2px solid #00FF00;
          box-shadow: 0 0 10px rgba(0, 255, 0, 0.5);
        `;
        liveView.appendChild(highlighter);
        liveView.appendChild(p);
        this.children.push(highlighter, p);
  }
}

  },
  mounted() {
    this.initializeObjectDetector();
  },
};
</script>

<style>
body {
  font-family: roboto;
  margin: 2em;
  color: #3d3d3d;
  --mdc-theme-primary: #007f8b;
  --mdc-theme-on-primary: #f1f3f4;
}

h1 {
  color: #007f8b;
}

h2 {
  clear: both;
}

em {
  font-weight: bold;
}

video {
  clear: both;
  display: block;
  margin: auto;
  width: 60%;
  border: solid 1px;
    margin: auto;
    left: 0px;
    right: 0px;
    top: 0px;
    bottom: 0px;
    position: absolute;
    border-radius: 100%;
    transition: linear 0.5s;
    
  /* animation-name: loading_scanner;
  animation-duration: 1.5s;
  animation-iteration-count: infinite;
  animation-timing-function: linear; */
}

/* @keyframes loading_scanner {
  from {width: 0%;height: 0%;border-radius: 100%;color: rgba(0, 255, 76, 0.493);}
  to {width: 100%;height: 100%;border-radius: 0%;color:rgba(0, 128, 0, 0);}
} */

section {
  opacity: 1;
  transition: opacity 500ms ease-in-out;
  width: 100%;
  background-color: bisque;
}

.mdc-button.mdc-button--raised.removed {
  display: none;
}

.invisible {
  opacity: 0.2;
}

.videoView{
  transform: scale(3.15);

  position: relative;
  width: 80px;
  height: 50px;
  margin: auto;
  cursor: pointer;
  background-color: #3d3d3d;

  overflow: hidden;
  backdrop-filter: brightness(1.1);
      /* position: absolute;
      left: 0px;
      right: 0px;
      top: 0px;
      bottom: 0px; */
      box-shadow: 0px 0px 60px 192px #101010c7;
      background-image: url("data:image/svg+xml,%3csvg width='100%25' height='100%25' xmlns='http://www.w3.org/2000/svg'%3e%3crect width='100%25' height='100%25' fill='none' stroke='%2300A960FF' stroke-width='2' stroke-dasharray='10%2c40%2c10%2c40' stroke-dashoffset='5' stroke-linecap='round'/%3e%3c/svg%3e");

}

.detectOnClick p {
  position: absolute;
  padding: 5px;
  background-color: #007f8b;
  color: #fff;
  border: 1px dashed rgba(255, 255, 255, 0.7);
  z-index: 2;
  font-size: 12px;
  margin: 0;
}

.videoView p {
  position: absolute;
  padding-bottom: 5px;
  padding-top: 5px;
  background-color: #007f8b;
  color: #fff;
  border: 1px dashed rgba(255, 255, 255, 0.7);
  z-index: 2;
  font-size: 12px;
  margin: 0;
}

.info {
  position: absolute;
  background-color: rgba(0, 0, 0, 0.5);
  color: white;
  font-size: 12px;
  padding: 2px 4px;
  border-radius: 3px;
}

#detectionGuide {
  position: absolute;
  top: 50%;
  left: 50%;
  width: 250px; /* Ajusta el tamaño del rectángulo */
  height: 150px; /* Ajusta el tamaño del rectángulo */
  transform: translate(-30%, -20%);
  border: 5px solid #022a5c;
  border-radius: 8px; /* Opcional: esquinas redondeadas */
  pointer-events: none; /* Permite interactuar con el video debajo */
  z-index: 2; /* Asegura que esté por encima del video */
}

.highlighter {
  position: absolute;
  border: 2px solid #00FF00;
  pointer-events: none;
}

.detectOnClick {
  z-index: 0;
}

.detectOnClick img {
  width: 100%;
}

.key-point {
  position: absolute;
  z-index: 1;
  width: 3px;
  height: 3px;
  background-color: #ff0000;
  /* border: 1px solid #ffffff; */
  border-radius: 50%;
  display: block;
}

#photoSection {
  margin-top: 20px;
}

#photoContainer img {
  border: 2px solid #007f8b;
  border-radius: 8px;
}
</style>
