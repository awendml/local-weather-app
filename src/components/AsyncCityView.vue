<template>
  <div class="flex flex-col flex-1 items-center">
    <!-- Cuaca Saat Ini -->
    <div class="flex flex-col items-center text-white py-12">
      <h1 class="text-4xl mb-2">{{ route.params.city }}</h1>
      <p class="text-sm mb-12">
        {{
          new Date(weatherData?.currentTime).toLocaleDateString("in", {
            weekday: "short",
            day: "2-digit",
            month: "long",
          })
        }}
        {{
          new Date(weatherData?.currentTime).toLocaleTimeString("in", {
            timeStyle: "short",
          })
        }}
      </p>
      <p class="text-8xl mb-8">
        {{ Math.round(weatherData?.main?.temp) }}&deg;F
      </p>
      <p class="text-sm">
        Feels like
        {{ Math.round(weatherData?.main?.feels_like) }}&deg;
      </p>
      <p class="capitalize">
        {{ weatherData?.weather[0]?.description }}
      </p>
      <img :src="`http://openweathermap.org/img/wn/${weatherData?.weather[0]?.icon}@2x.png`" alt="Weather Icon">
    </div>

    <hr class="border-white border-opacity-10 border w-full">

    <!-- Hourly weather -->
    <div class="max-w-screen-md w-full py-12">
      <div class="mx-8 text-white">
        <h2 class="mb-4">Hourly Weather</h2>
        <div class="flex gap-10 overflow-x-scroll scroll-smooth scrollbar-thumb-gray-500 scrollbar-track-gray-800">
          <div
            v-for="(hourData, index) in hourlyWeather"
            :key="index"
            class="flex flex-col gap-4 items-center"
          >
            <p class="whitespace-nowrap text-md">
              {{
                new Date(hourData.time).toLocaleTimeString("in", {
                  hour: "numeric",
                })
              }}
            </p>
            <img
              class="w-auto h-[50px] object-cover"
              :src="`http://openweathermap.org/img/wn/${getIconCode(hourData.icon)}@2x.png`"
              alt="Weather Icon"
            />
            <p class="text-xl">
              {{ Math.round(hourData.temp) }}&deg;
            </p>
          </div>
        </div>
      </div>
    </div>
    <hr class="border-white border-opacity-10 border w-full">

    <!-- Weekly Weather -->
    <div class="max-w-screen-md w-full py-12">
      <div class="mx-8 text-white">
        <h2 class="mb-4">7 Day Forecast</h2>
        <div v-for="(dayData, index) in weeklyWeather" :key="index" class="flex items-center">
          <p class="flex-1">
            {{
              new Date(dayData.date).toLocaleDateString("in", {
                weekday: "long",  // Nama hari
                year: "numeric",  // Tahun
                day: "2-digit",   // Tanggal
                month: "long",    // Nama bulan
              })
            }}
          </p>
          <img
            class="w-[50px] h-[50px] object-cover"
            :src="`http://openweathermap.org/img/wn/${getIconCode(dayData.icon)}@2x.png`"
            alt="Weather Icon"
          />
          <div class="flex gap-2 flex-1 justify-end">
            <p>H: {{ Math.round(dayData.tempMax) }}&deg;</p>
            <p>L: {{ Math.round(dayData.tempMin) }}&deg;</p>
          </div>
        </div>
      </div>
    </div>

    <div class="flex items-center gap-2 py-12 text-white cursor-pointer duration-150 hover:text-red-500 "
    @click="removeCity"
    >
      <i class="fa-solid fa-trash"></i>
      <p>Remove City</p>
    </div>

  </div>
</template>

<script setup>
import axios from 'axios';
import { ref, onMounted } from 'vue';
import { useRoute, useRouter } from 'vue-router';

const route = useRoute();
const weatherData = ref(null); // Menyimpan data dari OpenWeatherMap
const hourlyWeather = ref([]); // Menyimpan data hourly dari Open-Meteo
const weeklyWeather = ref([]); // Menyimpan data weekly dari Open-Meteo
const errorMessage = ref(""); // Menyimpan pesan error

const getWeatherData = async () => {
  try {
    const weatherResponse = await axios.get(
      `https://api.openweathermap.org/data/2.5/weather?lat=${route.query.lat}&lon=${route.query.lng}&appid=25801211b00911993674a12b75dc32ee&units=imperial`
    );
    weatherData.value = weatherResponse.data;

    const openMeteoResponse = await axios.get(
      `https://api.open-meteo.com/v1/forecast?latitude=${route.query.lat}&longitude=${route.query.lng}&hourly=temperature_2m,apparent_temperature,weathercode&daily=temperature_2m_max,temperature_2m_min,weathercode`
    );

    // Log the response to inspect the structure
    console.log("Open Meteo Response:", openMeteoResponse.data);

    // Check if 'hourly' data is valid
    if (Array.isArray(openMeteoResponse.data.hourly?.time)) {
      hourlyWeather.value = openMeteoResponse.data.hourly.time.map((time, index) => ({
        time: time,
        temp: openMeteoResponse.data.hourly.temperature_2m[index],
        icon: openMeteoResponse.data.hourly.weathercode[index],
      }));
    } else {
      throw new Error("Hourly data is not valid.");
    }

    // Extract daily data (temperature max and min)
    if (Array.isArray(openMeteoResponse.data.daily?.time)) {
      weeklyWeather.value = openMeteoResponse.data.daily.time.map((time, index) => ({
        date: new Date(time).toLocaleDateString("in", {
          weekday: "short",
          day: "2-digit",
          month: "long",
        }),
        tempMax: openMeteoResponse.data.daily.temperature_2m_max[index],
        tempMin: openMeteoResponse.data.daily.temperature_2m_min[index],
        icon: openMeteoResponse.data.daily.weathercode[index],

      }));
    } else {
      throw new Error("Daily data is not available or in incorrect format.");
    }

    const localOffset = new Date().getTimezoneOffset() * 60000;
    const utcTime = weatherData.value.dt * 1000 + localOffset;
    weatherData.value.currentTime = new Date(
      utcTime + 1000 * weatherData.value.timezone
    ).toISOString();
  } catch (error) {
    console.error("Error fetching weather data:", error);
    errorMessage.value = "Failed to load weather data.";
  }
};

// Fungsi untuk mendapatkan kode icon yang tepat berdasarkan kode weather
const getIconCode = (weatherCode) => {
  const iconMapping = {
    0: '01d', // Clear sky
    1: '02d', // Few clouds
    2: '03d', // Scattered clouds
    3: '04d', // Broken clouds
    45: '50d', // Mist
    51: '50d', // Light rain
    53: '50d', // Moderate rain
    55: '50d', // Heavy rain
    61: '09d', // Showers
    63: '09d', // Showers
    80: '10d', // Thunderstorms
    // Add more mappings for weather conditions
  };

  return iconMapping[weatherCode] || '01d'; // Default to 'clear sky' if not found
};

onMounted(getWeatherData);

const router =useRouter();
const removeCity = () => {
  const cities = JSON.parse(localStorage.getItem('savedCities'));
  
  const updatedCities = cities.filter(
    (city) => city.id !== route.query.id
  );
  localStorage.setItem(
    'savedCities',
    JSON.stringify(updatedCities)
  );
  router.push({
    name: "home",
  });
};
</script>
