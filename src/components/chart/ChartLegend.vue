<template>
  <StandardLegend
    v-if="showStandardLegend"
    :key="range + '-' + legendVersion"
    :items="legendItems"
    @toggle="toggleDataset"
  />
  <LegendCategoriesGroup
    v-else
    :categorized-items="categorizedLegendItems"
    @toggle="toggleDataset"
  />
</template>

<script>
import StandardLegend from "./StandardLegend.vue";
import LegendCategoriesGroup from "./LegendCategoriesGroup.vue";

export default {
  name: "ChartLegend",
  components: { LegendCategoriesGroup, StandardLegend },
  props: {
    chart: {
      type: Object,
      default: () => null,
    },
    range: {
      type: String,
      default: "day",
    },
  },
  data() {
    return {
      standard: false,
    };
  },
  computed: {
    legendItems() {
      //console.log("Chart or datasets not ready standard", this.chart);
      if (!this.chart || !this.chart.data || !Array.isArray(this.chart.data.datasets)) {
        return [];
      }
      const hidden = this.$store.state.chartLegend.hiddenDatasets;
      return this.chart.data.datasets.map((dataset, index) => ({
        label: dataset.label,
        index,
        category: dataset.category || "component",
        hidden: hidden.includes(dataset.label),
        borderColor: dataset.borderColor,
        borderDash: dataset.borderDash,
      }));
    },
    categorizedLegendItems() {
      if (!this.chart) {
        return { chargepoint: [], vehicle: [], component: [] };
      }
      let categories = {};
      const hidden = this.$store.state.chartLegend.hiddenDatasets;
      console.log("Categorizing legend items, hidden:", hidden);
      if (this.range === "day") {
        categories = { chargepoint: [], vehicle: [], component: [] };
      } else {
        categories = { chargepoint: [], component: [] };
      }
      const datasets = this.chart?.data?.datasets || [];
      datasets.forEach((dataset, index) => {
        const category = dataset.category || "component";
        if (!categories[category]) categories[category] = [];
        categories[category].push({
          label: dataset.label,
          index,
          hidden: hidden.includes(dataset.label),
          borderColor: dataset.borderColor,
          borderDash: dataset.borderDash,
        });
      });
      return categories;
    },
    showStandardLegend() {
      //Standard-Legende for less than 20 Items or if range is month or year
      return this.legendItems.length < 15;
      //|| this.range === "month" || this.range === "year";
    },
  },
  watch: {
    chart(newChart) {
      if (newChart && newChart.data && Array.isArray(newChart.data.datasets)) {
        // apply the Default-Hidden datasets when changing the time range
        const defaultHidden = newChart.data.datasets
          .filter((dataset) => dataset.hidden)
          .map((dataset) => dataset.label);
        this.$store.commit("chartLegend/setHiddenDatasets", defaultHidden);
        this.applyHiddenDatasetsToChart();
      }
    },
  },
  mounted() {
    console.log("ChartLegend mounted, chart:", this.chart);
    const waitForChart = () => {
      if (this.chart && this.chart.data && Array.isArray(this.chart.data.datasets)) {
        const defaultHiddenDatasets = this.chart.data.datasets
          .filter((dataset) => dataset.hidden)
          .map((dataset) => dataset.label);
        if (defaultHiddenDatasets.length) {
          this.$store.commit("chartLegend/setHiddenDatasets", defaultHiddenDatasets);
        }
        this.applyHiddenDatasetsToChart();
      } else {
        setTimeout(waitForChart, 50);
      }
    };
    waitForChart();
  },
  methods: {
    toggleDataset(indexOrLabel) {
      if (!this.chart) return;
      // Hole das Label des Datasets
      const dataset =
        typeof indexOrLabel === "number"
          ? this.chart.data.datasets[indexOrLabel]
          : this.chart.data.datasets.find((dataset) => dataset.label === indexOrLabel);

      if (!dataset) return;

      // Toggle im Store
      this.$store.commit("chartLegend/toggleDataset", dataset.label);

      // Hidden-Status aus Store anwenden
      this.applyHiddenDatasetsToChart();

      //this.legendVersion++; // erzwingt Neuberechnung
    },
    applyHiddenDatasetsToChart() {
      if (!this.chart || !this.chart.data) return;
      const hidden = this.$store.state.chartLegend.hiddenDatasets;
      this.chart.data.datasets.forEach((ds) => {
        ds.hidden = hidden.includes(ds.label);
      });
      this.chart.update();
    },
  },
};
</script>
