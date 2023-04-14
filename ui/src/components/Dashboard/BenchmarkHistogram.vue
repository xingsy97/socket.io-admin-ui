<template>
    <v-container fluid>
        <v-row>
            <v-col cols="4" md="3" lg="3">
                <v-btn color="primary" class="white--text" @click="startBenchmark(serverUrl, '/benchmark', wsOnly, path, parser)"> Start Benchmark </v-btn> <br><br>
                <v-btn color="primary" class="white--text" @click="endBenchmark"> END Benchmark </v-btn>
                <br>Total Clients <v-text-field class="white--text" v-model="maxClients" />
                Client Send Per Second <v-text-field type="number" class="white--text"  v-model="emitPerSecond" />
                Client Creation Interval (ms)<v-text-field type="number" class="white--text" v-model="clientCreationIntervalInMs" />
                Total benchmark time (s)<v-text-field type="number" class="white--text" v-model="totalBenchmarkSeconds" />
            </v-col>
            
            <v-col cols="8"  >
                <v-chart class="chart" :option="option" autoresize />
            </v-col>
        </v-row>
    </v-container>
  </template>
  
  <script>
  import { mapState } from "vuex";
  import { use } from 'echarts/core';
  import { CanvasRenderer } from 'echarts/renderers';
  import { LineChart, BarChart } from 'echarts/charts';
  import {
    TitleComponent,
    TooltipComponent,
    LegendComponent,
    GridComponent,
  } from 'echarts/components';
  import VChart, { THEME_KEY } from 'vue-echarts';
  import { ref, defineComponent, reactive } from 'vue';
  import { io } from "socket.io-client";
  
  use([
    CanvasRenderer,
    LineChart,
    TitleComponent,
    TooltipComponent,
    LegendComponent,
    GridComponent,
    BarChart
  ]);
  
  export default defineComponent({
    name: 'BenchmarkHistogram',
    components: {
      VChart,
    },
    provide: {
      [THEME_KEY]: 'dark',
    },
    data() {
        return {
            startTime : 0,
            lifetimeDatas : [],
            periodDatas : [],
            timeTasks : [],
            sockets : [],

            maxClients: 1,
            emitPerSecond: 50,
            clientCreationIntervalInMs: 10,
            totalBenchmarkSeconds: 60,
        };
    },
    setup() {
      const chartData = reactive({
        xAxisData: [],
        seriesData: [],
        seriesData2: [],
        seriesData3: [],
      });
  
      const option = reactive({
        backgroundColor:'rgba(128, 128, 128, 0.1)',
        tooltip: { trigger: 'axis', axisPointer: { type: 'cross' }, },
        title: { text: 'Benchmark Chart', left: '10vh', top: "20vh"},
        grid: {x: 70, y: 100, x2: 70, y2: 50},
        xAxis: { type: 'value', name: "Time (s)", position: "center"},
        yAxis: [ { type: 'value', name: "Packets Count", position: 'left', axisLabel: { formatter: '{value}' }},
                 { type: 'value', name: "Round Trip Time (ms)", position: 'right', axisLabel: { formatter: '{value} ms' }}],
        legend: {
            orient: 'horizontal', top: "10%", left: "30%",
            data: ['receivedPacketPerSecond', 'sentPacketPerSecond', 'maxRoundTripTimePerSecond']
        },
        series: [
          {
            name: "receivedPacketPerSecond",
            data: chartData.seriesData,
            type: 'bar',
            color: "green",
            yAxisIndex:0,
            showAllSymbol: true,
          },
          {
            name: "sentPacketPerSecond",
            data: chartData.seriesData2,
            type: 'bar',
            color: "red",
            yAxisIndex:0,
            showAllSymbol: true,
          },
          {
            name: "maxRoundTripTimePerSecond",
            data: chartData.seriesData3,
            type: 'line',
            yAxisIndex:1,
            showAllSymbol: true,
          },
        ],
      });
  
      // 在需要更新数据时，修改chartData.xAxisData和chartData.seriesData

      return { option, chartData };
    },

    computed: {
        ...mapState({
            serverUrl: (state) => state.connection.serverUrl,
            wsOnly: (state) => state.connection.wsOnly,
            path: (state) => state.connection.path,
            namespace: (state) => state.connection.namespace,
            parser: (state) => state.connection.parser,
        }),
    },

    methods: {
        startBenchmark(serverUrl, namespace, wsOnly, path, parser) {
            console.log(serverUrl);
            var MAX_CLIENTS = this.maxClients; var EMIT_PER_SECOND = this.emitPerSecond; var CLIENT_CREATION_INTERVAL_IN_MS = this.clientCreationIntervalInMs;
            console.log("MAX_CLIENTS=", MAX_CLIENTS, ",EMIT_PER_SECOND=", EMIT_PER_SECOND, ",CLIENT_CREATION_INTERVAL_IN_MS=", CLIENT_CREATION_INTERVAL_IN_MS);

            this.endBenchmark();
            // this.chartData.seriesData = [];
            // this.chartData.seriesData2 = [];
            // this.chartData.seriesData3 = [];

            var lifetimeData = { 
                startTime : 0,
                clientCount: 0,
                totalReceivedPackets: 0,
                totalEmittedPackets: 0,
                totalRoundTripTime: 0,
            }

            var periodData = {
                startTime: 0,
                maxRoundTripTime: 0
            }

            const updateChartData = (time) => {
                console.log(time);
                var i = this.lifetimeDatas.length - 1;
                
                if (i > 0) {
                    let currentData = this.lifetimeDatas[i];
                    let previousData = this.lifetimeDatas[i - 1];
                    let duration = (currentData.x - previousData.x) / 1000.0;

                    let v1 = (currentData.y["totalReceivedPackets"] - previousData.y["totalReceivedPackets"])  / duration;
                    let v2 = (currentData.y["totalEmittedPackets"] - previousData.y["totalEmittedPackets"]) / duration;

                    console.log("time=", time, ",duration=", duration, ",AVG RECEIVE=", v1, ",AVG SEND=", v2);

                    this.chartData.seriesData.push( [time, v1.toFixed(1) ]) // / (EMIT_PER_SECOND *  duration) ] );
                    this.chartData.seriesData2.push( [time, v2.toFixed(1) ]) /// (EMIT_PER_SECOND * duration)] );

                    this.chartData.seriesData3.push( [time, this.periodDatas[i].y["maxRoundTripTime"] == 0 ? null : this.periodDatas[i].y["maxRoundTripTime"]  ] );
                }  
            };

            const createClient = () => {
                const useWps = true;
                const transports = ["websocket"];

                const socket = useWps ? io(serverUrl + namespace, { transports: transports, path: path })
                                      : io(serverUrl  + namespace, { transports: transports });

                this.sockets.push(socket);

                socket.on("connect", () => {
                    console.log("client connected");
                    lifetimeData.clientCount++;
                    this.timeTasks.push(setInterval(() => {
                        for (var _ = 0; _ < EMIT_PER_SECOND; _++) {
                            socket.emit("client to server event", new Date().getTime());
                            lifetimeData.totalEmittedPackets++;
                        }
                    }, 1000));
                });

                socket.on("server to client event", (timestamp) => {
                    let costTime = new Date().getTime() - Number(timestamp);
                    lifetimeData.totalRoundTripTime += costTime;
                    lifetimeData.totalReceivedPackets ++;
                    periodData.maxRoundTripTime = Math.max(periodData.maxRoundTripTime, costTime);
                });

                socket.on("disconnect", (reason) => {
                    console.log(`disconnect due to ${reason}`);
                });

                if (lifetimeData.clientCount + 1 < MAX_CLIENTS) {
                    this.timeTasks.push(setTimeout(createClient, CLIENT_CREATION_INTERVAL_IN_MS));
                }
                
            };

            createClient();
            this.startTime = new Date().getTime();
            periodData.startTime = this.startTime;
            lifetimeData.startTime = this.startTime;
            this.periodDatas.push({x: 0, y: Object.assign({}, periodData) });
            this.lifetimeDatas.push({x: 0, y: Object.assign({}, lifetimeData) });

            const report = () => {
                if (this.periodDatas.length > this.totalBenchmarkSeconds) {
                    this.endBenchmark();
                    return ;
                }

                const now = new Date().getTime();
                periodData["duration"] = (now - periodData["startTime"] );

                console.log(`${lifetimeData.clientCount} Clients, ${JSON.stringify(periodData)}, ${JSON.stringify(lifetimeData)}`);

                this.lifetimeDatas.push({ x: (new Date().getTime() - this.startTime) , y: Object.assign({}, lifetimeData) });
                this.periodDatas.push({ x: (new Date().getTime() - this.startTime) , y: Object.assign({}, periodData) });

                // lifetimeData = Object.assign({}, lifetimeData);
                // periodData = Object.assign({}, periodData);

                periodData = {
                    startTime: new Date().getTime(),
                    maxRoundTripTime: 0
                }

                updateChartData((now - this.startTime) / 1000);
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
            this.lifetimeDatas = [];
            this.periodDatas = [];
        }
    },
  });
  </script>
  
  <style scoped>
  .chart {
    height: 70vh;
    width: 180vh;
  }
  </style>