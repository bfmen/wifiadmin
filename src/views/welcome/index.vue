<script setup lang="ts">
import { message } from "@/utils/message";
import Pie from "./components/Pie.vue";
import Line from "./components/Line.vue";
import TypeIt from "@/components/ReTypeit";
import { useWindowSize } from "@vueuse/core";
import { ref, watchEffect } from "vue";
import { useRouter } from "vue-router";
//下面是监听全局变量
import { useStore } from 'vuex';
import { setnet, postapi } from "@/api/user";
import { formatDataSize, bytesToKB, formatBytes, formatTimestamp } from "./components/fun";
// import { tr } from "element-plus/es/locale";

defineOptions({
  name: "Welcome"
});

const router = useRouter();
const loading = ref<boolean>(true);
const { version } = __APP_INFO__.pkg;
const { height } = useWindowSize();

const store = useStore();

const netname = ref('');
let newinfo = ref({});
let allinfo = ref({});
let all_list = ref([]);
let run_time = ref();
let qr_wifi = ref('./pic/qrcode_ssid_wifikey.png?_=' + Date.now());//wifi二维码
//折线图变量
let up = ref([]);
let down = ref([]);
let time = ref([]);
//流量统计饼图
let unit = ref();//方式
let size = ref();//套餐值
let percent = ref();//到达百分比提醒
let Enable = ref();//是否启用
let Used = ref();//已使用
let Used_time = ref();//使用时间
let switch_wifi = ref(false);//数据启动/停止
let wifi_switch = ref(false);//WIFI启动/停止
//实时上传下载
let volume_up = ref(0);
let volume_dwom = ref(0);
let load_switch = ref(false);//默认加载中
let switch_load = ref(false);//默认加载中
// 监听 signal 变量的变化
watchEffect(() => {
  loading.value = false;//加载完成
  newinfo.value = store.state.NetWork;
  allinfo.value = store.state.NetInfo;
  run_time.value = formatTimestamp(store.state.NetWork.realtime_time);
  //设置漫游状态
  volume_up.value = store.state.NetWork.realtime_tx_thrpt;
  volume_dwom.value = store.state.NetWork.realtime_rx_thrpt;
  switch_wifi.value = store.state.NetWork.ppp_status == 'ppp_connected' ? true : false;
  if (load_switch.value == true && switch_wifi.value == true) load_switch.value = false;
  wifi_switch.value = store.state.NetWork.wifi_cur_state == '1' ? true : false;
  if (switch_load.value == true && wifi_switch.value == true) switch_load.value = false;
  //流量统计变量
  unit.value = store.state.NetWork.data_volume_limit_unit;
  Enable.value = store.state.NetWork.data_volume_limit_switch;
  percent.value = store.state.NetWork.data_volume_alert_percent;
  if (unit.value == 'data') {
    size.value = formatDataSize(store.state.NetWork.data_volume_limit_size);
  } else {
    size.value = store.state.NetWork.data_volume_limit_size;
    // console.log(unit.value, size.value);
  }
  Used_time.value = store.state.NetWork.monthly_time;
  Used.value = parseInt(store.state.NetWork.monthly_tx_bytes) + parseInt(store.state.NetWork.monthly_rx_bytes);
  //折线图变量
  time.value.push(new Date().getSeconds());
  if (time.value.length > 30) {
    time.value.shift();
  }
  up.value.push(bytesToKB(volume_up.value));
  if (up.value.length > 30) {
    up.value.shift();
  }
  down.value.push(bytesToKB(volume_dwom.value));
  if (down.value.length > 30) {
    down.value.shift();
  }
  //更多信息
  all_list.value = [
    {
      name: '网络类型',
      val: store.state.NetInfo.sub_network_type
    },
    {
      name: 'ICCID',
      val: store.state.NetInfo.ziccid
    },
    {
      name: '网络名称(SSID)',
      val: store.state.NetWork.SSID1 != '' ? store.state.NetWork.SSID1 : store.state.NetInfo.m_SSID
    },
    {
      name: '最大连接数',
      val: store.state.NetInfo.MAX_Access_num
    },
    {
      name: 'Wi-Fi 覆盖范围远距离',
      val: store.state.NetInfo.wifi_coverage == 'long_mode' ? '远距离 Wi-Fi 范围' : store.state.NetInfo.wifi_coverage == 'medium_mode' ? '中距离 Wi-Fi 范围' : '近距离 Wi-Fi 范围 - 最佳电池续航'
    },
    {
      name: 'IP 地址',
      val: store.state.NetInfo.lan_ipaddr
    },
    {
      name: 'WAN IP 地址',
      val: store.state.NetInfo.wan_ipaddr
    },
    {
      name: 'IPV6 地址',
      val: store.state.NetInfo.ipv6_wan_ipaddr
    },
    {
      name: '软件版本',
      val: store.state.NetInfo.cr_version
    },
    {
      name: '硬件版本',
      val: store.state.NetInfo.hw_version
    },
    {
      name: 'rssi',
      val: store.state.NetInfo.detail_cell_rssi
    },
    {
      name: 'cellid',
      val: store.state.NetInfo.detail_cell_id
    }
  ]
  //网络运营商获取
  const str = store.state.NetWork.network_provider;
  if (store.state.NetWork.modem_main_state == 'modem_sim_undetected') {
    netname.value = 'SIM卡无效';
  } else {
    // 将字符串转为小写并去掉空格
    const processedStr = str.toLowerCase().replaceAll(" ", "");
    // 使用正则表达式匹配 chinamobile、chinatelecom、chinaunicom，并进行相应替换
    netname.value = processedStr.replace(/chinamobile|chinatelecom|chinaunicom/g, (match) => {
      switch (match) {
        case 'chinamobile':
          return '中国移动';
        case 'chinatelecom':
          return '中国电信';
        case 'chinaunicom':
          return '中国联通';
        default:
          return str != '' ? str : '服务受限';
      }
    });
  }
});

function upwifi(e) {
  load_switch.value = e;
  setnet(e == true ? 'CONNECT_NETWORK' : 'DISCONNECT_NETWORK').then(res => {
    if (res.result == "success") {
      message("成功发送请求", { type: "success" });
    } else {
      message("操作失败", { type: "error" });
      load_switch.value = false;
    }
  }).catch(error => {
    message("请求失败", { type: "error" });
    load_switch.value = false;
  });
}


function setwifi(e) {
  switch_load.value = e;
  postapi({ goformId: "SET_WIFI_INFO", wifiEnabled: e == true ? '1' : '0' }).then(res => {
    if (res.result == "success") {
      message("成功发送请求", { type: "success" });
    } else {
      message("操作失败", { type: "error" });
      switch_load.value = false;
    }
  }).catch(error => {
    message("请求失败", { type: "error" });
    switch_load.value = false;
  });
}

function gospeedset() {
  router.push({ name: 'QuickSettings' })
}
function GoDataPlan() {
  router.push({ name: 'DataPlan' })
}

function gotowifilist() {
  router.push({ name: 'connect' });
}
</script>

<template>
  <div>
    <el-row :gutter="24">
      <el-col :xs="24" :sm="24" :md="12" :lg="12" :xl="12" class="mb-[18px]" v-motion :initial="{ opacity: 0, y: 100 }"
        :enter="{ opacity: 1, y: 0, transition: { delay: 200 } }">
        <el-card shadow="never" :style="{ height: `calc(${height}px - 35vh - 250px)` }">
          <template #header>
            <div class="card-header">
              <div class="type-it-container">
                <TypeIt :className="'type-it2'" :values="['设备状态信息']" :cursor="false" :speed="60" />
              </div>
              <div class="button-container">
                <el-popover placement="bottom" :width="400">
                  <template #reference>
                    <el-button plain>更多信息</el-button>
                  </template>
                  <el-table :data="all_list">
                    <el-table-column property="name" label="项目" />
                    <el-table-column property="val" label="值" />
                  </el-table>
                </el-popover>
              </div>
            </div>
          </template>
          <el-scrollbar :height="`calc(${height}px - 35vh - 340px)`">
            <el-descriptions direction="vertical" :column="4" border size="default">
              <el-descriptions-item label="IMEI" :span="2">{{ allinfo.imei }}</el-descriptions-item>
              <el-descriptions-item label="ICCID" :span="2">{{ store.state.NetInfo.ziccid }}</el-descriptions-item>
                <el-descriptions-item label="IMSI" :span="2">{{ allinfo.sim_imsi }}</el-descriptions-item>
              <el-descriptions-item label="信号强度" :min-width="80">{{ allinfo.rssi }} dBm</el-descriptions-item>
              <el-descriptions-item label="运营商">{{ netname }}</el-descriptions-item>
              <el-descriptions-item label="网络名称" :min-width="120" :span="2">{{ newinfo.SSID1 != '' ?
                newinfo.SSID1 : allinfo.m_SSID
              }}</el-descriptions-item>
              <!-- <el-descriptions-item label="IP">{{ allinfo.lan_ipaddr }}</el-descriptions-item> -->
              <!-- <el-descriptions-item label="RSRQ">{{ allinfo.detail_cell_rsrq }}</el-descriptions-item>
              <el-descriptions-item label="SINR">{{ allinfo.detail_cell_sinr }}</el-descriptions-item>
              <el-descriptions-item label="PCI">{{ allinfo.detail_cell_pci }}</el-descriptions-item> -->
            </el-descriptions>
          </el-scrollbar>
        </el-card>
      </el-col>

      <el-col :xs="24" :sm="24" :md="12" :lg="12" :xl="12" class="mb-[18px]" v-motion :initial="{ opacity: 0, y: 100 }"
        :enter="{ opacity: 1, y: 0, transition: { delay: 200 } }">
        <el-card shadow="never" :style="{ height: `calc(${height}px - 35vh - 250px)` }">
          <template #header>
            <div class="card-header">
              <div class="type-it-container">
                <TypeIt :className="'type-it1'" :values="[`设备实时信息`]" :cursor="false" :speed="120" />
              </div>
              <div class="button-container">
                <el-button plain @click="gospeedset">快速设置</el-button>
              </div>
            </div>
          </template>
          <el-scrollbar :height="`calc(${height}px - 35vh - 340px)`" v-loading="loading || load_switch || switch_load">
            <el-descriptions direction="vertical" :column="3" border size="default">
              <el-descriptions-item label="WIFI状态" :align="'center'">
                <el-switch size="small" v-model="wifi_switch" @change="setwifi" />
              </el-descriptions-item>
              <el-descriptions-item label="数据漫游状态" :align="'center'">
                <el-switch size="small" v-model="switch_wifi" @change="upwifi" />
              </el-descriptions-item>
              <el-descriptions-item label="网络类型" :align="'center'">
                <el-text>{{ store.state.NetInfo.sub_network_type }}</el-text>
              </el-descriptions-item>
              <el-descriptions-item label="漫游时间" :align="'center'">
                <el-text>{{ run_time }}</el-text>
              </el-descriptions-item>
              <el-descriptions-item label="实时上传">
                {{ formatBytes(volume_up.toString()) }}
              </el-descriptions-item>
              <el-descriptions-item label="实时下载">
                {{ formatBytes(volume_dwom.toString()) }}
              </el-descriptions-item>
            </el-descriptions>
          </el-scrollbar>
        </el-card>
      </el-col>

      <el-col :xs="24" :sm="24" :md="12" :lg="8" :xl="8" class="mb-[18px]" v-motion :initial="{ opacity: 0, y: 100 }"
        :enter="{ opacity: 1, y: 0, transition: { delay: 400 } }">
        <el-card shadow="never">
          <template #header>
            <div class="card-header">
              <div class="type-it-container">
                <TypeIt :className="'type-it4'" :values="['上传下载折线图(KB)']" :cursor="false" :speed="120" />
              </div>
            </div>
          </template>
          <Line :time="time" :up="up" :down="down" />
        </el-card>
      </el-col>

      <el-col :xs="24" :sm="24" :md="12" :lg="8" :xl="8" class="mb-[18px]" v-motion :initial="{ opacity: 0, y: 100 }"
        :enter="{ opacity: 1, y: 0, transition: { delay: 400 } }">
        <el-card shadow="never">
          <template #header>
            <div class="card-header">
              <div class="type-it-container">
                <TypeIt :className="'type-it3'" :values="['流量统计']" :cursor="false" :speed="120" />
                <el-tag v-show="unit === 'data' && Enable === '1'" class="ml-2">套餐：{{ size }}</el-tag>
                <el-tag v-show="unit === 'time' && Enable === '1'" class="ml-2" type="success">套餐：{{ size }}h</el-tag>
                <el-tag v-show="Enable === '0'" class="ml-2" type="danger">未设置套餐</el-tag>
              </div>
              <div class="button-container">
                <el-button plain @click="GoDataPlan">设置流量套餐</el-button>
              </div>
            </div>
          </template>
          <Pie :unit="unit" :size="size" :percent="percent" :Enable="Enable" :Used="Used.toString()"
            :Used_time="Used_time" />
        </el-card>
      </el-col>

      <el-col :xs="24" :sm="24" :md="12" :lg="8" :xl="8" class="mb-[18px]" v-motion :initial="{ opacity: 0, y: 100 }"
        :enter="{ opacity: 1, y: 0, transition: { delay: 400 } }">
        <el-card shadow="never">
          <template #header>
            <div class="card-header">
              <div class="type-it-container">
                <TypeIt :className="'type-it5'" :values="[`已接入设备`]" :cursor="false" :speed="120" />
              </div>
              <div class="button-container">
                <el-popover placement="left" :width="180" popper-style="padding: 5px;">
                  <template #reference>
                    <el-button plain>连接二维码</el-button>
                  </template>
                  <template #default>
                    <div class="image-container" style="display: flex; gap: 16px; flex-direction: column">
                      <!-- <el-avatar :size="180" :src="qr_wifi" style="margin-bottom: 8px" /> -->
                      <img id="qrcode_img" :src="qr_wifi">
                    </div>
                  </template>
                </el-popover>
              </div>
            </div>
          </template>
          <!-- <Bar /> -->
          <div class="connected_devices_container opposite_color"
            style="width: 100%; height: 35vh; display: flex; flex-direction: column; justify-content: space-between;">
            <div class="inner_container" style="display: flex; justify-content: center; align-items: center;">
              <div class="icon_container" style="display: flex; justify-content: center; align-items: center;">
                <IconifyIconOnline icon="circum:server" style="margin: 1vh;width: 15vh; height: 15vh" />
              </div>
            </div>
            <div class="display_container" style="text-align: center;" @click="gotowifilist">
              <div id="h_wire_device" style="display: inline-block;">
                <div style="display: flex; align-items: center;">
                  <IconifyIconOnline icon="fa6-solid:network-wired" style="margin: 5px; width: 8vh; height: 8vh;" />
                  <span style="font-size: 70px;">{{ store.state.LanList.length }}</span>
                  <sub style="font-size: 16px;top: 16px;">台</sub>
                </div>
              </div>
              <div id="h_wireless_device" style="display: inline-block;margin-left: 30px;">
                <div style="display: flex; align-items: center;">
                  <IconifyIconOnline icon="icon-park-outline:wifi" style="margin: 5px; width: 8vh; height: 8vh;" />
                  <span style="font-size: 70px;">{{ store.state.NetWork.sta_count }}</span>
                  <sub style="font-size: 16px;top: 16px;">台</sub>
                </div>
              </div>
            </div>
            <div class="bottom_container" style="display: flex; justify-content: center; align-items: flex-end;">
              <div class="descriptions_container" style="width: 100%;">
                <el-descriptions :column="2" border size="default">
                  <el-descriptions-item label="当前WEB版本" :align="'center'">
                    Y_V{{ version }}
                  </el-descriptions-item>
                </el-descriptions>
              </div>
            </div>
          </div>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<style lang="scss" scoped>

.main-content {
  margin: 20px 20px 0 !important;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.el-card__header-button {
  margin-left: auto;
}

.button-container {
  height: 0px;
  margin-top: -30px;
}

.connected_devices_container {
  float: left;
  width: 100%;
  height: 100%;
  margin-bottom: 20px;
  border-radius: 10px;
  background: -webkit-linear-gradient(left top, #70ebe6, #059ec4);
  background: -o-linear-gradient(bottom right, #70ebe6, #059ec4);
  background: -moz-linear-gradient(bottom right, #70ebe6, #059ec4);
  background: linear-gradient(to bottom right, #70ebe6, #059ec4);
}

.opposite_color {
  color: white;
}

.image-container {
  width: 100%;
  /* 设置容器的宽度为 100% */
}

.image-container img {
  width: 100%;
  /* 设置 img 元素的宽度为 100% */
  height: auto;
  /* 自动计算高度，保持图片比例 */
}

.dialog-footer button:first-child {
  margin-right: 10px;
}
</style>