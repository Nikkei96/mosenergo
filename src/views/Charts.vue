<script>
import Papa from "papaparse";
import Highcharts from "highcharts";
import stockInit from "highcharts/modules/stock";
stockInit(Highcharts);

export default {
  data() {
    return {
      chartOptions: {
        title: {
          text: "График медианных значений потребления электроэнергии по всем юридическим лицам",
        },
        xAxis: {
          type: "datetime",
          title: {
            text: "Дата",
          },
        },
        yAxis: {
          title: {
            text: "Медианное значение потребления электроэнергии",
          },
        },
        series: [
          {
            showInLegend: false,
            data: [],
            // point: {
            //   events: {
            //     mouseOver: (event) => {
            //       this.sync(event, "over");
            //     },
            //     mouseOut: (event) => {
            //       this.sync(event, "out");
            //     },
            //   },
            // },
          },
        ],
      },

      chartOptionsCompany: {
        title: {
          text: "",
        },
        xAxis: {
          type: "datetime",
          title: {
            text: "Дата",
          },
        },
        yAxis: {
          title: {
            text: "Медианное значение потребления электроэнергии",
          },
        },
        series: [
          {
            showInLegend: false,
            data: [],
            // point: {
            //   events: {
            //     mouseOver: (event) => {
            //       this.sync(event, "over");
            //     },
            //     mouseOut: (event) => {
            //       this.sync(event, "out");
            //     },
            //   },
            // },
          },
        ],
      },

      progress: 0,
      isLoading: false,
      sortedHourlyConsumption: [],
      priceCategory: [],
      company: [],
      okved: {},
      selectedCompany: null,
      hourlyConsumptionFull: [],
      reinit: false,
    };
  },
  computed: {
    okvedKeys() {
      return Object.keys(this.okved);
    },
  },
  watch: {
    selectedCompany() {
      if (this.selectedCompany) {
        this.chartOptionsCompany.title.text = `График медианных значений потребления электроэнергии для отрасли: ${this.selectedCompany.okved_name}`;
      }
    },
  },
  methods: {
    selectFile(file) {
      if (!file) {
        this.chartOptions.series[0].data = [];
        this.progress = 0;
        return;
      }
      this.isLoading = true;
      let i = 0;
      const totalIterations = 1539625;
      // const totalIterations = 153985;
      const hourlyConsumption = [];

      Papa.parse(file, {
        header: true,
        step: (results) => {
          const { registration_date, hour } = results.data;
          results.data.unixTime = new Date(`${registration_date} ${hour}:00`).getTime() / 1000;
          this.hourlyConsumptionFull.push(results.data);
          hourlyConsumption.push(results.data);
          i++;
          this.progress = Math.floor((i / totalIterations) * 100);

          if (i >= totalIterations) {
            this.isLoading = false;
            this.sortedHourlyConsumption = hourlyConsumption.sort(
              (hc1, hc2) => hc1.unixTime - hc2.unixTime
            );
            this.generateChartData(hourlyConsumption, this.chartOptions.series[0].data);
          }
        },
      });
    },

    generateChartData(hourlyConsumption, target) {
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
            unixTime: hourlyConsumption[i - 1].unixTime,
          });
          if (energy_taken_kVth > max_energy_taken_kVth) {
            max_energy_taken_kVth = Math.floor(energy_taken_kVth);
          }
          energy_taken_kVth = 0;
        }
      }

      const cd = chartData.map((cd) => {
        const energy_taken_kVth = Math.floor((cd.energy_taken_kVth / max_energy_taken_kVth) * 100);
        return [cd.unixTime * 1000 + 10800 * 1000, energy_taken_kVth];
      });

      target.push(...cd);
    },

    setOkved(file) {
      if (!file) {
        this.okved = {};
        return;
      }

      Papa.parse(file, {
        header: true,
        complete: (results) => {
          let resObj = {};
          results.data.forEach((el) => {
            resObj[el.id] = el.okved_name;
          });
          this.okved = resObj;
        },
      });
    },

    setPriceCategory(file) {
      if (!file) {
        this.priceCategory = [];
        return;
      }

      let company = [];

      Papa.parse(file, {
        header: true,
        complete: (results) => {
          this.priceCategory = results.data.filter((r) => Number(r.useful_vacation_kVt > 0));
          this.priceCategory.forEach((pc) => {
            if (typeof this.okved[pc.id] !== "undefined") {
              company.push({
                id: pc.id,
                okved_name: this.okved[pc.id],
              });
            }
          });
          this.company = company;
        },
      });
    },

    createChart() {
      this.chartOptionsCompany.series[0].data = [];
      setTimeout(() => {
        let hcCash = this.hourlyConsumptionFull.filter((hc) => hc.id == this.selectedCompany.id);
        hcCash = hcCash.sort((hc1, hc2) => hc1.unixTime - hc2.unixTime);
        this.generateChartData(hcCash, this.chartOptionsCompany.series[0].data);
      }, 100);
    },

    sync(event, type) {
      // console.log(this.$refs.highcharts);
      // this.$refs.highcharts.forEach(({ chart }) => {
      //   chart.series.forEach((series) => {
      //     series.data.forEach((point) => {
      //       if (type === "over") {
      //         point.setState("hover");
      //         chart.tooltip.refresh(point);
      //         chart.xAxis[0].drawCrosshair(event, point);
      //       } else {
      //         point.setState();
      //         chart.tooltip.hide();
      //         chart.xAxis[0].hideCrosshair();
      //       }
      //     });
      //   });
      // });
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

    <v-row v-if="chartOptions.series[0].data.length">
      <v-col>
        <highcharts :options="chartOptions" ref="highcharts" :constructor-type="'stockChart'" />
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-label>Выберите файл с ОКВЭД по ЮЛ </v-label>
      </v-col>
      <v-col cols="9">
        <v-file-input
          :disabled="!chartOptions.series[0].data.length"
          accept=".csv"
          hide-details
          label="csv-file"
          show-size
          outlined
          dense
          @change="setOkved"
        />
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-label>Выберите файл с ценовыми категориями </v-label>
      </v-col>
      <v-col cols="9">
        <v-file-input
          :disabled="!okvedKeys.length"
          accept=".csv"
          hide-details
          label="csv-file"
          show-size
          outlined
          dense
          @change="setPriceCategory"
        />
      </v-col>
    </v-row>

    <v-row>
      <v-col>
        <v-select
          v-if="company.length"
          v-model="selectedCompany"
          @change="createChart"
          :items="company"
          label="Выберите отрасль"
          item-text="okved_name"
          return-object
          dense
          outlined
        ></v-select>
      </v-col>
    </v-row>

    <v-row v-if="chartOptionsCompany.series[0].data.length && !reinit">
      <v-col>
        <highcharts
          ref="highcharts"
          :options="chartOptionsCompany"
          :constructor-type="'stockChart'"
        />
      </v-col>
    </v-row>
  </div>
</template>
