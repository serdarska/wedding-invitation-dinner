
<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue'

const vScrollReveal = {
  mounted(el: HTMLElement, binding: { value?: number }) {
    el.classList.add('reveal-hidden')
    if (binding.value) el.style.transitionDelay = `${binding.value}ms`
    // double rAF ensures browser paints the hidden state before observer fires
    requestAnimationFrame(() => requestAnimationFrame(() => {
      const observer = new IntersectionObserver(
        ([entry]) => { if (entry?.isIntersecting) { el.classList.add('reveal-visible'); observer.disconnect() } },
        { threshold: 0.12 }
      )
      observer.observe(el)
    }))
  }
}

type Stage = 'idle' | 'opening' | 'revealed'
const stage = ref<Stage>('idle')
const videoRef = ref<HTMLVideoElement | null>(null)

const envelopeVideoSrc = new URL('./images/envelope-video.mp4', import.meta.url).href
const envelopePosterSrc = new URL('./images/envelope-picture.png', import.meta.url).href
const hotelBg = new URL('./images/hotel.jpg', import.meta.url).href
const songSrc = new URL('./images/A Whole New World - Zayn Zhavia Ward lirik video.mp3', import.meta.url).href

const audioRef = ref<HTMLAudioElement | null>(null)
const muted = ref(false)

function toggleMute() {
  muted.value = !muted.value
  if (audioRef.value) audioRef.value.muted = muted.value
}

function open() {
  if (stage.value !== 'idle') return
  stage.value = 'opening'
  const v = videoRef.value
  if (v) {
    v.play()
    v.onended = () => {
      stage.value = 'revealed'
      audioRef.value?.play()
    }
  }
}

const weddingDate = new Date('2026-09-13T00:00:00')
const timeLeft = ref({ days: 0, hours: 0, minutes: 0, seconds: 0 })
function calcTimeLeft() {
  const diff = weddingDate.getTime() - Date.now()
  if (diff <= 0) { timeLeft.value = { days: 0, hours: 0, minutes: 0, seconds: 0 }; return }
  timeLeft.value = {
    days: Math.floor(diff / (1000 * 60 * 60 * 24)),
    hours: Math.floor((diff / (1000 * 60 * 60)) % 24),
    minutes: Math.floor((diff / 1000 / 60) % 60),
    seconds: Math.floor((diff / 1000) % 60),
  }
}
let timer: ReturnType<typeof setInterval>
onMounted(() => { calcTimeLeft(); timer = setInterval(calcTimeLeft, 1000) })
onUnmounted(() => clearInterval(timer))

type Guest = { name: string; menu: '' | 'regular' | 'vegetarian' }
const rsvp = ref({
  name: '',
  attending: '',
  menu: '' as '' | 'regular' | 'vegetarian',
  guests: [] as Guest[],
})
const submitted = ref(false)
function addGuest() {
  if (rsvp.value.guests.length < 4) rsvp.value.guests.push({ name: '', menu: '' })
}
function removeGuest(i: number) {
  rsvp.value.guests.splice(i, 1)
}
async function submitRsvp() {
  const base = rsvp.value.name && rsvp.value.attending
  const menuOk = rsvp.value.attending === 'no' || rsvp.value.menu
  const guestsOk = rsvp.value.guests.every(g => g.name && g.menu)
  if (!base || !menuOk || !guestsOk) return

  const formUrl = 'https://docs.google.com/forms/d/e/1FAIpQLSdk5BLZPEEkh9FHdvSjBBR7dVUMMzsdY2eOySULRNi0POPoBw/formResponse'
  const body = new FormData()
  body.append('entry.128665478',  rsvp.value.name)
  body.append('entry.1660373188', rsvp.value.attending)
  body.append('entry.1652317488', rsvp.value.menu)
  body.append('entry.1329776834', rsvp.value.guests[0]?.name ?? '')
  body.append('entry.1634828065', rsvp.value.guests[0]?.menu ?? '')
  body.append('entry.1802687977', rsvp.value.guests[1]?.name ?? '')
  body.append('entry.34598583',   rsvp.value.guests[1]?.menu ?? '')
  body.append('entry.1895251606', rsvp.value.guests[2]?.name ?? '')
  body.append('entry.1000297850', rsvp.value.guests[2]?.menu ?? '')
  body.append('entry.1406806521', rsvp.value.guests[3]?.name ?? '')
  body.append('entry.1858170175', rsvp.value.guests[3]?.menu ?? '')

  await fetch(formUrl, { method: 'POST', body, mode: 'no-cors' })
  submitted.value = true
}
</script>

<template>
  <!-- Audio -->
  <audio ref="audioRef" :src="songSrc" loop preload="auto"></audio>

  <!-- Floating mute button (only when revealed) -->
  <Transition name="rsvp-fade">
    <button
      v-if="stage === 'revealed'"
      @click="toggleMute"
      class="fixed bottom-6 right-6 z-50 bg-black/60 backdrop-blur-sm rounded-full w-10 h-10 flex items-center justify-center border border-white/20"
    >
      <svg v-if="!muted" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"/><path d="M19.07 4.93a10 10 0 0 1 0 14.14"/><path d="M15.54 8.46a5 5 0 0 1 0 7.07"/>
      </svg>
      <svg v-else width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
        <polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"/><line x1="23" y1="9" x2="17" y2="15"/><line x1="17" y1="9" x2="23" y2="15"/>
      </svg>
    </button>
  </Transition>

  <!-- ===== ENVELOPE OVERLAY ===== -->
  <Transition name="overlay">
    <div
      v-if="stage !== 'revealed'"
      class="env-screen"
      :class="{ 'cursor-pointer': stage === 'idle' }"
      @click="open"
    >
      <video
        ref="videoRef"
        class="env-video"
        :src="envelopeVideoSrc"
        :poster="envelopePosterSrc"
        playsinline
        muted
        preload="auto"
      ></video>
      <p v-if="stage === 'idle'" class="env-hint">ДОПРИ ЗА ДА ОТВОРИШ!</p>
    </div>
  </Transition>

  <!-- ===== FULL INVITATION ===== -->
  <Transition>
    <div v-if="stage === 'revealed'">
      <div class="relative w-full overflow-hidden" style="height: 100svh;">
        <img class="absolute inset-0 w-full h-full object-cover" src="../src/images/the_ring.jpeg" alt="the ring">
        <!-- gradient overlay: dark at top & bottom, transparent in middle -->
        <div class="absolute inset-0" style="background: linear-gradient(to bottom, rgba(0,0,0,0.55) 0%, rgba(0,0,0,0.1) 40%, rgba(0,0,0,0.1) 60%, rgba(0,0,0,0.65) 100%);"></div>

        <!-- Top label -->
        <div class="absolute top-12 left-0 right-0 flex flex-col items-center gap-2">
          <div class="flex items-center gap-3">
            <div class="w-8 h-px bg-[#c9a96e]/70"></div>
            <p class="font-['Montserrat'] text-[12px] tracking-[0.4em] uppercase text-[#c9a96e]">таму каде започна се</p>
            <div class="w-8 h-px bg-[#c9a96e]/70"></div>
          </div>
        </div>

        <!-- Bottom scroll hint -->
        <div class="absolute bottom-8 left-1/2 -translate-x-1/2 flex flex-col items-center gap-2 animate-bounce">
          <p class="font-['Montserrat'] text-[14px] tracking-[0.35em] uppercase text-white/70 whitespace-nowrap">Скролни надолу</p>
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="white" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" style="opacity:0.7">
            <path d="M12 5v14M5 12l7 7 7-7"/>
          </svg>
        </div>
      </div>

      <!-- Names + date hero -->
      <div class="bg-[#181818] flex flex-col items-center text-center px-8 pt-16 pb-16">

        <!-- Small divider -->
        <div v-scroll-reveal="0" class="flex items-center gap-3 mb-10">
          <div class="w-8 h-px bg-[#c9a96e]/40"></div>
          <svg width="8" height="8" viewBox="0 0 24 24" fill="#c9a96e"><circle cx="12" cy="12" r="6"/></svg>
          <div class="w-8 h-px bg-[#c9a96e]/40"></div>
        </div>

        <h1 v-scroll-reveal="100" class="font-['Cormorant_Garamond'] font-light italic text-7xl text-white leading-none">Јована</h1>
        <p v-scroll-reveal="150" class="font-['Cormorant_Garamond'] text-3xl text-[#c9a96e] my-3">&</p>
        <h1 v-scroll-reveal="200" class="font-['Cormorant_Garamond'] font-light italic text-7xl text-white leading-none">Борис</h1>

        <p v-scroll-reveal="300" class="font-['Cormorant_Garamond'] font-light italic text-lg text-[#9b8c7a] mt-8 mb-10 leading-relaxed">
          Со задоволство ве каниме<br/>на нашата свадба
        </p>

        <!-- Date card -->
        <div v-scroll-reveal="400" class="bg-white/15 backdrop-blur-sm rounded-2xl px-10 py-8 text-center w-full max-w-xs">
          <p class="font-['Cormorant_Garamond'] text-2xl text-white mb-1">Недела, 13 Септември 2026</p>
        </div>
      </div>




      <!-- Location -->
      <div class="relative py-14 px-6" :style="{ backgroundImage: `url(${hotelBg})`, backgroundSize: 'cover', backgroundPosition: 'center' }">
        <div class="absolute inset-0 bg-black/50"></div>
        <div class="relative z-10">
          <p v-scroll-reveal="0" class="font-['Montserrat'] text-[9px] tracking-[0.35em] uppercase text-[#c9a96e] text-center mb-3 font-bold">Придружете ни се во</p>
          <h2 v-scroll-reveal="100" class="font-['Cormorant_Garamond'] font-light text-4xl text-white text-center mb-8">Хотел Брана Градче</h2>

          <!-- Info cards -->
          <div class="grid grid-rows-2 gap-6 mb-6">
            <div v-scroll-reveal="200" class="bg-white/15 backdrop-blur-sm rounded-xl p-4 flex items-start gap-3">
              <span class="text-[#c9a96e] mt-0.5">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
                  <path d="M12 2C8.13 2 5 5.13 5 9c0 5.25 7 13 7 13s7-7.75 7-13c0-3.87-3.13-7-7-7z"/><circle cx="12" cy="9" r="2.5"/>
                </svg>
              </span>
              <div>
                <p class="font-['Montserrat'] text-[9px] tracking-[0.2em] uppercase text-[#c9a96e] ">Локација</p>
                <p class="font-['Cormorant_Garamond'] text-base text-white">Хотел Брана Градче</p>
              </div>
            </div>
            <div v-scroll-reveal="300" class="bg-white/15 backdrop-blur-sm rounded-xl p-4 flex items-start gap-3">
              <span class="text-[#c9a96e] mt-0.5">
                <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
                  <circle cx="12" cy="12" r="10"/><path d="M12 6v6l4 2"/>
                </svg>
              </span>
              <div>
                <p class="font-['Montserrat'] text-[9px] tracking-[0.2em] uppercase text-[#c9a96e] ">Време на пристигнување</p>
                <p class="font-['Cormorant_Garamond'] text-base text-white">Прием од 19:00 </p>
              </div>
            </div>
          </div>

          <!-- Map embed -->
<!--          <div v-scroll-reveal="400" class="rounded-2xl overflow-hidden mb-6">-->
<!--            <iframe-->
<!--              src="https://maps.google.com/maps?q=Хотел+Брана+Градче&output=embed&hl=mk"-->
<!--              width="100%"-->
<!--              height="260"-->
<!--              style="border:0;"-->
<!--              loading="lazy"-->
<!--              referrerpolicy="no-referrer-when-downgrade"-->
<!--            ></iframe>-->
<!--          </div>-->

          <!-- Directions button -->
          <a
            v-scroll-reveal="500"
            href="https://maps.app.goo.gl/XhAJJ5LPd8aZkshM8"
            target="_blank"
            rel="noopener"
            class="flex items-center justify-center gap-2 w-full bg-white text-[#2e2118] rounded-xl py-4 font-['Montserrat'] text-[10px] tracking-[0.3em] uppercase"
          >
            <svg width="14" height="14" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round">
              <path d="M12 2L4.5 20.29l.71.71L12 18l6.79 3 .71-.71z"/>
            </svg>
            Преземи насоки
          </a>
        </div>
      </div>


      <!-- Schedule -->
      <div class="bg-[#181818] py-14 px-6">
        <p v-scroll-reveal="0" class="font-['Montserrat'] text-[9px] tracking-[0.35em] uppercase text-[#9b8c7a] text-center mb-3">Редослед</p>
        <h2 v-scroll-reveal="100" class="font-['Cormorant_Garamond'] font-light text-4xl text-white text-center mb-8">Денот на свадбата</h2>

        <div class="flex flex-col gap-3">
          <div v-for="(event, i) in [
            { icon: '️🤵🏽', label: 'Сватови',            time: '10:00' },
            { icon: '👰🏽‍♀️', label: 'По невестата',       time: '12:00' },
            { icon: '⛪', label: 'Црква',              time: '14:30' },
            { icon: '💍', label: 'Регистрација',       time: '18:00' },
            { icon: '🥂', label: 'Прием во ресторан',  time: '19:00' },
            { icon: '🎵', label: 'Првиот танц',        time: '20:00' },
            { icon: '🍽️', label: 'Вечера',             time: '22:00' },
            { icon: '🎂', label: 'Торта',              time: '23:00' },
            { icon: '✨', label: 'Крај на забавата',   time: '01:00' },
          ]" :key="event.label"
            v-scroll-reveal="200 + i * 80"
            class="bg-white/15 backdrop-blur-sm  rounded-xl  px-4 py-4 flex items-center gap-4">
            <span class="text-lg w-7 text-center">{{ event.icon }}</span>
            <span class="font-['Cormorant_Garamond'] text-lg text-white flex-1">{{ event.label }}</span>
            <span class="font-['Montserrat'] text-[11px] text-[#9b8c7a] tracking-wide">{{ event.time }}</span>
          </div>
        </div>
      </div>


      <!-- Countdown -->
      <div class="relative py-26 px-10 overflow-hidden">
        <img src="../src/images/us.png" alt="" class="absolute inset-0 w-full h-full object-cover scale-105" style="filter: blur(2px);">
        <div class="absolute inset-0 bg-black/50"></div>
        <div class="relative z-10">
          <p v-scroll-reveal="0" class="font-['Montserrat'] text-[9px] tracking-[0.35em] uppercase text-[#c9a96e] text-center mb-4">Одбројување до</p>
          <h2 v-scroll-reveal="100" class="font-['Cormorant_Garamond'] font-light text-2xl text-white text-center mb-10">НАШЕТО ЗАСЕКОГАШ</h2>
          <div class="grid grid-cols-2 gap-3">
            <div v-for="(unit, i) in [
            { label: 'Денови',  value: timeLeft.days },
            { label: 'Часови',  value: timeLeft.hours },
            { label: 'Минути',  value: timeLeft.minutes },
            { label: 'Секунди', value: timeLeft.seconds },
          ]" :key="unit.label"
                 v-scroll-reveal="200 + i * 100"
                 class="bg-white/15 backdrop-blur-sm rounded-xl flex flex-col items-center justify-center py-5 gap-3">
            <span class="font-['Cormorant_Garamond'] font-light italic text-5xl text-white leading-none">
              {{ String(unit.value).padStart(2, '0') }}
            </span>
              <span class="font-['Montserrat'] text-[8px] tracking-[0.2em] uppercase text-[#c9a96e]">{{ unit.label }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Important info -->
      <div class="bg-[#181818] py-14 px-6">
        <p v-scroll-reveal="0" class="font-['Montserrat'] text-[9px] tracking-[0.35em] uppercase text-[#9b8c7a] text-center mb-3">Внимание</p>
        <h2 v-scroll-reveal="100" class="font-['Cormorant_Garamond'] font-light text-4xl text-white text-center mb-8">Важни информации</h2>

        <div class="flex flex-col gap-3">
          <div v-for="(info, i) in [

            'Професионални фотографи ќе го забележат овој ден. Ве молиме уживајте во моментот!',
            'Обезбеден е паркинг во рамки на локацијата за сите гости.',
          ]" :key="info"
            v-scroll-reveal="200 + i * 100"
            class="bg-white/15 backdrop-blur-sm rounded-xl px-4 py-4 flex items-start gap-3">
            <span class="text-[#c9a96e] mt-0.5 shrink-0">
              <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
                <circle cx="12" cy="12" r="10"/><line x1="12" y1="8" x2="12" y2="12"/><line x1="12" y1="16" x2="12.01" y2="16"/>
              </svg>
            </span>
            <p class="font-['Cormorant_Garamond'] text-lg text-white leading-snug">{{ info }}</p>
          </div>
        </div>
      </div>

      <!-- RSVP -->
      <div class="bg-[#f5f0ea] py-14 px-6">
        <p v-scroll-reveal="0" class="font-['Montserrat'] text-[9px] tracking-[0.35em] uppercase text-[#c9a96e] text-center mb-3">Потврди присуство</p>
        <h2 v-scroll-reveal="100" class="font-['Cormorant_Garamond'] font-light text-4xl text-[#2e2118] text-center mb-2">RSVP</h2>
        <p v-scroll-reveal="150" class="font-['Cormorant_Garamond'] italic text-[#9b8c7a] text-center mb-8">Ве молиме потврдете до 25ти Август 2026</p>

        <Transition name="rsvp-fade" mode="out-in">
          <div v-if="!submitted" key="form" class="flex flex-col gap-4">

            <!-- Name -->
            <div v-scroll-reveal="200">
              <label class="font-['Montserrat'] text-[9px] tracking-[0.3em] uppercase text-[#9b8c7a] block mb-2">Име и презиме</label>
              <input
                v-model="rsvp.name"
                type="text"
                placeholder="Вашето име..."
                class="w-full bg-white rounded-xl px-4 py-3 font-['Cormorant_Garamond'] text-lg text-[#2e2118] placeholder-[#9b8c7a] outline-none border border-transparent focus:border-[#c9a96e] transition-colors duration-200"
              />
            </div>

            <!-- Attending -->
            <div v-scroll-reveal="300">
              <label class="font-['Montserrat'] text-[9px] tracking-[0.3em] uppercase text-[#9b8c7a] block mb-2">Дали ќе присуствувате?</label>
              <div class="flex gap-3">
                <button
                  @click="rsvp.attending = 'yes'"
                  :class="rsvp.attending === 'yes' ? 'bg-[#2e2118] text-white' : 'bg-white text-[#2e2118]'"
                  class="flex-1 rounded-xl py-3 font-['Montserrat'] text-[10px] tracking-[0.2em] uppercase transition-colors duration-200"
                >Да, ќе дојдам</button>
                <button
                  @click="rsvp.attending = 'no'"
                  :class="rsvp.attending === 'no' ? 'bg-[#2e2118] text-white' : 'bg-white text-[#2e2118]'"
                  class="flex-1 rounded-xl py-3 font-['Montserrat'] text-[10px] tracking-[0.2em] uppercase transition-colors duration-200"
                >За жал, не можам</button>
              </div>
            </div>

            <!-- Menu + plus guest (only when attending) -->
            <template v-if="rsvp.attending === 'yes'">

              <!-- Menu choice -->
              <div v-scroll-reveal="400">
                <label class="font-['Montserrat'] text-[9px] tracking-[0.3em] uppercase text-[#9b8c7a] block mb-2">Избор на мени</label>
                <div class="flex gap-3">
                  <button
                    @click="rsvp.menu = 'regular'"
                    :class="rsvp.menu === 'regular' ? 'bg-[#2e2118] text-white' : 'bg-white text-[#2e2118]'"
                    class="flex-1 rounded-xl py-3 font-['Montserrat'] text-[10px] tracking-[0.2em] uppercase transition-colors duration-200"
                  >Стандардно</button>
                  <button
                    @click="rsvp.menu = 'vegetarian'"
                    :class="rsvp.menu === 'vegetarian' ? 'bg-[#2e2118] text-white' : 'bg-white text-[#2e2118]'"
                    class="flex-1 rounded-xl py-3 font-['Montserrat'] text-[10px] tracking-[0.2em] uppercase transition-colors duration-200"
                  >Вегетаријанско</button>
                </div>
              </div>

              <!-- Added guests -->
              <div v-for="(guest, i) in rsvp.guests" :key="i" class="bg-white rounded-xl px-4 py-4 flex flex-col gap-3">
                <div class="flex items-center justify-between">
                  <label class="font-['Montserrat'] text-[9px] tracking-[0.3em] uppercase text-[#9b8c7a]">Гостин {{ i + 1 }}</label>
                  <button @click="removeGuest(i)" class="font-['Montserrat'] text-[9px] tracking-[0.2em] uppercase text-[#9b8c7a] hover:text-[#2e2118] transition-colors duration-200">Отстрани</button>
                </div>
                <input
                  v-model="guest.name"
                  type="text"
                  placeholder="Ime и презиме..."
                  class="w-full bg-[#f5f0ea] rounded-xl px-4 py-3 font-['Cormorant_Garamond'] text-lg text-[#2e2118] placeholder-[#9b8c7a] outline-none border border-transparent focus:border-[#c9a96e] transition-colors duration-200"
                />
                <div class="flex gap-3">
                  <button
                    @click="guest.menu = 'regular'"
                    :class="guest.menu === 'regular' ? 'bg-[#2e2118] text-white' : 'bg-[#f5f0ea] text-[#2e2118]'"
                    class="flex-1 rounded-xl py-3 font-['Montserrat'] text-[10px] tracking-[0.2em] uppercase transition-colors duration-200"
                  >Стандардно</button>
                  <button
                    @click="guest.menu = 'vegetarian'"
                    :class="guest.menu === 'vegetarian' ? 'bg-[#2e2118] text-white' : 'bg-[#f5f0ea] text-[#2e2118]'"
                    class="flex-1 rounded-xl py-3 font-['Montserrat'] text-[10px] tracking-[0.2em] uppercase transition-colors duration-200"
                  >Вегетаријанско</button>
                </div>
              </div>

              <!-- Add guest button (max 4) -->
              <button
                v-if="rsvp.guests.length < 4"
                v-scroll-reveal="500"
                @click="addGuest"
                class="w-full border border-[#c9a96e] text-[#c9a96e] rounded-xl py-3 font-['Montserrat'] text-[10px] tracking-[0.2em] uppercase transition-colors duration-200"
              >+ Додај гостин</button>

            </template>

            <button
              v-scroll-reveal="0"
              @click="submitRsvp"
              :disabled="!rsvp.name || !rsvp.attending || (rsvp.attending === 'yes' && (!rsvp.menu || rsvp.guests.some(g => !g.name || !g.menu)))"
              class="w-full bg-[#2e2118] text-white rounded-xl py-4 font-['Montserrat'] text-[10px] tracking-[0.3em] uppercase disabled:opacity-40 transition-opacity duration-200 mt-2"
            >Потврди</button>
          </div>

          <div v-else key="thanks" class="flex flex-col items-center gap-4 py-6 text-center">
            <svg width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="#c9a96e" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round">
              <path d="M12 21.593c-5.63-5.539-11-10.297-11-14.402C1 3.379 4.068 1 7.5 1c1.65 0 3.28.724 4.5 2.113C13.22 1.724 14.85 1 16.5 1 19.932 1 23 3.379 23 7.191c0 4.105-5.37 8.863-11 14.402z"/>
            </svg>
            <h3 class="font-['Cormorant_Garamond'] font-light text-3xl text-[#2e2118]">Ви благодариме!</h3>
            <p class="font-['Cormorant_Garamond'] italic text-lg text-[#9b8c7a]">
              {{ rsvp.attending === 'yes' ? 'Со нетрпение ве очекуваме на нашиот ден.' : 'Ни е жал што нема да бидете со нас, но ви благодариме.' }}
            </p>
          </div>
        </Transition>
      </div>

      <footer class="bg-[#181818] py-16 px-6 text-center flex flex-col items-center gap-6">

        <!-- Divider top -->
        <div v-scroll-reveal="0" class="flex items-center gap-4 w-full max-w-xs">
          <div class="flex-1 h-px bg-[#c9a96e]/40"></div>
          <svg width="12" height="12" viewBox="0 0 24 24" fill="#c9a96e"><circle cx="12" cy="12" r="5"/></svg>
          <div class="flex-1 h-px bg-[#c9a96e]/40"></div>
        </div>

        <!-- Tagline -->
        <p v-scroll-reveal="100" class="font-['Cormorant_Garamond'] font-light italic text-[#9b8c7a] text-sm tracking-widest uppercase">Славиме љубов</p>

        <!-- Names -->
        <div v-scroll-reveal="200" class="flex flex-col items-center gap-1">
          <h2 class="font-['Cormorant_Garamond'] font-light italic text-5xl text-white leading-tight">Јована</h2>
          <svg xmlns="http://www.w3.org/2000/svg" width="20" height="20" viewBox="0 0 24 24" fill="#c9a96e" stroke="#c9a96e" stroke-width="1"><path d="M19 14c1.49-1.46 3-3.21 3-5.5A5.5 5.5 0 0 0 16.5 3c-1.76 0-3 .5-4.5 2-1.5-1.5-2.74-2-4.5-2A5.5 5.5 0 0 0 2 8.5c0 2.3 1.5 4.05 3 5.5l7 7Z"/></svg>
          <h2 class="font-['Cormorant_Garamond'] font-light italic text-5xl text-white leading-tight">Борис</h2>
        </div>

        <!-- Date -->
        <div v-scroll-reveal="300" class="flex items-center gap-4">
          <div class="w-8 h-px bg-[#c9a96e]/60"></div>
          <p class="font-['Montserrat'] text-[9px] tracking-[0.4em] uppercase text-[#c9a96e]">13 Септември 2026</p>
          <div class="w-8 h-px bg-[#c9a96e]/60"></div>
        </div>

        <!-- Divider bottom -->
        <div v-scroll-reveal="400" class="flex items-center gap-4 w-full max-w-xs">
          <div class="flex-1 h-px bg-[#c9a96e]/40"></div>
          <svg width="12" height="12" viewBox="0 0 24 24" fill="#c9a96e"><circle cx="12" cy="12" r="5"/></svg>
          <div class="flex-1 h-px bg-[#c9a96e]/40"></div>
        </div>

      </footer>

    </div>
  </Transition>
</template>

<style>
html { scroll-behavior: smooth; }

/* ─── Envelope ─── */
.env-screen {
  position: fixed;
  inset: 0;
  z-index: 50;
  background: #f0ebe3;
  overflow: hidden;
}

.env-video {
  position: absolute;
  top: 0; left: 0;
  width: 100%; height: 100%;
  object-fit: cover;
  object-position: center center;
}

.env-hint {
  position: fixed;
  bottom: 28px;
  left: 50%;
  transform: translateX(-50%);
  font-family: 'Montserrat', sans-serif;
  font-size: 16px;
  letter-spacing: 0.4em;
  text-transform: uppercase;
  color: black;
  white-space: nowrap;
  animation: envblink 2s ease-in-out infinite;
}
@keyframes envblink { 0%, 100% { opacity: 1; } 50% { opacity: 0.3; } }

/* ─── Transitions ─── */
.overlay-leave-active { transition: opacity 0.6s ease; pointer-events: none; }
.overlay-leave-to { opacity: 0; }

.invite-enter-active { transition: opacity 0.9s ease; }
.invite-enter-from { opacity: 0; }

/* ─── RSVP transition ─── */
.rsvp-fade-enter-active, .rsvp-fade-leave-active { transition: opacity 0.4s ease; }
.rsvp-fade-enter-from, .rsvp-fade-leave-to { opacity: 0; }

/* ─── Scroll reveal ─── */
.reveal-hidden {
  opacity: 0;
  transform: translateY(28px);
  transition: opacity 0.75s ease, transform 0.75s ease;
}
.reveal-visible {
  opacity: 1;
  transform: translateY(0);
}
</style>