<template>
  <div class="dynamic-player">
    <swiper-container
      ref="swiperEl"
      loop
      noSwiping
      init="false"
      navigation="true"
      pagination="true"
      @init="onInit"
      @slidechange="onSlideChange"
    >
      <div slot="container-start" class="p-5 text-white text-sm">
        To: Recipient
      </div>
      <swiper-slide v-for="(slide, index) in slides" :key="index">
        <video
          ref="videoRefs"
          width="600"
          height="400"
          playsinline
          autobuffer
          mute
          :id="'video-' + index"
        >
          <source :src="slide.src" type="video/mp4" />
        </video>
      </swiper-slide>
    </swiper-container>
    <button @click="playVideo" class="btn btn-primary btn-lg">Play</button>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUpdate } from 'vue';
import { register } from 'swiper/element/bundle';
register();

// Ref for swiper element
const swiperEl = ref(null);
const videoRefs = ref([]);
const slides = [
  {
    src: 'intro.mp4',
  },
  {
    src: 'jimmy.mp4',
  },
  {
    src: 'luke.mp4',
  },
];

onMounted(() => {
  // Swiper parameters
  const swiperParams = {
    autoplay: false,
    loop: true,
    injectStyles: [
      `
        :root {--swiper-theme-color: #fff;}
        swiper-container {height:100%;background-color:#202124;}
        swiper-slide {display:flex;align-items:center;justify-content:center;color:#fff;}
      `,
    ],
  };

  // Assign all parameters to Swiper element and initialize it
  Object.assign(swiperEl.value, swiperParams);
  swiperEl.value.initialize();
});

// Methods
const onInit = (e) => {
  console.log('swiper initialized');
};

/*const onSlideChange = (e) => {
  const detail = e.detail[0];
  const index = detail.realIndex;
  if (index >= 0 && index < videoRefs.value.length) {
    const currentVideo = videoRefs.value[index];
    console.log('Current video:', currentVideo);
    console.log('Current index:', index);
  }
};*/

const onSlideChange = (e) => {
  console.log('slide changed');
  const currentVideo = videoRefs.value[e.detail[0].realIndex];
  currentVideo.muted = false;
  currentVideo.play();
  const currentIndex = e.detail[0].realIndex;
  currentVideo.addEventListener('ended', () => {
    const nextIndex = currentIndex + 1;
    const nextVideo = videoRefs.value[nextIndex];
    if (nextVideo) {
      currentVideo.pause();
      nextVideo.muted = false;
      nextVideo.play();
      swiperEl.value.swiper.slideNext();
    } else {
      swiperEl.value.swiper.slideTo(0);
    }
  });
};

const playVideo = () => {
  if (videoRefs.value.length > 0) {
    const firstVideo = videoRefs.value[0];
    firstVideo.muted = false;
    firstVideo.play();
    firstVideo.addEventListener('ended', () => {
      swiperEl.value.swiper.slideNext();
    });
  }
};
</script>

<style scoped>
body {
  margin: 0;
}
.dynamic-player {
  height: 100vh;
}
</style>
