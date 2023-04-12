<template>
    <v-card>
        <v-btn color="primary" class="white--text" @click="startBenchmark"> Start Benchmark
        </v-btn>

        <v-btn color="primary" class="white--text" @click="endBenchmark"> End Benchmark
        </v-btn>

        <v-card-title class="text-center">
            {{ $t("dashboard.benchmarkHistogram.title") }}
        </v-card-title>

        <v-card-text>
            <v-row>
                <Bar :chart-data="chartData" :chart-options="chartOptions" style="width: 100%" :height="chartHeight" />
            </v-row>
        </v-card-text>

        <v-card-text>
            <v-row>
                <!-- <Bar :chart-data="chartData" :chart-options="chartOptions" style="width: 100%" :height="chartHeight" /> -->
            </v-row>
        </v-card-text>
    </v-card>
</template>
  
<script>
import colors from "vuetify/lib/util/colors";
import { Bar } from "vue-chartjs/legacy";
// import { Bar } from "vue-chartjs/legacy";
import { io } from "socket.io-client";

export default {
    name: "BenchmarkHistogram",
    components: { Bar },
    // components: { VLine},
    data() {
        return {
            chartHeight: 120,
            chartOptions: {
                parsing: false,
                scales: {
                    x: { type: "linear", beginAtZero: true, suggestedMax: 50, ticks: { precision: 1, }, },
                    y: { type: "linear", beginAtZero: true, suggestedMax: 50, ticks: { precision: 1, }, },
                },
            },
            startTime: 0,
            packetsPerSecond: [],
            timeTasks: [],
            sockets: []
        };
    },

    computed: {
        chartData() {
            return {
                datasets: [
                    {
                        label: this.$i18n.t("dashboard.benchmarkHistogram.PPS"),
                        backgroundColor: colors.green.base,
                        data: this.packetsPerSecond,
                    },
                    // {
                    //   label: this.$i18n.t("dashboard.benchmarkHistogram.RTT"),
                    //   backgroundColor: colors.red.base,
                    //   data: this.roundTripTime,
                    // },
                ],
            };
        },
    },

    created() { },

    beforeDestroy() { clearInterval(this.interval); },

    methods: {
        updateChartBounds() {
            const now = new Date();
            this.chartOptions.scales.x.max = (now - this.startTime) / 1000;
        },

        startBenchmark() {
            this.updateChartBounds();
            this.interval = setInterval(this.updateChartBounds, 2000);
            this.packetsPerSecond = [];
            this.endBenchmark();

            const ENDPOINT = "ws://localhost:3000";
            const ENDPOINT_WPS = "ws://localhost:8080";
            const MAX_CLIENTS = 1;
            const EMIT_PER_SECOND = 50;
            const CLIENT_CREATION_INTERVAL_IN_MS = 10;

            var clientCount = 0;
            var lastReport = new Date().getTime();
            var packetsSinceLastReport = 0;
            var totalEmittedPackets = 0;
            var totalReceivedPackets = 0;
            var totalRoundTripTime = 0;
            var maxRoundTripTime = 0;

            const createClient = () => {
                const useWps = true;
                const transports = ["websocket"];

                const socket = useWps ? io(ENDPOINT_WPS + "/benchmark", { transports: transports, path: "/clients/engineio/hubs/eio_hub" })
                    : io(ENDPOINT, { transports: transports });

                this.sockets.push(socket);

                this.timeTasks.push(setInterval(() => {
                    socket.emit("client to server event", Date.now());
                    totalEmittedPackets++;
                }, 1000 / EMIT_PER_SECOND));

                socket.on("server to client event", (timestamp) => {
                    let costTime = Date.now() - Number(timestamp);
                    maxRoundTripTime = Math.max(maxRoundTripTime, costTime);
                    totalRoundTripTime += Date.now() - Number(timestamp);
                    totalReceivedPackets++;
                    packetsSinceLastReport++;
                });

                socket.on("disconnect", (reason) => {
                    console.log(`disconnect due to ${reason}`);
                });

                if (++clientCount < MAX_CLIENTS) {
                    this.timeTasks.push(setTimeout(createClient, CLIENT_CREATION_INTERVAL_IN_MS));
                }
            };

            this.timeTasks.push(createClient());
            this.startTime = new Date().getTime();

            const report = () => {
                const now = new Date().getTime();
                const durationSinceLastReport = (now - lastReport) / 1000;
                const packetsPerSecond = (packetsSinceLastReport / durationSinceLastReport).toFixed(2);

                console.log(
                    `${clientCount} Clients, |PKTS RECV| = ${totalReceivedPackets} , PPS = ${packetsPerSecond} ; AVG RTT = ${totalRoundTripTime / totalReceivedPackets} ms ; MAX RTT = ${maxRoundTripTime} ms, |Emitted| = ${totalEmittedPackets}`
                );

                this.packetsPerSecond.push({ x: (new Date().getTime() - this.startTime) / 1000, y: parseFloat(packetsPerSecond) });

                packetsSinceLastReport = 0;
                lastReport = now;
            };

            this.timeTasks.push(setInterval(() => { report(); }, 1000));
        },

        endBenchmark() {
            this.timeTasks.forEach(task => {
                clearInterval(task);
            });
            this.sockets.forEach(socket => {
                socket.close();
            });
            this.sockets = [];
        }
    },
};
</script>
  