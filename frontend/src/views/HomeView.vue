<script setup>
import ModelView from '@/views/ModelView.vue';
import LineChart from '@/components/graphs/LineChart.vue';
import Api from '@/lib/api';
import { ref, onMounted } from 'vue';

// Import images directly
import lightOn from '@/assets/light-on.png';
import lightOff from '@/assets/light-off.png';
import waterOn from '@/assets/water-on.png';
import windOn from '@/assets/wind-on.png';
import windOff from '@/assets/wind-off.png';

// Convert reactive references for chart data
const labels = ref([]);
const temperatureData = ref({
  labels: [],
  datasets: [
    {
      label: 'Temperature (°C)',
      data: [],
      fill: true,
      borderColor: '#FF6B6B',
      backgroundColor: 'rgba(255, 107, 107, 0.2)',
      tension: 0.4
    }
  ]
});

const humidityData = ref({
  labels: [],
  datasets: [
    {
      label: 'Humidity (%)',
      data: [],
      fill: true,
      borderColor: '#48DBFB',
      backgroundColor: 'rgba(72, 219, 251, 0.2)',
      tension: 0.4
    }
  ]
});

// Keep the soil moisture sample data as is
const soilMoistureData = {
  labels: ['Jan', 'Feb', 'Mar', 'Apr', 'May', 'Jun', 'Jul', 'Aug', 'Sep', 'Oct', 'Nov', 'Dec'],
  datasets: [
    {
      label: 'Soil Moisture (%)',
      data: [70, 65, 60, 55, 50, 45, 40, 38, 35, 33, 30, 28],
      fill: true,
      borderColor: '#1DD1A1',
      backgroundColor: 'rgba(29, 209, 161, 0.2)',
      tension: 0.4
    }
  ]
};

// Common base chart options
const baseChartOptions = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: {
    legend: {
      display: false
    },
    tooltip: {
      mode: 'index',
      intersect: false,
      backgroundColor: 'rgba(20, 30, 40, 0.9)',
      titleColor: '#fff',
      bodyColor: '#fff',
      borderColor: 'rgba(0, 229, 255, 0.3)',
      borderWidth: 1
    }
  },
  scales: {
    x: {
      display: false
    },
    y: {
      display: true,
      grid: {
        color: 'rgba(255, 255, 255, 0.05)',
        borderDash: [2, 4]
      },
      ticks: {
        color: 'rgba(255, 255, 255, 0.5)',
        font: {
          size: 9
        },
        padding: 10,
        maxTicksLimit: 5
      }
    }
  },
  elements: {
    point: {
      radius: 2,
      hoverRadius: 4,
      borderWidth: 2
    },
    line: {
      borderWidth: 2
    }
  },
  animation: {
    duration: 800,
    easing: 'easeOutQuart'
  }
};

// Temperature chart options with fixed Y axis from -5 to 50
const temperatureChartOptions = {
  ...JSON.parse(JSON.stringify(baseChartOptions)),
  scales: {
    ...JSON.parse(JSON.stringify(baseChartOptions.scales)),
    y: {
      ...JSON.parse(JSON.stringify(baseChartOptions.scales.y)),
      min: -5,
      max: 50,
      suggestedMin: -5,
      suggestedMax: 50
    }
  }
};

// Humidity chart options with fixed Y axis from 0 to 100
const humidityChartOptions = {
  ...JSON.parse(JSON.stringify(baseChartOptions)), 
  scales: {
    ...JSON.parse(JSON.stringify(baseChartOptions.scales)),
    y: {
      ...JSON.parse(JSON.stringify(baseChartOptions.scales.y)),
      min: 0,
      max: 100,
      suggestedMin: 0,
      suggestedMax: 100
    }
  }
};

// Soil moisture chart options (using the original chart options)
const soilMoistureChartOptions = JSON.parse(JSON.stringify(baseChartOptions));

const lifeformData = ref(null);
const showFallbackImage = ref(false);
const videoElement = ref(null);

// Add a reactive reference for atmospherics data
const atmosphericsData = ref({
  sunlight: { status: 0, r: 0, g: 0, b: 0, brightness: 0 },
  wind: { status: 0, speed: 0 }
});

const handleVideoError = () => {
  console.error('Video stream failed to load');
  showFallbackImage.value = true;
};

const fetchLifeformData = async () => {
  try {
    const response = await Api.get('/lifeform/');
    lifeformData.value = response;
  } catch (error) {
    console.error('Error fetching lifeform data:', error);
  }
};

const fetchTempHumidityData = async () => {
  try {
    const response = await Api.get('/temphumidity/');
    
    if (response && response.length > 0) {
      // Sort data chronologically
      const sortedData = [...response].sort((a, b) => new Date(a.timestamp) - new Date(b.timestamp));
      
      // Extract timestamps, temperatures and humidity values
      const timestamps = sortedData.map(entry => {
        const date = new Date(entry.timestamp);
        return date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });
      });
      const temperatures = sortedData.map(entry => entry.temperature);
      const humidities = sortedData.map(entry => entry.humidity);
      
      // Update chart data
      temperatureData.value = {
        labels: timestamps,
        datasets: [
          {
            label: 'Temperature (°C)',
            data: temperatures,
            fill: true,
            borderColor: '#FF6B6B',
            backgroundColor: 'rgba(255, 107, 107, 0.2)',
            tension: 0.4
          }
        ]
      };
      
      humidityData.value = {
        labels: timestamps,
        datasets: [
          {
            label: 'Humidity (%)',
            data: humidities,
            fill: true,
            borderColor: '#48DBFB',
            backgroundColor: 'rgba(72, 219, 251, 0.2)',
            tension: 0.4
          }
        ]
      };
    }
    
    console.log('Temperature & humidity data updated:', response);
  } catch (error) {
    console.error('Error fetching temperature/humidity data:', error);
  }
};

const fetchAtmosphericsData = async () => {
  try {
    const response = await Api.get('/atmospherics/');
    if (response) {
      atmosphericsData.value = response;
      console.log('Atmospherics data updated:', response);
    }
  } catch (error) {
    console.error('Error fetching atmospherics data:', error);
  }
};

onMounted(async () => {
  // Initial data loading
  await fetchLifeformData();
  await fetchTempHumidityData();
  await fetchAtmosphericsData();

  // Attempt to reload video once after a delay
  setTimeout(() => {
    if (videoElement.value) {
      videoElement.value.load();
    }
  }, 1000);
  
  // Refresh all data every 11 minutes (60000 ms)
  setInterval(async () => {
    await fetchLifeformData();
    await fetchTempHumidityData();
    await fetchAtmosphericsData();
 
    console.log('Data refreshed at:', new Date().toLocaleTimeString());
  }, 60000);
});
</script>
<template>
    <div class="top-container">
        <div class="left-container">
            <div class="left-container-content">
                <h1 v-if="lifeformData">LIFEFORM: <span class="lifeform-name">{{ lifeformData.lifeform }}</span></h1>
                <img src="http://127.0.0.1:8000/map/" alt="map" class="map-img">
            </div>
        </div>
        <div class="right-container">
            <img  
                src="http://127.0.0.1:8000/camera/stream/0/" 
                alt="Camera feed" 
                class="camera-feed"

            >
        </div>
    </div>
    <div class="graph-container">
        <div class="graph-item-1">
            <h1 class="lower_graph_h1">Temperature</h1>
            <LineChart 
                :chart-data="temperatureData"
                :chart-options="temperatureChartOptions"
                :height="100"
                css-classes="chart-container"
                :styles="{ width: '100%' }"
            />
        </div>
        <div class="graph-item-2">
            <h1 class="lower_graph_h1">Humidity</h1>
            <LineChart 
                :chart-data="humidityData"
                :chart-options="humidityChartOptions"
                :height="100"
                css-classes="chart-container"
                :styles="{ width: '100%' }"
            />
        </div>
        <div class="graph-item-3" style="background-color: rgba(19, 34, 36);">
            <h1 class="lower_graph_h1">ATMOSPHERICS</h1>
            
            <div class="atmos-container">
                <div class="atmos-item">
                    Sunlight Simulation
                    <img 
                      :src="atmosphericsData.sunlight && atmosphericsData.sunlight.status ? lightOn : lightOff" 
                      alt="Sunlight" 
                      class="control-icon"
                    >
                    <span class="atmos-item-text">
                      {{ atmosphericsData.sunlight && atmosphericsData.sunlight.status ? 'ENABLED' : 'DISABLED' }}
                      <span v-if="atmosphericsData.sunlight && atmosphericsData.sunlight.status" class="atmos-item-detail">
                        ({{ Math.round(atmosphericsData.sunlight.brightness * 100) }}%)
                      </span>
                    </span>
                </div>
                <div class="atmos-item">
                    Soil Moisture
                    <img :src="waterOn" alt="Soil Moisture" class="control-icon">
                    <span class="atmos-item-text">DETECTED</span>
                </div>
                <div class="atmos-item">
                    Wind Simulation
                    <img 
                      :src="atmosphericsData.wind && atmosphericsData.wind.status ? windOn : windOff" 
                      alt="Wind" 
                      class="control-icon"
                    >
                    <span class="atmos-item-text">
                      {{ atmosphericsData.wind && atmosphericsData.wind.status ? 
                         atmosphericsData.wind.speed.toFixed(1) + 'm/s' : 
                         'DISABLED' }}
                    </span>
                </div>
            </div>
        </div>
    </div>

</template>

<style scoped>
/* Add necessary container styling */
.right-container {
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
}

.camera-feed {
    max-width: 100%;
    max-height: 100%;
}

.atmos-item-detail {
    font-size: 0.8em;
    opacity: 0.7;
    margin-left: 5px;
}
</style>