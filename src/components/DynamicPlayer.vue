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
        <!--<div v-show="showOverlay" class="overlay">
          <div class="tooltip">Please unmute before playing.</div>
        </div>-->

        <!-- Mute/Unmute button -->
        <!--<button
          @click="toggleMute"
          class="flex items-center justify-center z-10"
        >
          <img
            :src="isMuted ? volumeOffImage : volumeOnImage"
            class="w-6 h-6"
            alt="Mute/Unmute Button"
          />
        </button>-->
      </div>

      <!-- Intro slide -->
      <swiper-slide class="intro-slide">
        <div></div>
      </swiper-slide>

      <!-- Swiper slides -->
      <swiper-slide v-for="(slide, index) in slides" :key="index">
        <!--<video
          ref="videoRefs"
          width="600"
          height="400"
          playsinline
          autobuffer
          :muted="isMuted"
          :id="'video-' + index"
        >-->
        <video
          ref="videoRefs"
          width="600"
          height="400"
          playsinline
          autobuffer
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
const hlsInstances = ref([]);
const currentPlayingIndex = ref(null);
const isPlaying = ref(false);
const isMuted = ref(true);
const hasStartedPlaying = ref(false);
const endedHandlers = ref([]);
const slides = [
  {
    src: 'https://player.vimeo.com/external/823050002.m3u8?s=3f3e42fa89c4193ccc3e30707faf593eccbb9696',
  },
  {
    src: 'https://player.vimeo.com/external/832639348.m3u8?s=3f11e4fe435908857eb79bc44a44ddcd16a5733e',
  },
  {
    src: 'https://player.vimeo.com/external/832639376.m3u8?s=2d8046d016c0c46c925f64d4c1e61bc148dacd46',
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

  // Assign all parameters to Swiper element and initialize it
  Object.assign(swiperEl.value, swiperParams);
  swiperEl.value.initialize();

  // Initialize all Hls instances, but do not attach media yet
  slides.forEach((slide, index) => {
    if (Hls.isSupported()) {
      var hls = new Hls();
      hls.loadSource(slide.src);
      hlsInstances.value[index] = hls;
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
    video.muted = true; // Add this line
    video
      .play()
      .then(() => {
        video.muted = isMuted.value; // After autoplay starts, set the muted state according to the user's setting
      })
      .catch((error) => {
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
    const currentVideo = videoRefs.value[swiperEl.value.swiper.realIndex - 1];
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

const toggleMute = () => {
  if (swiperEl.value && swiperEl.value.swiper) {
    //const currentVideo = videoRefs.value[swiperEl.value.swiper.realIndex];
    //if (currentVideo) {
    if (isMuted.value) {
      console.log('asfsafsa');
      //currentVideo.muted = false; // unmute
      isMuted.value = false;
    } else {
      console.log('dddd');
      //currentVideo.muted = true; // mute
      isMuted.value = true;
    }
    showOverlay.value = window.innerWidth < 768 && isMuted.value;
    //}
  }
};

const onSlideChange = (e) => {
  console.log('slide changed');

  // pause the previous video, if it exists and isn't the intro slide
  if (currentPlayingIndex.value !== null && currentPlayingIndex.value !== 0) {
    const previousVideo = videoRefs.value[currentPlayingIndex.value - 1];
    if (previousVideo) {
      previousVideo.pause();
      isPlaying.value = false;

      // Remove the 'ended' event listener for this video
      if (endedHandlers.value[currentPlayingIndex.value - 1]) {
        previousVideo.removeEventListener(
          'ended',
          endedHandlers.value[currentPlayingIndex.value - 1]
        );
      }
    }
  }

  // update the currently playing video index
  currentPlayingIndex.value = e.detail[0].realIndex;

  // If it's not the intro slide
  if (currentPlayingIndex.value !== 0) {
    // play the current video
    const currentVideo = videoRefs.value[currentPlayingIndex.value - 1];
    if (
      currentVideo &&
      hlsInstances.value[currentPlayingIndex.value - 1].media !== currentVideo
    ) {
      hlsInstances.value[currentPlayingIndex.value - 1].attachMedia(
        currentVideo
      ); // attach Hls only if it's not already attached
    }
    playVideo(currentVideo);
    hasStartedPlaying.value = true;
    isPlaying.value = true;

    // Add event listener for 'ended' event
    const handler = function () {
      swiperEl.value.swiper.slideNext();
      currentVideo.currentTime = 0; // Add this line
    };
    endedHandlers.value[currentPlayingIndex.value - 1] = handler; // Store this handler
    currentVideo.addEventListener('ended', handler);
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
