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
    src: 'https://media.videoask.com/transcoded/b06c8fd3-66dd-4424-8503-4726ccf67431/video.mp4?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJtZWRpYV9pZCI6ImIwNmM4ZmQzLTY2ZGQtNDQyNC04NTAzLTQ3MjZjY2Y2NzQzMSIsImV4cCI6MTY0ODU4ODYyOX0.YpzAucjHVNb9E52ix0FEyjfEMKXapxWTlwewb-kfu2S-JYLl8lpvgMZtCZF2vhE1RueOEja8_z4pxyGBXbl8cGTWOXKoeAXEvHWbZJryfUFk4gB7KKs_0-qN0CtZjF3yjHdli-vYJyzKoYmCbrpO7I8v8UUzQaWxe44np9AtxiNZ7Kzi-KHnYGCd5ufI_HBaDwUyJA4PCrsaEc5qIPzzVtSWnJRy99xGHacHGK6uqApJUvT5gj2IqcWAEZEVgABu4lYyUzepKmXgcJ0JXfSpDzOFe5ooWxjEecIK_geZ9EH7iBDA3vqQa6C-R4QgSAW494746noWRWGQ43AP8OBo8JJBDpPY9St5YmUgZL-bT7hizQrTn_jQ0EICMxRhURvRKlA8DrMnn_WcJ_1g4ydNckEQbPIM4qb_Zo12vnyEwQZUzUMqE8U33Vx-L1iz4iHRD61Ms-uHg8ud6rSyzgIh6NqEytamIXsTqeJ9-rPPFbkzLf8NPwVnP7a4TeYxm1iIqgp3S6bHWgYXAhrN6Hpjs2jVKkmEA3OI2C4SOvUbSxYC7RgRCrYgp8qvM3y_B0gkGfiFVvZWoHtOrZ4AuQY9UOLL5e-cu2XX1Rr9urRoWGQG5DPqQVW4KgKON_DAw_lCb6Ebw4GdrhcHjvUvJ1Zi6XPnWjKfU9_pptjpcCchA8E',
  },
  {
    src: 'https://media.videoask.com/transcoded/a41c4ae3-acee-49a5-a521-8593965a5861/video.mp4?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJtZWRpYV9pZCI6ImE0MWM0YWUzLWFjZWUtNDlhNS1hNTIxLTg1OTM5NjVhNTg2MSIsImV4cCI6MTY0ODU4Nzg4NH0.uJ4j9EAkAfPiLHSQ0EUPmxLkkgilLtsw0zxWDYdLECNT6IhPnweTAGFmFXvqPMZLkvdAlEPcrj4SZfIg5hdR3IJMTFaerHl_lo4IAQeFLadElt4Xy77k_PezhnN4WSDfQb9EZ1zCGZ1GlIeoT2a1E0XReYFJ2E3BeZcW5ALxmpjEdporjc3AxaaPOoRxFwb4EGiiCoqOSC6PDzAnqSPQfXAUV5uBb4YIlVKgzwLlPeKff8co2we-m5KhlRue1eO8IfCJMiq5ROJhjVWSXJpiGA_0XbUXPUzEwdM3v-oOlg17FY79xCV0_MFaSjaCdyy3FBanH67rnBUg7ytUV0MrjuUaZxa6TIzDF8RplQXKsbNI0Z5sWnWC0WJB1F8Zb_Ou6SOH_PAkuLcTmcfSYRXmkIl3uWZ5RcSjhPPLJKgCVOZg1lZwlwn2pdZnV-KK5Y1PF5bn-mFMzNXpKo23U0Q2MyFzOzR-LB7-EhOebXZByXowPcp9LaM1tetlYrFGB3eSrXg0--uemcYWRqDr5n7DVmG5P0Rb5O9o1A56HK6KVzS8qFHNyG5ifRhZAi4EOl-xnPok9C6a3ap5tVTX2ChMDu6hYiD3gIzFDp-GgyyPLmBFEpOqwAQWWlfTRK_q0fpQyn029K7VSHCOf6silogX4MJ_4G_MiZSoJywqOKmIbYA',
  },
  {
    src: 'https://media.videoask.com/transcoded/62aeb258-f82c-4117-9396-0ecfdf67fe4a/video.mp4?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9.eyJtZWRpYV9pZCI6IjYyYWViMjU4LWY4MmMtNDExNy05Mzk2LTBlY2ZkZjY3ZmU0YSIsImV4cCI6MTY0ODU4Nzg4M30.BIIAl7BOWiCFAuFAFlIaFm8tVPr7FJjI5-LWYkUjvSPrD2Mk_v9ZJTcHh4h9Rkp0uSnLVD-rul1aUsiUB3-qqtLrJDa98AvuthyQBKWJHHFaWQrhRPkp-cAgZoZd_Lp8FtAzzRuQZhlvPpVzfQ7WlMHbm_RqvUYO8VbpIvO-MfYxR8Dq9UB25XHw2nlhYHM9ToY-HMSPvGSG5fvyrvbREjdVtcwEFrM8tASq2JEUrRLjVsjy1yn6uq9flUCQFdP0KZu5gaxJg4ud6cs8ju8515iDtgrirvhKfAwCS2AsFDLBzzLpeo0HFTXf2Utt3ckYwClxBY5LkkB_MZDUiwAxqJnszCEfsaa3honxAWU_ZMqf1SjkEbH_u9VmFDgeAAjRWssdDfS_YCrcsnUh4xFLQ_UdjNDLEosCnyvHZopFRRZITh_B4sM_mr4Zawj1YPyFFQL6Uz2MdKkIDJ2jQjiFjSLr7a1ntomgJQx1tojVDIqsyWv6BOUKPRRAWrwu3w4wIK_OrBJ1ynzAKCXnG1OQ1E_UU99aerS5qS4oacJtkOQ71X78j4ZMP7Z6wLrTPak9CsEIEqJoEnO54MI9PP9totSA4-VVZABAJKNm1vSIhgeyrIiwxyXjE5dC6UkkOQyo2pqaakqnnP_rdP9YqLOUJ2rWS50coMb9421HH7xS_qk',
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
