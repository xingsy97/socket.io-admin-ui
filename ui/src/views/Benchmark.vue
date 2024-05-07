<template>
  <div>
    <v-breadcrumbs :items="breadcrumbItems" />

    <v-container fluid>
      <v-row>

        <v-col cols="6" md="8">
          <BenchmarkHistogram />
        </v-col>
      </v-row>
    </v-container>
  </div>
</template>

<script>
import BenchmarkHistogram from "../components/Benchmark/BenchmarkHistogram";
import { mapGetters } from "vuex";

export default {
  name: "Benchmark",

  components: {
    BenchmarkHistogram,
},

  computed: {
    breadcrumbItems() {
      return [
        {
          text: this.$t("benchmark.title"),
          disabled: true,
        },
      ];
    },
    ...mapGetters("config", ["hasAggregatedValues"]),
  },
};
</script>
