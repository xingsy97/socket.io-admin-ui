<!-- <template>
    <div>
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
                    <Bar :chart-data="chartData" :chart-options="chartOptions1" style="width: 100%" :height="chartHeight" />
                </v-row>
            </v-card-text>

        </v-card>

        <v-card>

            <v-card-text>
                <v-row>
                    <Bar :chart-data="chartData2" :chart-options="chartOptions2" style="width: 100%" :height="chartHeight" />
                </v-row>
            </v-card-text>
        </v-card>
    </div>
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
            chartOptions1: {
                parsing: false,
                scales: {
                    x: { type: "linear", beginAtZero: true, suggestedMax: 50, ticks: { precision: 1, }, },
                    y: { type: "linear", beginAtZero: true, suggestedMax: 50, ticks: { precision: 1, }, },
                },
            },
            chartOptions2: {
                parsing: false,
                scales: {
                    x: { type: "linear", beginAtZero: true, suggestedMax: 50, ticks: { precision: 1, }, },
                    y: { type: "linear", beginAtZero: true, suggestedMax: 50, ticks: { precision: 0, }, },
                },
            },
            startTime: 0,
            lifetimeDatas: [],
            periodDatas: [],
            timeTasks: [],
            sockets: [],

        };
    },

    computed: {
        chartData() {
            return {
                datasets: [
                    {
                        label: "Received Packets Per Second",
                        backgroundColor: colors.green.base,
                        data: this.receivedPacketsPerSecond,
                    },
                    {
                      label: "Sent Packets per Second",
                      backgroundColor: colors.red.base,
                      data: this.sentPacketPerSecond,
                    },
                ],
            };
        },
        chartData2() {
            return {
                datasets: [
                    {
                        label: "Max Round Trip Time per Second",
                        backgroundColor: colors.blue.base,
                        data: this.periodMaxRoundTripTime,
                    },
                ],
            };
        },
        receivedPacketsPerSecond() {
                var a = [];
                for (var i = 1; i < this.lifetimeDatas.length; i++) {
                    a.push({ x: this.lifetimeDatas[i].x / 1000, y: this.lifetimeDatas[i].y["totalReceivedPackets"] / ((this.lifetimeDatas[i].x - this.lifetimeDatas[i - 1].x) / 1000) });
                }
                return a;
        },
        sentPacketPerSecond() {
            var a = [];
            for (var i = 1; i < this.lifetimeDatas.length; i++) {
                a.push({ x: this.lifetimeDatas[i].x / 1000, y: this.lifetimeDatas[i].y["totalEmittedPackets"] / ((this.lifetimeDatas[i].x - this.lifetimeDatas[i - 1].x) / 1000) });
            }
            return a;
        },
        periodMaxRoundTripTime() {
            var a = [];
            for (var i = 0; i < this.periodDatas.length; i++) {
                a.push({ x: this.periodDatas[i].x / 1000, y: this.periodDatas[i].y["maxRoundTripTime"] });
            }
            return a;
        },
    },

    created() { },

    beforeDestroy() { this.endBenchmark(); },

    methods: {
        updateChartBounds() {
            const now = new Date();
            this.chartOptions1.scales.x.max = (now - this.startTime) / 1000 + 1;
            this.chartOptions2.scales.x.max = (now - this.startTime) / 1000 + 1;
        },

        startBenchmark( ) {
            var MAX_CLIENTS = 1; var EMIT_PER_SECOND = 150; var CLIENT_CREATION_INTERVAL_IN_MS = 10;

            this.updateChartBounds();
            this.timeTasks.push(setInterval(this.updateChartBounds, 2000));
            this.endBenchmark();

            const ENDPOINT = "ws://localhost:3000";
            const ENDPOINT_WPS = "ws://localhost:8080";

            var lifetimeData = { 
                startTime : new Date().getTime(),
                clientCount: 0,
                totalReceivedPackets: 0,
                totalEmittedPackets: 0,
                totalRoundTripTime: 0,
            }

            var periodData = {
                startTime: new Date().getTime(),
                maxRoundTripTime: 0
            }

            const createClient = () => {
                const useWps = true;
                const transports = ["websocket"];

                const socket = useWps ? io(ENDPOINT_WPS + "/echoBenchmark", { transports: transports, path: "/clients/socketio/hubs/eio_hub" })
                                      : io(ENDPOINT  + "/echoBenchmark", { transports: transports });

                this.sockets.push(socket);

                this.timeTasks.push(setInterval(() => {
                    socket.emit("client to server event", Date.now());
                    lifetimeData.totalEmittedPackets++;
                }, 1000 / EMIT_PER_SECOND));

                socket.on("server to client event", (timestamp) => {
                    let costTime = Date.now() - Number(timestamp);
                    lifetimeData.totalRoundTripTime += costTime;
                    lifetimeData.totalReceivedPackets ++;
                    periodData.maxRoundTripTime = Math.max(periodData.maxRoundTripTime, costTime);
                });

                socket.on("disconnect", (reason) => {
                    console.log(`disconnect due to ${reason}`);
                });

                console.log("cc", lifetimeData.clientCount + 1, MAX_CLIENTS)
                if (lifetimeData.clientCount + 1 < MAX_CLIENTS) {
                    this.timeTasks.push(setTimeout(createClient, CLIENT_CREATION_INTERVAL_IN_MS));
                }
                lifetimeData.clientCount++;
            };

            createClient();
            this.startTime = new Date().getTime();

            const report = () => {
                if (this.periodDatas.length > 60) {
                    this.endBenchmark();
                    return ;
                }

                const now = new Date().getTime();
                periodData["duration"] = (now - periodData["startTime"] );
                periodData["sentPPS"] = (periodData["totalReceivedPackets"] / periodData["durationSinceLastReport"]).toFixed(2);

                console.log(`${lifetimeData.clientCount} Clients, ${JSON.stringify(periodData)}, ${JSON.stringify(lifetimeData)}`);

                this.lifetimeDatas.push(Object.assign({},  { x: (new Date().getTime() - this.startTime) , y: lifetimeData }));
                this.periodDatas.push(Object.assign({},  { x: (new Date().getTime() - this.startTime) , y: periodData }));

                periodData = {
                    startTime: new Date().getTime(),
                    maxRoundTripTime: 0
                }
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
   -->