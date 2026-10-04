<script setup lang="ts">
import { ref, onMounted } from 'vue'

const city = ref<string>(localStorage.getItem('lastCity') || '')
const weather = ref<any>(null)
const forecast = ref<any[]>([])
const loading = ref<boolean>(false)
const error = ref<string>('')

const apiKey = import.meta.env.VITE_WEATHER_API_KEY

const getOutfitAdvice = (temp: number, description: string, windSpeed: number): string => {
  const advice: string[] = []
  
  if (temp <= -10) {
    advice.push('Теплая зимняя куртка или пуховик')
    advice.push('Шапка, шарф и варежки')
    advice.push('Утепленные сапоги')
  } else if (temp <= 0) {
    advice.push('Зимняя куртка')
    advice.push('Шапка и шарф')
  } else if (temp <= 10) {
    advice.push('Пальто или демисезонная куртка')
    advice.push('Легкий шарф')
  } else if (temp <= 18) {
    advice.push('Легкая куртка или ветровка')
    advice.push('Кофта или свитер')
  } else if (temp <= 25) {
    advice.push('Футболка или легкая блузка')
    advice.push('Джинсы или легкие брюки')
  } else {
    advice.push('Легкая одежда из натуральных тканей')
    advice.push('Солнцезащитные очки')
    advice.push('Вода с собой')
  }

  if (windSpeed > 7) advice.push('Непродуваемая одежда')

  const desc = description.toLowerCase()
  if (desc.includes('дождь') || desc.includes('ливень')) {
    advice.push('Зонт')
  } else if (desc.includes('снег')) {
    advice.push('Теплая одежда')
  }

  return advice.join(', ')
}

const getWeather = async () => {
  if (!city.value.trim()) {
    error.value = 'Введите название города'
    return
  }

  localStorage.setItem('lastCity', city.value)
  loading.value = true
  error.value = ''
  weather.value = null
  forecast.value = []

  try {
    const response = await fetch(
      `https://api.openweathermap.org/data/2.5/weather?q=${city.value}&appid=${apiKey}&units=metric&lang=ru`
    )
    if (!response.ok) throw new Error('Город не найден')
    weather.value = await response.json()

    const forecastResponse = await fetch(
      `https://api.openweathermap.org/data/2.5/forecast?q=${city.value}&appid=${apiKey}&units=metric&lang=ru`
    )
    const forecastData = await forecastResponse.json()
    forecast.value = forecastData.list.filter((item: any) => item.dt_txt.includes("12:00:00"))
  } catch (err: any) {
    error.value = err.message
  } finally {
    loading.value = false
  }
}

const getLocation = () => {
  if (!navigator.geolocation) {
    error.value = 'Геолокация не поддерживается'
    return
  }

  loading.value = true
  error.value = ''
  weather.value = null
  forecast.value = []

  navigator.geolocation.getCurrentPosition(
    async (position) => {
      const { latitude, longitude } = position.coords
      try {
        const response = await fetch(
          `https://api.openweathermap.org/data/2.5/weather?lat=${latitude}&lon=${longitude}&appid=${apiKey}&units=metric&lang=ru`
        )
        if (!response.ok) throw new Error('Не удалось получить погоду')
        const data = await response.json()
        
        city.value = data.name
        weather.value = data
        localStorage.setItem('lastCity', data.name)

        const forecastResponse = await fetch(
          `https://api.openweathermap.org/data/2.5/forecast?lat=${latitude}&lon=${longitude}&appid=${apiKey}&units=metric&lang=ru`
        )
        const forecastData = await forecastResponse.json()
        forecast.value = forecastData.list.filter((item: any) => item.dt_txt.includes("12:00:00"))
      } catch (err: any) {
        error.value = err.message
      } finally {
        loading.value = false
      }
    },
    (geoError) => {
      error.value = 'Не удалось определить местоположение'
      loading.value = false
    }
  )
}

onMounted(() => {
  if (city.value) {
    getWeather()
  }
})
</script>

<template>
  <div class="min-h-screen relative overflow-hidden" style="background: linear-gradient(180deg, #ff9a76 0%, #ff6b6b 30%, #4a4a4a 70%, #2d2d2d 100%)">
    <!-- Горы на фоне -->
    <svg class="absolute bottom-0 w-full h-64 opacity-30" viewBox="0 0 1200 300" preserveAspectRatio="none">
      <path d="M0,300 L0,200 Q150,100 300,180 T600,150 T900,200 T1200,180 L1200,300 Z" fill="#1a1a1a" />
      <path d="M0,300 L0,250 Q200,150 400,220 T800,200 T1200,250 L1200,300 Z" fill="#0d0d0d" />
    </svg>

    <div class="relative z-10 container mx-auto px-6 py-8 max-w-6xl">
      <!-- Шапка -->
      <div class="flex items-center justify-between mb-12">
        <h1 class="text-2xl font-bold text-white tracking-wider">ПОГОДА</h1>
        <div class="flex gap-4">
          <input
            v-model="city"
            type="text"
            placeholder="Город"
            @keydown.enter="getWeather"
            class="px-4 py-2 bg-white/10 backdrop-blur-sm border border-white/20 rounded-lg text-white placeholder-white/50 focus:outline-none focus:ring-2 focus:ring-white/30"
          />
          <button
            @click="getWeather"
            :disabled="loading"
            class="px-6 py-2 bg-white/20 backdrop-blur-sm border border-white/30 rounded-lg text-white hover:bg-white/30 transition disabled:opacity-50"
          >
            {{ loading ? '...' : 'Найти' }}
          </button>
          <button
            @click="getLocation"
            :disabled="loading"
            class="px-6 py-2 bg-white/10 backdrop-blur-sm border border-white/20 rounded-lg text-white hover:bg-white/20 transition disabled:opacity-50"
          >
            Мое местоположение
          </button>
        </div>
      </div>

      <div v-if="error" class="mb-8 p-4 bg-red-500/20 backdrop-blur-sm border border-red-500/30 rounded-lg text-white">
        {{ error }}
      </div>

      <template v-if="weather">
        <!-- Основная информация о погоде -->
        <div class="grid grid-cols-1 md:grid-cols-3 gap-8 mb-12">
          <!-- Температура и иконка -->
          <div class="md:col-span-1">
            <div class="text-white/70 text-sm mb-2">
              {{ new Date().toLocaleDateString('ru-RU', { weekday: 'long', day: 'numeric', month: 'long', hour: '2-digit', minute: '2-digit' }) }}
            </div>
            <div class="flex items-center gap-6">
              <img
                :src="`https://openweathermap.org/img/wn/${weather.weather[0].icon}@4x.png`"
                alt="icon"
                class="w-32 h-32"
              />
              <div>
                <div class="text-7xl font-bold text-yellow-400">
                  +{{ Math.round(weather.main.temp) }}°
                </div>
                <div class="text-white/70 text-lg mt-2">
                  Ощущается как {{ Math.round(weather.main.feels_like) }}°
                </div>
              </div>
            </div>
            <div class="mt-4 text-white/60 text-sm">
              {{ weather.weather[0].description }}
            </div>
          </div>

          <!-- Детали -->
          <div class="md:col-span-1">
            <h3 class="text-white/50 text-sm font-semibold mb-4 tracking-wider">ПОДРОБНЕЕ</h3>
            <div class="space-y-3 text-white/80">
              <div class="flex justify-between">
                <span>Скорость ветра:</span>
                <span class="font-semibold">{{ weather.wind.speed }} м/с</span>
              </div>
              <div class="flex justify-between">
                <span>Влажность:</span>
                <span class="font-semibold">{{ weather.main.humidity }}%</span>
              </div>
              <div class="flex justify-between">
                <span>Давление:</span>
                <span class="font-semibold">{{ Math.round(weather.main.pressure * 0.750062) }} мм</span>
              </div>
            </div>
          </div>

          <!-- Советы по одежде -->
          <div class="md:col-span-1">
            <h3 class="text-white/50 text-sm font-semibold mb-4 tracking-wider">ЧТО НАДЕТЬ</h3>
            <div class="bg-white/10 backdrop-blur-sm border border-white/20 rounded-lg p-4">
              <p class="text-white/90 text-sm leading-relaxed">
                {{ getOutfitAdvice(weather.main.temp, weather.weather[0].description, weather.wind.speed) }}
              </p>
            </div>
          </div>
        </div>

        <!-- Прогноз на 5 дней -->
        <div v-if="forecast.length > 0" class="border-t border-white/20 pt-8">
          <h3 class="text-white/50 text-sm font-semibold mb-6 tracking-wider">ПРОГНОЗ НА 5 ДНЕЙ</h3>
          <div class="grid grid-cols-5 gap-4">
            <div v-for="day in forecast" :key="day.dt" class="text-center">
              <div class="text-white/70 text-sm font-semibold mb-2">
                {{ new Date(day.dt * 1000).toLocaleDateString('ru-RU', { weekday: 'long' }) }}
              </div>
              <div class="text-white/50 text-xs mb-3">
                {{ new Date(day.dt * 1000).toLocaleDateString('ru-RU', { day: 'numeric', month: 'short' }) }}
              </div>
              <img
                :src="`https://openweathermap.org/img/wn/${day.weather[0].icon}.png`"
                alt="icon"
                class="w-16 h-16 mx-auto mb-2"
              />
              <div class="text-yellow-400 text-2xl font-bold">
                +{{ Math.round(day.main.temp) }}°
              </div>
            </div>
          </div>
        </div>
      </template>

      <div v-if="!weather && !error && !loading" class="text-center text-white/50 mt-32">
        <p class="text-xl">Введите город для просмотра погоды</p>
      </div>
    </div>
  </div>
</template>