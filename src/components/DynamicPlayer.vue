<template>
  <div class="dynamic-player max-w-2xl mx-auto">
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
          w-full
          z-10
          flex
          items-center
          justify-between
          py-6
          px-3
          md:p-6
          text-white text-sm
          font-semibold
        "
      >
        <!-- Tribute info -->
        <div class="flex items-center text-base">
          <img
            src="https://images.pexels.com/photos/4757976/pexels-photo-4757976.jpeg?auto=compress&cs=tinysrgb&w=800"
            class="w-8 h-8 mr-2 rounded-full object-cover"
          />
          Tribute for Pete
        </div>

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
        <div v-else-if="slide.type === 'video'" class="w-full">
          <video
            :ref="
              (el) => {
                blurredVideoRefs[index] = el;
              }
            "
            class="
              blur-bg-video
              absolute
              top-0
              left-0
              h-full
              object-cover
              blur-2xl
            "
            autoplay
            muted
            loop
          >
            <source :src="slide.mp4" type="video/mp4" />
          </video>
          <video
            :ref="
              (el) => {
                videoRefs[index] = el;
              }
            "
            playsinline
            autobuffer
            :muted="isMuted"
            preload="auto"
            :id="'video-' + index"
            class="relative z-10 w-full h-full"
            @loadeddata="console.log(`Video at index ${index} loaded.`)"
          >
            <source :src="slide.hls" type="application/x-mpegURL" />
          </video>
        </div>
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
            z-20
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
    hls: 'https://player.vimeo.com/external/834147408.m3u8?s=d5c93d439ae555eb5f34886d4193a946040db4bd',
    mp4: 'https://player.vimeo.com/progressive_redirect/playback/834147408/rendition/720p/file.mp4?loc=external&signature=9cd831b08b33525f2ce17200a927d5a179b91667186bc7f7263eb850396b4fba',
  },
  {
    type: 'video',
    hls: 'https://player.vimeo.com/external/834176404.m3u8?s=625cd0be4dea8efe8130fe2635ca697b7520aad1',
    mp4: 'https://player.vimeo.com/progressive_redirect/playback/834176404/rendition/720p/file.mp4?loc=external&signature=6b55173801b71c98213f38ca47d33a5667b4f6f93931b718240827a40ca38f37',
  },
  {
    type: 'video',
    hls: 'https://player.vimeo.com/external/832639412.m3u8?s=af15996d49ebcb49d9444c3547919ad70256da15',
    mp4: 'https://player.vimeo.com/progressive_redirect/playback/832639412/rendition/720p/file.mp4?loc=external&signature=a9438d9f7b9652f034fd4a3d26764f87ac293fbaf06d268c86d23ad78155c130',
  },
  {
    type: 'html',
    content:
      '<div class="w-full h-full flex flex-col items-center justify-center p-16" style="background-color: #fec7ce;"><img src="title.svg"></div>',
    duration: 5000,
  },
  {
    type: 'video',
    hls: 'https://player.vimeo.com/external/832639348.m3u8?s=3f11e4fe435908857eb79bc44a44ddcd16a5733e',
    mp4: 'https://player.vimeo.com/progressive_redirect/playback/832639348/rendition/720p/file.mp4?loc=external&signature=3d826abcf784440c8b3970c3d2f5788e5fe7b2b25d58011bda742a827836320b',
  },
  {
    type: 'video',
    hls: 'https://player.vimeo.com/external/832639376.m3u8?s=2d8046d016c0c46c925f64d4c1e61bc148dacd46',
    mp4: 'https://player.vimeo.com/progressive_redirect/playback/832639376/rendition/720p/file.mp4?loc=external&signature=85694be4f99c5d6c889535036f6ebb438308c500578166ea0ec35ea94573a03a',
  },
];
// Define videoRefs as an array of refs initially filled with null
const videoRefs = ref(slides.map(() => null));
const blurredVideoRefs = ref(slides.map(() => null));

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
    on: {
      slideNextTransitionStart: function () {
        preloadAdjacentVideos(this.activeIndex);
      },
      slidePrevTransitionStart: function () {
        preloadAdjacentVideos(this.activeIndex);
      },
    },
    injectStyles: [
      `
        :root {--swiper-theme-color: #fff;}
        swiper-container {height:100%;background-color:#35363a;box-shadow:inset 0px 100px 50px -50px rgba(0,0,0,.25);}
        swiper-slide {display:flex;align-items:center;justify-content:center;color:#fff;transition:all 0.5s ease-out;}
        .swiper-button-next,.swiper-button-prev {z-index:9;}
        swiper-container.intro::part(button-prev) {display:none;}
        swiper-container.bumper::part(button-prev) {display:none;}
        swiper-container.bumper::part(button-next) {display:none;}
        swiper-container.bumper .blur-bg-video {display:none;}
        swiper-container.slide-2::part(button-prev) {opacity:.35;cursor: auto;pointer-events:none;}
        swiper-container.intro::part(button-next) {position:absolute;top:50%;right:0;left:0;display:flex;align-items:center;justify-content:center;width:4rem;height:4rem;margin:auto;background-color:#fff;border-radius:99px;transform:translateY(-50%);}
        swiper-container.intro::part(button-next):after {content:url('play.svg');width:2rem;height:2rem;margin-left:0.375rem;line-height:0;}
        swiper-slide.swiper-slide-active {opacity:1;}
        swiper-slide:not(.swiper-slide-active) {opacity:0;}
        .swiper-pagination-progressbar.swiper-pagination-horizontal {z-index:9;}
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
      hls.loadSource(slide.hls);
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
          blurredVideoRefs.value[swiperEl.value.swiper.realIndex - 1].pause();
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
          playVideo(
            blurredVideoRefs.value[swiperEl.value.swiper.realIndex - 1]
          );
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
      Howler.volume(0.75);
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
      // Set background audio to 50% for video slides
      Howler.volume(0.5);
    }
  }
};

// Preload videos in adjacent slides
const preloadAdjacentVideos = (index) => {
  // Preload previous video if any
  const previousVideo = videoRefs.value[index - 1];
  if (previousVideo && previousVideo.readyState < 3) {
    previousVideo.load();
  }

  // Preload next video if any
  const nextVideo = videoRefs.value[index + 1];
  if (nextVideo && nextVideo.readyState < 3) {
    nextVideo.load();
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
  background-color: #f2615b;
}
.intro-slide + swiper-slide .hide-unless-hovered {
  display: none;
}
.intro,
.bumper {
  background-color: #f2615b;
}
</style>
