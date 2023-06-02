<template>
  <div class="dynamic-player">
    <swiper-container
      ref="swiperEl"
      noSwiping
      init="false"
      navigation="true"
      pagination="true"
      @init="onInit"
      @slidechange="onSlideChange"
    >
      <!-- Video header with Tribute info and mute button -->
      <div
        slot="container-start"
        class="
          relative
          z-10
          flex
          items-center
          justify-between
          p-5
          text-white text-sm
          font-semibold
        "
      >
        <!-- Tribute info -->
        <div>To: Recipient</div>

        <!-- Overaly for mobile interaction -->
        <div v-show="showOverlay" class="overlay">
          <div class="tooltip">Please unmute before playing.</div>
        </div>

        <!-- Mute/Unmute button -->
        <button
          @click="toggleMute"
          class="flex items-center justify-center z-10"
        >
          <img
            :src="isMuted ? volumeOffImage : volumeOnImage"
            class="w-6 h-6"
            alt="Mute/Unmute Button"
          />
        </button>
      </div>

      <!-- Swiper slides -->
      <swiper-slide v-for="(slide, index) in slides" :key="index">
        <video
          ref="videoRefs"
          width="600"
          height="400"
          playsinline
          autobuffer
          :muted="isMuted"
          :id="'video-' + index"
        >
          <source :src="slide.src" type="application/x-mpegURL" />
        </video>
        <button
          @click="togglePlay"
          :class="{ 'hide-unless-hovered': hasStartedPlaying }"
          class="
            absolute
            top-1/2
            right-0
            left-0
            flex
            w-16
            h-16
            items-center
            justify-center
            m-auto
            bg-white
            rounded-full
            -translate-y-1/2
          "
        >
          <img
            :src="isPlaying ? pauseImage : playImage"
            class="w-8 h-8"
            :class="{ 'ml-1.5': !isPlaying }"
            alt="Play/Pause Button"
          />
        </button>
      </swiper-slide>
    </swiper-container>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted, nextTick } from 'vue';
import { register } from 'swiper/element/bundle';
import Hls from 'hls.js';
register();

// Mobile unmute helper
const showOverlay = ref(false);

// Ref for images
const playImage = ref('play.svg');
const pauseImage = ref('pause.svg');
const volumeOnImage = ref('volume-on.svg');
const volumeOffImage = ref('volume-off.svg');

// Ref for swiper element
const swiperEl = ref(null);
const videoRefs = ref([]);
const currentPlayingIndex = ref(null);
const isPlaying = ref(false);
const isMuted = ref(true);
const hasStartedPlaying = ref(false);
const slides = [
  {
    src: 'https://player.vimeo.com/external/823050002.m3u8?s=3f3e42fa89c4193ccc3e30707faf593eccbb9696',
  },
  {
    src: 'https://player.vimeo.com/external/832639412.m3u8?s=af15996d49ebcb49d9444c3547919ad70256da15',
  },
  {
    src: 'https://player.vimeo.com/external/832639348.m3u8?s=3f11e4fe435908857eb79bc44a44ddcd16a5733e',
  },
];

onMounted(() => {
  // Show tooltip helper to unmute on mobile
  showOverlay.value = window.innerWidth < 768 && isMuted.value;
  window.addEventListener('resize', () => {
    showOverlay.value = window.innerWidth < 768 && isMuted.value;
  });

  // Swiper parameters
  const swiperParams = {
    autoplay: false,
    injectStyles: [
      `
        :root {--swiper-theme-color: #fff;}
        swiper-container {height:100%;background-color:#202124;}
        swiper-slide {display:flex;align-items:center;justify-content:center;color:#fff;}
        .swiper-button-next,.swiper-button-prev {z-index: 9;}
      `,
    ],
  };

  // Assign all parameters to Swiper element and initialize it
  Object.assign(swiperEl.value, swiperParams);
  swiperEl.value.initialize();

  // Iterate over slides
  slides.forEach((slide, index) => {
    // HLS
    if (Hls.isSupported()) {
      var hls = new Hls();
      hls.loadSource(slide.src);
      hls.attachMedia(videoRefs.value[index]);
      videoRefs.value[index].onended = async () => {
        if (swiperEl.value && swiperEl.value.swiper) {
          // stop the current video
          videoRefs.value[currentPlayingIndex.value].pause();
          videoRefs.value[currentPlayingIndex.value].currentTime = 0;

          // slide to the next video
          swiperEl.value.swiper.slideNext();

          // wait until Vue updates the DOM
          await nextTick();

          // play the new current video
          videoRefs.value[currentPlayingIndex.value].play();
        }
      };
    }
  });
});

onUnmounted(() => {
  // Remove event listener when component is unmounted to avoid memory leaks
  window.removeEventListener('resize', () => {
    showOverlay.value = window.innerWidth < 768 && isMuted.value;
  });
});

// Methods
const onInit = (e) => {
  console.log('swiper initialized');
};

const playVideo = (video) => {
  if (video) {
    //video.setVolume(isMuted.value ? 0 : 1);
    video.play().catch((error) => {
      if (
        error.name === 'NotAllowedError' ||
        error.name === 'NotSupportedError'
      ) {
        video.setVolume(0);
        video.play();
      }
    });
  }
};

const togglePlay = () => {
  if (swiperEl.value && swiperEl.value.swiper) {
    const realIndex = swiperEl.value.swiper.realIndex;
    const currentVideo = videoRefs.value[realIndex];
    if (currentVideo) {
      if (isPlaying.value) {
        currentVideo.pause();
        isPlaying.value = false;
      } else {
        playVideo(currentVideo);
        isPlaying.value = true;
        hasStartedPlaying.value = true;
        currentPlayingIndex.value = realIndex; // Add this line
      }
    }
  }
};

const toggleMute = () => {
  if (swiperEl.value && swiperEl.value.swiper) {
    const currentVideo = videoRefs.value[swiperEl.value.swiper.realIndex];
    if (currentVideo) {
      if (isMuted.value) {
        currentVideo.muted = false;
        isMuted.value = false;
      } else {
        currentVideo.muted = true;
        isMuted.value = true;
      }
      showOverlay.value = window.innerWidth < 768 && isMuted.value;
    }
  }
};

const onSlideChange = (e) => {
  console.log('slide changed');

  // pause the previous video
  if (currentPlayingIndex.value !== null) {
    const previousVideo = videoRefs.value[currentPlayingIndex.value];
    if (previousVideo) {
      previousVideo.pause();
    }
  }

  // play the current video
  const currentVideo = videoRefs.value[e.detail[0].realIndex];
  playVideo(currentVideo);
  hasStartedPlaying.value = true;

  // update the currently playing video index
  currentPlayingIndex.value = e.detail[0].realIndex;

  const currentIndex = e.detail[0].realIndex;
  currentVideo.addEventListener('ended', function () {
    const nextIndex = currentIndex + 1;
    const nextVideo = videoRefs.value[nextIndex];
    if (nextVideo) {
      currentVideo.volume = 0;
      playVideo(nextVideo);
      swiperEl.value.swiper.slideNext();
    } else {
      swiperEl.value.swiper.slideTo(0);
    }
  });
};
</script>

<style scoped>
body {
  margin: 0;
}
@media (max-width: 767px) {
  swiper-slide > div {
    width: 100%;
    overflow: hidden;
  }
}
iframe {
  display: none !important;
}
.dynamic-player {
  height: 100vh;
}
.overlay {
  position: fixed;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  background-color: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 9;
}
.tooltip {
  background-color: white;
  color: black;
  padding: 15px;
  border-radius: 5px;
  position: absolute;
  top: 60px;
  right: 10px;
}
.tooltip::after {
  content: '';
  position: absolute;
  bottom: 100%;
  right: 16px;
  margin-left: -8px;
  border-width: 8px;
  border-style: solid;
  border-color: transparent transparent white transparent;
}
.hide-unless-hovered {
  opacity: 0;
  transition: opacity 0.25s ease-in-out;
}
.hide-unless-hovered:hover,
swiper-slide > video:hover + .hide-unless-hovered {
  opacity: 1;
}
</style>
