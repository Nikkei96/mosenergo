<script>
import Papa from "papaparse";

export default {
  data() {
    return {
      // hourlyConsumption: [],
      progress: 0,
      isLoading: false,
    };
  },
  methods: {
    selectFile(file) {
      if (!file) return;
      this.isLoading = true;
      let i = 0;
      // const totalIterations = 1539625;
      const totalIterations = 153985;
      const hourlyConsumption = [];

      Papa.parse(file, {
        header: true,
        step: (results) => {
          const { registration_date, hour } = results.data;
          results.data.unixTime = new Date(`${registration_date} ${hour}:00`).getTime() / 1000;
          hourlyConsumption.push(results.data);
          i++;
          this.progress = Math.floor((i / totalIterations) * 100);

          if (i >= totalIterations) {
            this.isLoading = false;
            hourlyConsumption.sort((hc1, hc2) => hc1.unixTime - hc2.unixTime);
            this.generateChartData(hourlyConsumption);
          }
        },
      });
    },

    generateChartData(hourlyConsumption) {
      const chartData = [];
      let energy_taken_kVth = 0;
      let max_energy_taken_kVth = 0;

      for (let i = 1; i < hourlyConsumption.length; i++) {
        if (i === 1) {
          energy_taken_kVth += Number(hourlyConsumption[1].energy_taken_kVth);
        } else if (hourlyConsumption[i].unixTime === hourlyConsumption[i - 1].unixTime) {
          energy_taken_kVth += Number(hourlyConsumption[i].energy_taken_kVth);
        } else {
          chartData.push({
            energy_taken_kVth: Math.floor(energy_taken_kVth),
            unixTime: new Date(hourlyConsumption[i - 1].unixTime * 1000),
          });
          if (energy_taken_kVth > max_energy_taken_kVth) {
            max_energy_taken_kVth = Math.floor(energy_taken_kVth);
          }
          energy_taken_kVth = 0;
        }
      }

      chartData.map((cd) => {
        cd.energy_taken_kVth = Math.floor((cd.energy_taken_kVth / max_energy_taken_kVth) * 100);
      });

      console.log(chartData);
    },
  },
};
</script>

<template>
  <div>
    <h1 class="text-h5 mb-4">Анализ данных</h1>
    <v-row>
      <v-col>
        <v-label>Выберите файл с данными часового потребления ЮЛ</v-label>
      </v-col>
      <v-col cols="9">
        <v-file-input
          :disabled="isLoading"
          accept=".csv"
          hide-details
          label="csv-file"
          show-size
          outlined
          dense
          @change="selectFile"
        />
        <v-progress-linear class="mt-1" v-if="isLoading" v-model="progress" height="20">
          <strong>Анализ данных: {{ progress }}%</strong>
        </v-progress-linear>
      </v-col>
    </v-row>
  </div>
</template>
