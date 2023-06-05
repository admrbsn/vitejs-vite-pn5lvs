<template>
  <div class="dynamic-player">
    <swiper-container
      ref="swiperEl"
      init="false"
      @init="onInit"
      @slidechange="onSlideChange"
      :class="{ intro: !hasStartedPlaying }"
    >
      <!-- Video header with Tribute info and mute button -->
      <div
        slot="container-start"
        class="
          fixed
          w-full
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

      <!-- Intro slide -->
      <swiper-slide class="intro-slide">
        <div></div>
      </swiper-slide>

      <!-- Swiper slides -->
      <swiper-slide v-for="(slide, index) in slides" :key="index">
        <video
          ref="videoRefs"
          width="600"
          height="400"
          playsinline
          autobuffer
          :muted="isMuted"
          preload="auto"
          :id="'video-' + index"
        >
          <source :src="slide.src" type="application/x-mpegURL" />
        </video>
        <!-- Play/pause button -->
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
// Imports
import { ref, onMounted, onUnmounted } from 'vue';
import { register } from 'swiper/element/bundle';
import Hls from 'hls.js';
register();

// All of our reactive data
const showOverlay = ref(false);
const playImage = ref('play.svg');
const pauseImage = ref('pause.svg');
const volumeOnImage = ref('volume-on.svg');
const volumeOffImage = ref('volume-off.svg');
const swiperEl = ref(null);
const videoRefs = ref([]);
const currentPlayingIndex = ref(null);
const isPlaying = ref(false);
const isMuted = ref(true);
const hasStartedPlaying = ref(false);
const endedHandlers = ref([]);

// Video slides
const slides = [
  {
    src: 'https://player.vimeo.com/external/823050002.m3u8?s=3f3e42fa89c4193ccc3e30707faf593eccbb9696',
  },
  {
    src: 'https://player.vimeo.com/external/759635579.m3u8?s=219ffe87caf9351cae8dad3c45c00f656ef35ad8',
  },
  {
    src: 'https://player.vimeo.com/external/832639348.m3u8?s=3f11e4fe435908857eb79bc44a44ddcd16a5733e',
  },
  {
    src: 'https://player.vimeo.com/external/832639376.m3u8?s=2d8046d016c0c46c925f64d4c1e61bc148dacd46',
  },
];

// Mounted
onMounted(() => {

  // If desktop, unmute by default
  if (window.innerWidth >= 768) {
    isMuted.value = false;
  }

  // Show fouce unmute overloay on mobile
  updateShowOverlay();
  window.addEventListener('resize', updateShowOverlay);

  // Swiper params
  const swiperParams = {
    autoplay: false,
    navigation: true,
    pagination: {
      type: 'progressbar',
    },
    injectStyles: [
      `
        :root {--swiper-theme-color: #fff;}
        swiper-container {height:100%;background-color:#202124;}
        swiper-slide {display:flex;align-items:center;justify-content:center;color:#fff;}
        .swiper-button-next,.swiper-button-prev {z-index:9;}
        swiper-container.intro::part(button-prev) {display:none;}
        swiper-container.intro::part(button-next) {position:absolute;top:50%;right:0;left:0;display:flex;align-items:center;justify-content:center;width:4rem;height:4rem;margin:auto;background-color:#fff;border-radius:99px;transform:translateY(-50%);}
        swiper-container.intro::part(button-next):after {content:url('play.svg');width:2rem;height:2rem;margin-left:0.375rem;line-height:0;}
      `,
    ],
  };

  // Assign params and init swiper instance
  Object.assign(swiperEl.value, swiperParams);
  swiperEl.value.initialize();

  // Attach Hls.js to each video
  slides.forEach((slide, index) => {
    if (Hls.isSupported()) {
      var hls = new Hls();
      hls.loadSource(slide.src);
      hls.attachMedia(videoRefs.value[index]);
      videoRefs.value[index].onended = () => {
        if (swiperEl.value && swiperEl.value.swiper) {
          swiperEl.value.swiper.slideNext();
        }
      };
    }
  });
});

// To avoid memory leaks
onUnmounted(() => {
  window.removeEventListener('resize', updateShowOverlay);
});

// Init swiper
const onInit = (e) => {
  console.log('swiper initialized');
};

// Update the mute/unmute overlay for mobile
const updateShowOverlay = () => {
  showOverlay.value = window.innerWidth < 768 && isMuted.value;
};

// Play each video
const playVideo = (video) => {
  if (video) {
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

// Toggle play/pause buttons
const togglePlay = () => {
  if (swiperEl.value && swiperEl.value.swiper) {
    const currentVideo =
      swiperEl.value.swiper.realIndex > 0
        ? videoRefs.value[swiperEl.value.swiper.realIndex - 1]
        : null;
    if (currentVideo) {
      if (isPlaying.value) {
        currentVideo.pause();
        isPlaying.value = false;
      } else {
        playVideo(currentVideo);
        isPlaying.value = true;
        hasStartedPlaying.value = true;
      }
    }
  }
};

// Toggle mute/unmute buttons
const toggleMute = () => {
  if (swiperEl.value && swiperEl.value.swiper) {
    if (isMuted.value) {
      isMuted.value = false;
    } else {
      isMuted.value = true;
    }
    showOverlay.value = window.innerWidth < 768 && isMuted.value;
  }
};

// Handle slide changing
const onSlideChange = (e) => {
  // Pause and reset the time of previous video if any
  if (currentPlayingIndex.value !== null && currentPlayingIndex.value !== 0) {
    const previousVideo = videoRefs.value[currentPlayingIndex.value - 1];
    if (previousVideo) {
      previousVideo.pause();
      previousVideo.currentTime = 0;
      const previousHandler =
        endedHandlers.value[currentPlayingIndex.value - 1];
      if (previousHandler) {
        previousVideo.removeEventListener('ended', previousHandler);
      }
    }
  }

  // Assign current video to real index value from Swiper
  currentPlayingIndex.value = e.detail[0].realIndex;

  // Check if it's not the intro slide
  if (currentPlayingIndex.value > 0) {
    const currentVideo = videoRefs.value[currentPlayingIndex.value - 1];
    if (currentVideo) {
      playVideo(currentVideo);
      hasStartedPlaying.value = true;
      isPlaying.value = true;

      // Remove previous handler and add new one
      const handler = function () {
        swiperEl.value.swiper.slideNext();
      };
      endedHandlers.value[currentPlayingIndex.value - 1] = handler;
      currentVideo.addEventListener('ended', handler);
    }
  }
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
  height: 100dvh;
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
.intro-slide div {
  width: 600px;
  height: 338.02px;
  background-color: #76cdbe;
}
</style>
