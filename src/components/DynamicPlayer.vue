<template>
  <div class="dynamic-player max-w-3xl mx-auto">
    <swiper-container
      ref="swiperEl"
      init="false"
      @init="onInit"
      @slidechange="onSlideChange"
      :class="[
        { intro: !hasStartedPlaying, bumper: tributeBumperPlaying },
        'slide-' + currentPlayingIndex,
      ]"
    >
      <!-- Video header with Tribute info and mute button -->
      <div
        slot="container-start"
        class="
          absolute
          w-[calc(100%-1.5rem)]
          md:w-full
          z-10
          flex
          items-center
          justify-between
          py-3
          md:p-6
          text-white text-sm
          font-semibold
        "
      >
        <!-- Tribute info -->
        <div>To: Pete</div>

        <!-- Overaly for mobile interaction -->
        <div v-show="showOverlay" class="overlay">
          <div class="tooltip">Please unmute before playing.</div>
        </div>

        <!-- Mute/Unmute button -->
        <button
          @click="toggleMute"
          class="flex items-center justify-center z-10 mr-1 md:mr-0"
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
        <div class="rounded-lg"></div>
      </swiper-slide>

      <!-- Swiper slides -->
      <swiper-slide v-for="(slide, index) in slides" :key="index">
        <!-- HTML slide type -->
        <div
          class="html-slide w-full h-full flex items-center justify-center"
          v-if="slide.type === 'html'"
          v-html="slide.content"
        ></div>
        <!-- Video slide type -->
        <video
          v-else-if="slide.type === 'video'"
          :ref="
            (el) => {
              videoRefs[index] = el;
            }
          "
          width="600"
          height="400"
          playsinline
          autobuffer
          :muted="isMuted"
          preload="auto"
          :id="'video-' + index"
          class="rounded-lg"
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
import { Howl, Howler } from 'howler';
register();

// All of our reactive data
const showOverlay = ref(false);
const playImage = ref('play.svg');
const pauseImage = ref('pause.svg');
const volumeOnImage = ref('volume-on.svg');
const volumeOffImage = ref('volume-off.svg');
const swiperEl = ref(null);
const currentPlayingIndex = ref(null);
const isPlaying = ref(false);
const isMuted = ref(true);
const sound = ref(null);
const hasStartedPlaying = ref(false);
const tributeBumperPlaying = ref(false);
const htmlSlideTimeout = ref(null);
const isHTMLPaused = ref(false);
const endedHandlers = ref([]);

// Video slides
const slides = [
  {
    // Tribute bumper
    type: 'video',
    src: 'https://player.vimeo.com/external/823050002.m3u8?s=3f3e42fa89c4193ccc3e30707faf593eccbb9696',
  },
  {
    type: 'video',
    src: 'https://player.vimeo.com/external/832639412.m3u8?s=af15996d49ebcb49d9444c3547919ad70256da15',
  },
  {
    type: 'html',
    content:
      '<div class="w-[600px] h-[400px] flex flex-col items-center justify-center p-8 rounded-lg" style="background-color: #ffce50;"><div class="mb-6 text-black text-4xl font-semibold">Title slide</div><svg xmlns="http://www.w3.org/2000/svg" class="w-24" fill="none" viewBox="0 0 48 48" stroke-width="2"><path fill="#94e986" d="M3.13477 20.8571L10.4451 2H30.7769L23.4665 20.8571H34.3559L8.3129 46L11.3589 20.8571H3.13477Z"></path><path stroke="#000000" stroke-linejoin="round" d="M3.13477 20.8571L10.4451 2H30.7769L23.4665 20.8571H34.3559L8.3129 46L11.3589 20.8571H3.13477Z"></path><path fill="#ffce51" d="M34.3574 20.8564L8.31445 45.9993H18.823L44.866 20.8564H34.3574Z"></path><path fill="#ffce51" d="M30.7751 2L23.4648 20.8571H33.9734L41.2837 2H30.7751Z"></path><path stroke="#000000" stroke-linejoin="round" d="M34.3574 20.8564L8.31445 45.9993H18.823L44.866 20.8564H34.3574Z"></path><path stroke="#000000" stroke-linejoin="round" d="M30.7751 2L23.4648 20.8571H33.9734L41.2837 2H30.7751Z"></path></svg></div>',
    duration: 5000,
  },
  {
    type: 'video',
    src: 'https://player.vimeo.com/external/832639348.m3u8?s=3f11e4fe435908857eb79bc44a44ddcd16a5733e',
  },
  {
    type: 'video',
    src: 'https://player.vimeo.com/external/832639376.m3u8?s=2d8046d016c0c46c925f64d4c1e61bc148dacd46',
  },
];
// Define videoRefs as an array of refs initially filled with null
const videoRefs = ref(slides.map(() => null));

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
    //effect: 'fade',
    pagination: {
      type: 'progressbar',
    },
    injectStyles: [
      `
        :root {--swiper-theme-color: #fff;}
        swiper-container {height:100%;background-color:#35363a;}
        @media (max-width: 767px) {swiper-container {padding:0.75rem;}}
        swiper-slide {display:flex;align-items:center;justify-content:center;color:#fff;transition:all 0.5s ease-out;}
        .swiper-button-next,.swiper-button-prev {z-index:9;}
        swiper-container.intro::part(button-prev) {display:none;}
        swiper-container.bumper::part(button-prev) {display:none;}
        swiper-container.bumper::part(button-next) {display:none;}
        swiper-container.slide-2::part(button-prev) {opacity:.35;cursor: auto;pointer-events:none;}
        swiper-container.intro::part(button-next) {position:absolute;top:50%;right:0;left:0;display:flex;align-items:center;justify-content:center;width:4rem;height:4rem;margin:auto;background-color:#fff;border-radius:99px;transform:translateY(-50%);}
        swiper-container.intro::part(button-next):after {content:url('play.svg');width:2rem;height:2rem;margin-left:0.375rem;line-height:0;}
        swiper-slide.swiper-slide-active {opacity:1;}
        swiper-slide:not(.swiper-slide-active) {opacity:0;}
      `,
    ],
  };

  // Assign params and init swiper instance
  Object.assign(swiperEl.value, swiperParams);
  swiperEl.value.initialize();

  // Attach Hls.js to each video
  slides.forEach((slide, index) => {
    if (slide.type === 'video' && Hls.isSupported() && videoRefs.value[index]) {
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
        video.volume = 0;
        video.play();
      }
    });
  }
};

// Toggle play/pause buttons
const togglePlay = () => {
  if (swiperEl.value && swiperEl.value.swiper) {
    const currentSlide =
      swiperEl.value.swiper.realIndex > 0
        ? slides[swiperEl.value.swiper.realIndex - 1]
        : null;
    if (currentSlide) {
      if (isPlaying.value) {
        if (currentSlide.type === 'video') {
          videoRefs.value[swiperEl.value.swiper.realIndex - 1].pause();
        } else if (currentSlide.type === 'html') {
          clearTimeout(htmlSlideTimeout.value);
          isHTMLPaused.value = true;
        }
        isPlaying.value = false;
        // Pause the background audio
        if (sound.value) {
          sound.value.pause();
        }
      } else {
        if (currentSlide.type === 'video') {
          playVideo(videoRefs.value[swiperEl.value.swiper.realIndex - 1]);
        } else if (currentSlide.type === 'html' && isHTMLPaused.value) {
          htmlSlideTimeout.value = setTimeout(nextSlide, currentSlide.duration);
          isHTMLPaused.value = false;
        }
        isPlaying.value = true;
        hasStartedPlaying.value = true;
        // Play the background audio
        if (sound.value) {
          sound.value.play();
        }
      }
    }
  }
};

// Toggle mute/unmute buttons
const toggleMute = () => {
  isMuted.value = !isMuted.value;
  showOverlay.value = window.innerWidth < 768 && isMuted.value;

  // Update volume of videos
  videoRefs.value.forEach((video) => {
    if (video) {
      video.muted = isMuted.value;
    }
  });

  // Update volume of background audio
  if (sound.value) {
    Howler.mute(isMuted.value);
  }
};

// Check swiper to see if it's playing before auto-advancing
const nextSlide = () => {
  if (isPlaying.value) {
    swiperEl.value.swiper.slideNext();
  }
};

// Handle slide changing
const onSlideChange = (e) => {
  // Clear any existing HTML slide timeout
  if (htmlSlideTimeout.value) {
    clearTimeout(htmlSlideTimeout.value);
    htmlSlideTimeout.value = null;
  }

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
  console.log('playing slide ' + currentPlayingIndex.value);

  if (currentPlayingIndex.value === 1) {
    tributeBumperPlaying.value = true;
  }

  // Play background audio on the first real slide
  if (currentPlayingIndex.value === 2) {
    playSound();
    tributeBumperPlaying.value = false;
  }

  // Check if it's not the intro slide
  if (currentPlayingIndex.value > 0) {
    const slide = slides[currentPlayingIndex.value - 1];
    if (slide.type === 'html') {
      // If it's an HTML slide, wait for the duration and then advance to next slide
      htmlSlideTimeout.value = setTimeout(nextSlide, slide.duration);
      // Set background audio to 100% for html slides
      Howler.volume(1.0);
    } else if (slide.type === 'video') {
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
      // Set background audio to 25% for video slides
      Howler.volume(0.25);
    }
  }
};

// Background audio
const playSound = () => {
  if (!sound.value) {
    sound.value = new Howl({
      src: ['audio.mp3'],
      autoplay: false,
      loop: false,
      volume: 1.0,
    });
  }
  sound.value.play();
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
.intro-slide + swiper-slide .hide-unless-hovered {
  display: none;
}
</style>
