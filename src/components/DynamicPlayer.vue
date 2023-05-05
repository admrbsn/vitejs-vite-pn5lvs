<template>
  <div class="dynamic-player">
    <audio ref="audioRef" id="audio" playsinline>
      <source
        src="https://storage.googleapis.com/tribute-music-prod/205_full_clap-tap_0102.mp3"
        type="audio/mpeg"
      />
    </audio>
    <swiper-container
      ref="swiperEl"
      noSwiping
      init="false"
      navigation="true"
      pagination="true"
      @init="onInit"
      @slidechange="onSlideChange"
    >
      <div slot="container-start" class="p-5 text-white text-sm">
        To: Recipienttt
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
const audioRef = ref(null);
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

const onSlideChange = (e) => {
  console.log('slide changed');
  const swiper = swiperEl.value.swiper;
  const currentIndex = e.detail[0].realIndex;
  const previousIndex = swiper.previousIndex;

  const previousVideo = videoRefs.value[previousIndex];
  if (previousVideo) {
    previousVideo.pause();
    previousVideo.currentTime = 0;
  }

  const currentVideo = videoRefs.value[currentIndex];
  currentVideo.muted = false;
  currentVideo.play();

  currentVideo.addEventListener('ended', () => {
    const nextIndex = (currentIndex + 1) % videoRefs.value.length;
    const nextVideo = videoRefs.value[nextIndex];
    if (nextVideo) {
      currentVideo.pause();
      currentVideo.currentTime = 0;
      nextVideo.muted = false;
      nextVideo.play();
      swiper.slideNext();
    } else {
      swiper.slideTo(0);
    }
  });
};

const playVideo = async () => {
  if (videoRefs.value.length > 0) {
    const firstVideo = videoRefs.value[0];
    firstVideo.muted = false;

    try {
      await firstVideo.play();
      if (audioRef.value) {
        setTimeout(async () => {
          audioRef.value.volume = 0.5;
          try {
            await audioRef.value.play();
          } catch (error) {
            console.error('Audio playback failed:', error);
          }
        }, 7000);
      }
    } catch (error) {
      console.error('Video playback failed yo:', error);
    }

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
