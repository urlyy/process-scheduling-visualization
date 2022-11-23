<script setup>
import 'element-plus/es/components/message/style/css'

import { ref, watch } from 'vue'

class PCB {
  //进程名
  name;
  //进程ID
  ID;
  //优先级
  priority;
  //到达时间
  arriveTime;
  //需运行时间
  serviceTime;
  //已用CPU时间
  useCPUTime;
  //进程状态
  status;
  //阻塞原因
  blockReason;
  //等待时间
  waitTime;
  //高亮(界面效果)
  highlight;
  constructor(name, ID, serviceTime, priority, arriveTime) {
    this.name = name;
    this.ID = ID;
    this.priority = priority;
    this.arriveTime = arriveTime;
    this.serviceTime = serviceTime;
    if (this.arriveTime == context.value.passTime) {
      this.status = "ready";
    } else {
      this.status = "new";
    }
    this.useCPUTime = 0;
    this.blockReason = "";
    this.waitTime = 0;
    this.highlight = false;
  }
}

const selectedSchedulingAlgorithIndex = ref("");

const initData = [
  {
    name: "空",
    data: [],
  },
  {
    name: "数据一(可用来测试先来先服务)",
    data: [
      { name: "p-0", ID: 0, serviceTime: 5, priority: 0, arriveTime: 0 },
      { name: "p-1", ID: 1, serviceTime: 4, priority: 0, arriveTime: 0 },
      { name: "p-2", ID: 2, serviceTime: 3, priority: 0, arriveTime: 0 },
      { name: "p-3", ID: 3, serviceTime: 2, priority: 0, arriveTime: 0 }
    ],
  },
  {
    name: "数据二(可用来测试时间片轮转)",
    data: [
      { name: "p-0", ID: 0, serviceTime: 6, priority: 0, arriveTime: 0 },
      { name: "p-1", ID: 1, serviceTime: 7, priority: 0, arriveTime: 0 },
      { name: "p-2", ID: 2, serviceTime: 8, priority: 0, arriveTime: 0 },
      { name: "p-3", ID: 3, serviceTime: 9, priority: 0, arriveTime: 0 }
    ],
  },
  {
    name: "数据三(可用来测试优先级)",
    data: [
      { name: "p-0", ID: 0, serviceTime: 5, priority: 0, arriveTime: 0 },
      { name: "p-1", ID: 1, serviceTime: 5, priority: 2, arriveTime: 0 },
      { name: "p-2", ID: 2, serviceTime: 5, priority: 4, arriveTime: 0 },
      { name: "p-3", ID: 3, serviceTime: 5, priority: 6, arriveTime: 0 }
    ],
  },
  {
    name: "数据四(可用来测试高响应比优先)",
    data: [
      { name: "p-0", ID: 0, serviceTime: 8, priority: 0, arriveTime: 0 },
      { name: "p-1", ID: 1, serviceTime: 7, priority: 0, arriveTime: 0 },
      { name: "p-2", ID: 2, serviceTime: 3, priority: 0, arriveTime: 0 },
      { name: "p-3", ID: 3, serviceTime: 2, priority: 0, arriveTime: 0 }
    ],
  },
  {
    name: "数据五(可用来测试多级反馈队列)",
    data: [
      { name: "p-0", ID: 0, serviceTime: 10, priority: 0, arriveTime: 0 },
      { name: "p-1", ID: 1, serviceTime: 16, priority: 0, arriveTime: 0 },
      { name: "p-2", ID: 2, serviceTime: 5, priority: 0, arriveTime: 0 },
      { name: "p-3", ID: 3, serviceTime: 8, priority: 0, arriveTime: 0 }
    ],
  },
]


//先来先服务算法对象
const FCFS = {
  schedule: function () {
    if (context.value.runQueue.length != 0) { return; }
    if (context.value.readyQueues[0].length == 0) { return; }
    let item = context.value.readyQueues[0].shift();
    item.status = 'running';
    item.highlight = false;
    context.value.runQueue.push(item);
  }
}

//时间片，默认为1
const RR_timeSlice = ref(1);
//时间片轮转算法对象
const RR = {
  //当前已用时间片
  tmpTimeSlice: 0,
  schedule: function () {
    RR.tmpTimeSlice++;
    if (RR.tmpTimeSlice == 1 || context.value.runQueue.length == 0) {
      RR.tmpTimeSlice = 1;
      //放出
      if (context.value.runQueue.length != 0) {
        let item = context.value.runQueue.shift();
        context.value.readyQueues[0].push(item);
        item.status = 'ready';
        item.highlight = false;
      }
      //放入
      if (context.value.readyQueues[0].length != 0) {
        let item = context.value.readyQueues[0].shift();
        context.value.runQueue.push(item);
        item.status = 'running';
        item.highlight = false;
      }
    }
    if (RR.tmpTimeSlice == RR_timeSlice.value) {
      RR.tmpTimeSlice = 0;
    }
  }
}
//优先级调度算法对象
const PSA = {
  schedule: function () {
    if (context.value.runQueue.length != 0) { return; }
    if (context.value.readyQueues[0].length == 0) { return; }
    let flag = 0;
    for (let i = 1; i < context.value.readyQueues[0].length; i++) {
      let a = context.value.readyQueues[0][flag];
      let b = context.value.readyQueues[0][i];
      if (a.priority < b.priority) { flag = i; }
    }
    let item = context.value.readyQueues[0][flag];
    context.value.readyQueues[0].splice(flag, 1);
    item.status = 'running';
    item.highlight = false;
    context.value.runQueue.push(item);
  }
}
//高响应比优先调度算法对象
const HRRN = {
  schedule: function () {
    if (context.value.runQueue.length != 0) { return; }
    if (context.value.readyQueues[0].length == 0) { return; }
    let flag = 0;
    for (let i = 1; i < context.value.readyQueues[0].length; i++) {
      let item1 = context.value.readyQueues[0][flag];
      let item2 = context.value.readyQueues[0][i];
      let rr1 = 1 + (context.value.passTime - 1 - item1.arriveTime) / item1.serviceTime;
      let rr2 = 1 + (context.value.passTime - 1 - item1.arriveTime) / item2.serviceTime;
      if (rr1 < rr2) {
        flag = i;
      }
    }
    let item = context.value.readyQueues[0][flag];
    item.status = 'running';
    item.highlight = false;
    context.value.readyQueues[0].splice(flag, 1);
    context.value.runQueue.push(item);
  },
}



//最大队列级数，至少为1
const MFQ_maxLevel = ref(1);
//各级队列的时间片大小，默认第一级时间片为2
const MFQ_timeSlices = ref([2]);
const handleMFQSlicesChange = (value) => {
  MFQ_timeSlices.value = []
  for (let i = 1; i <= value; i++) {
    MFQ_timeSlices.value.push(i * 2);
  }
}
//多级反馈队列算法对象
const MFQ = {
  //当前已使用的时间片
  tmpTimeSlice: 0,
  //当前运行的进程之前在的就绪队列级数
  curLevel: 0,
  schedule: function () {
    MFQ.tmpTimeSlice++;
    if (MFQ.tmpTimeSlice == 1 || context.value.runQueue.length == 0) {
      MFQ.tmpTimeSlice = 1;
      //放出
      if (context.value.runQueue.length != 0) {
        let item = context.value.runQueue.shift();
        let targetLevel = MFQ.curLevel + 1 <= context.value.readyQueues.length - 1 ? MFQ.curLevel + 1 : context.value.readyQueues.length - 1;
        context.value.readyQueues[targetLevel].push(item);
        item.status = 'ready';
        item.highlight = false;
      }
      //找到第一个就放入
      for (let i = 0; i < context.value.readyQueues.length; i++) {
        if (context.value.readyQueues[i].length > 0) {
          let item = context.value.readyQueues[i].shift();
          MFQ.curLevel = i;
          context.value.runQueue.push(item);
          item.status = 'running';
          item.highlight = false;
          break;
        }
      }
    }
    if (MFQ.tmpTimeSlice == MFQ_timeSlices.value[MFQ.curLevel]) {
      MFQ.tmpTimeSlice = 0;
    }
  }
}
const context = ref({
  newQueue: [],
  readyQueues: [],
  runQueue: [],
  blockQueue: [],
  finishQueue: [],
  //0未开始,1运行中,2暂停
  globalStatus: 0,
  passTime: 0,
  globalTaskTimer: null,
  schedulingAlgorithm: null,
  processNum: 0,
  clear: function () {
    context.value.newQueue = [];
    context.value.readyQueues = [];
    context.value.runQueue = [];
    context.value.blockQueue = [];
    context.value.finishQueue = [];
    context.value.passTime = 0;
    context.value.processNum = 0;
    // context.value.globalTaskTimer = null;
    context.value.schedulingAlgorithm = null;
    selectedSchedulingAlgorithIndex.value = 0;
    RR_timeSlice.value = 1;
    MFQ_maxLevel.value = 1;
    MFQ_timeSlices.value = [2];
    MFQ.tmpTimeSlice = 0;
    MFQ.curLevel = 0;
    selectedInitProcessDataIndex.value = 0;
    selectedBlockProcessIndex.value = [];
    dialogFormVisible.value = false;
  }
})

//每秒改变当前界面内容
const changeState = () => {
  //全局状态
  context.value.passTime++;
  //更新队列时间
  context.value.newQueue.forEach(item => { item.waitTime++; })

  context.value.readyQueues.forEach(q => {
    q.forEach(item => { item.waitTime++; })
  })

  context.value.runQueue.forEach(item => {
    item.useCPUTime++;
  })
  context.value.blockQueue.forEach(item => { item.waitTime++; })
  //再更新状态
  //run->finish
  if (context.value.runQueue.length > 0) {
    let runItem = context.value.runQueue[0];
    if (runItem.useCPUTime == runItem.serviceTime) {
      context.value.runQueue.shift();
      runItem.status = "finish";
      context.value.finishQueue.push(runItem);
      runItem.highlight = false;
      // runItem.waitTime--;
    }
  }
  //ready->run
  context.value.schedulingAlgorithm.schedule();
  //new->ready
  for (let i = context.value.newQueue.length - 1; i >= 0; i--) {
    let item = context.value.newQueue[i];
    if (context.value.passTime == item.arriveTime) {
      context.value.newQueue.splice(i, 1);
      item.status = "ready";
      context.value.readyQueues[0].push(item);
      item.highlight = false;
    }
  }
}
//阻塞当前运行进程
const doBlock = () => {
  if (context.value.runQueue.length != 0) {
    let item = context.value.runQueue.shift();
    item.status = "block";
    context.value.blockQueue.push(item);
  }
}
//唤醒多个进程
const doWake = () => {
  selectedBlockProcessIndex.value.sort((a, b) => {
    return b - a;
  })
  selectedBlockProcessIndex.value.forEach(index => {
    let item = context.value.blockQueue[index];
    context.value.blockQueue.splice(index, 1);
    context.value.readyQueues[0].push(item);
  })
  selectedBlockProcessIndex.value = []
}
const doTimingGlobalTask = () => {
  context.value.globalTaskTimer = setTimeout(() => {
    if (context.value.globalStatus != 1) { return; }
    changeState();
    doTimingGlobalTask();
  }, 1000)
}

const selectedInitProcessDataIndex = ref(0);

const start = () => {
  context.value.schedulingAlgorithm = schedulingAlgorithms[selectedSchedulingAlgorithIndex.value].sa;
  if (selectedSchedulingAlgorithIndex.value != 4) {
    context.value.readyQueues.push([]);
  } else {
    context.value.readyQueues = [];
    for (let i = 1; i <= MFQ_maxLevel.value; i++) {
      context.value.readyQueues.push([]);
    }
  }
  context.value.processNum = initData[selectedInitProcessDataIndex.value].data.length;
  initData[selectedInitProcessDataIndex.value].data.forEach(item => {
    let p = new PCB(item.name, item.ID, item.serviceTime, item.priority, item.arriveTime)
    if (p.arriveTime == 0) {
      context.value.readyQueues[0].push(p);
    } else {
      context.value.newQueue.push(p);
    }
  })
  context.value.schedulingAlgorithm.schedule();
  if (context.value.globalStatus == 1) {
    doTimingGlobalTask();
  } else {
    // next();
  }
}




const pause = () => {
  context.value.globalStatus = 2;
}
const continuee = () => {
  context.value.globalStatus = 1;
  context.value.globalTaskTimer = setTimeout(() => {
    changeState();
    doTimingGlobalTask();
  }, 1000)
}
const end = () => {
  context.value.globalStatus = 0;
  context.value.clear();
}
const next = () => {
  changeState();
}

const selectedBlockProcessIndex = ref([]);

const schedulingAlgorithms = [
  {
    name: "先来先服务",
    sa: FCFS,
  },
  {
    name: "时间片轮转",
    sa: RR,
  },
  {
    name: "优先级",
    sa: PSA,

  }, {
    name: "高响应比优先",
    sa: HRRN,
  },
  {
    name: "多级反馈队列",
    sa: MFQ,
  }
]
const changeSchedulingAlgorithm = (val) => {
  context.value.schedulingAlgorithm = schedulingAlgorithms[selectedSchedulingAlgorithIndex.value].sa;
}
const addProcessForm = ref({
  ID: context.value.processNum,
  name: "p-" + context.value.processNum,
  serviceTime: 0,
  arriveTime: 0,
  priority: 0,
})
const formRef = ref(null);
const addProcess = (formEl) => {
  formEl.validate((valid) => {
    if (valid) {
      let p = new PCB(addProcessForm.value.name, addProcessForm.value.ID, addProcessForm.value.serviceTime, addProcessForm.value.priority, addProcessForm.value.arriveTime + context.value.passTime);
      context.value.processNum += 1;
      if (p.arriveTime == 0) {
        context.value.readyQueues[0].push(p);
      } else {
        context.value.newQueue.push(p);
      }
      addProcessForm.value.serviceTime = 0;
      addProcessForm.value.arriveTime = 0;
      addProcessForm.value.priority = 0;
      resetAddProcessForm();
    } else {
      return false;
    }
  })
}
const resetAddProcessForm = () => {
  addProcessForm.value.serviceTime = 0;
  addProcessForm.value.arriveTime = 0;
  addProcessForm.value.priority = 0;

}
//深层监听
watch(() => context.value.processNum, (newNum, oldNum) => {
  addProcessForm.value.ID = newNum;
  addProcessForm.value.name = 'p-' + newNum;
});
const dialogFormVisible = ref(false);

</script>

<template>
  <div id="page">
    <div style="font-size: 24px;font-weight: 500;display: flex;justify-content: center;margin-bottom: 5px;">
      <span v-if="context.globalStatus == 0">进程状态转换与进程调度</span>
      <span v-else>
        <span v-if="context.globalStatus != 0">{{ schedulingAlgorithms[selectedSchedulingAlgorithIndex].name
        }}</span>
        <span v-show="context.globalStatus != 0" style="margin-left: 10px;">{{ context.passTime }}秒</span>
      </span>
    </div>
    <!-- 头部 -->
    <div style="font-size: 24px;font-weight: 500;display: flex;justify-content: center;">
      <!-- <el-select v-show="context.globalStatus != 0" style="font-size: 30px;" v-model="selectedSchedulingAlgorithIndex"
        placeholder="选择调度算法" @change="changeSchedulingAlgorithm">
        <el-option v-for="(item, index) in schedulingAlgorithms"
          :disabled="(selectedSchedulingAlgorithIndex == 4 && index != 4) || (selectedSchedulingAlgorithIndex != 4 && index == 4)"
          :key="item.name" :label="item.name" :value="index" />
      </el-select> -->

      <span style="margin-left:10px;">
        <el-button type="primary" style="font-size: 20px;"
          @click="selectedSchedulingAlgorithIndex = 0; dialogFormVisible = true" v-if="context.globalStatus == 0">
          开始
        </el-button>
        <el-button style="font-size: 20px;" type="success" @click="next" v-if="context.globalStatus == 2">下一秒
        </el-button>
        <el-button style="font-size: 20px;" type="success" @click="continuee" v-if="context.globalStatus == 2">继续
        </el-button>
        <el-button style="font-size: 20px;" type="info" @click="pause" v-if="context.globalStatus == 1">暂停
        </el-button>
        <el-button style="font-size: 20px;" type="warning" @click="doBlock"
          v-if="context.globalStatus == 1 || context.globalStatus == 2">
          阻塞当前线程</el-button>
        <el-popover v-if="context.globalStatus == 1 || context.globalStatus == 2" placement="bottom" :width="200"
          trigger="click">
          <template #reference>
            <el-button style="margin-right: 16px;font-size: 20px;" type="warning">选取进程以唤醒</el-button>
          </template>
          <div v-show="context.blockQueue.length > 0" v-for='(item, index) in context.blockQueue'>
            <input type="checkbox" v-model="selectedBlockProcessIndex" :value="index">
            <label>{{ item.name }}</label>
          </div>
          <div v-show="context.blockQueue.length == 0" style="color:#cccccc;text-align: center;">无可唤醒进程</div>
          <div v-show="context.blockQueue.length != 0">
            <el-button @click="doWake">唤醒选取进程</el-button>
          </div>
        </el-popover>
        <el-button style="font-size: 20px;" type="danger" @click="end" v-if="context.globalStatus != 0">结束
        </el-button>
        <el-dialog v-model="dialogFormVisible" title="开始运行">
          <el-form label-width="120px">
            <el-form-item label="选择调度算法">
              <el-select v-model="selectedSchedulingAlgorithIndex" placeholder="选择调度算法">
                <el-option v-for="(item, index) in schedulingAlgorithms" :key="item.name" :label="item.name"
                  :value="index" />
              </el-select>
            </el-form-item>
            <el-form-item label="选择初始线程">
              <el-select v-model="selectedInitProcessDataIndex" placeholder="选择初始线程">
                <el-option v-for="(item, index) in initData" :key="item.name" :label="item.name" :value="index" />
              </el-select>
            </el-form-item>
            <el-form-item v-show="selectedSchedulingAlgorithIndex == 4" label="反馈队列级数">
              <el-input-number v-model="MFQ_maxLevel" :min="1" :max="4" @change="handleMFQSlicesChange">
              </el-input-number>
            </el-form-item>
            <el-form-item v-show="selectedSchedulingAlgorithIndex == 1" label="时间片大小">
              <el-input-number v-model="RR_timeSlice" :min="1">
              </el-input-number>
            </el-form-item>
            <el-form-item v-show="selectedSchedulingAlgorithIndex == 4" label="各反馈队列">
              <table style="border-collapse: collapse;" border="1px">
                <tr>
                  <td width="60px">级数</td>
                  <td>时间片长度</td>
                </tr>
                <tr v-for="(item, index) in MFQ_timeSlices">
                  <td>{{ index + 1 }}</td>
                  <td>
                    <el-input-number v-model="MFQ_timeSlices[index]"
                      :min="index >= 1 ? MFQ_timeSlices[index - 1] + 1 : 1"
                      :max="index + 1 < MFQ_timeSlices.length ? MFQ_timeSlices[index + 1] - 1 : 99">
                    </el-input-number>
                  </td>
                </tr>
              </table>
            </el-form-item>
          </el-form>
          <template #footer>
            <span class="dialog-footer">
              <el-button @click="dialogFormVisible = false">取消</el-button>
              <el-button type="primary" @click="dialogFormVisible = false; context.globalStatus = 1; start()">
                自动运行
              </el-button>
              <el-button type="warning" @click="dialogFormVisible = false; context.globalStatus = 2; start()">
                手动下一步
              </el-button>
            </span>
          </template>
        </el-dialog>
      </span>
    </div>
    <!-- 队列部分 -->
    <div style="width:100%;margin-bottom:5px;display: flex;justify-content: center;">
      <table style="width:100%">
        <tr>
          <td width="85px"></td>
          <td>
            <div style="text-align: left;">队头</div>
          </td>
        </tr>
        <tr>
          <td style="border:1px solid black;border-radius: 5px;">新建队列</td>
          <td>
            <div class="queue">
              <el-popover v-for="(item) in context.newQueue" :hide-after="50" placement="top-start" :width="200"
                trigger="hover">
                <template #reference>
                  <div
                    :style="item.highlight ? 'color:white;background-color:red' : 'color:black;background-color:white'">
                    {{ item.name }}</div>
                </template>
                <table>
                  <tr class="hover-red">
                    <td>ID</td>
                    <td>{{ item.ID }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>名称</td>
                    <td>{{ item.name }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>优先级</td>
                    <td>{{ item.priority }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>到达时间</td>
                    <td>{{ item.arriveTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>需要时间</td>
                    <td>{{ item.serviceTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>已运行时间</td>
                    <td>{{ item.useCPUTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>等待时间</td>
                    <td>{{ item.waitTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>状态</td>
                    <td>{{ item.status }}</td>
                  </tr>
                </table>
              </el-popover>
            </div>
          </td>
        </tr>
        <tr>
          <td style="border:1px solid black;border-radius: 5px;">就绪队列</td>
          <td>
            <div v-for="(q, idx) in context.readyQueues">
              <div style="text-align: left;" v-if="selectedSchedulingAlgorithIndex == 4"><span
                  style="border: 1px solid black;">时间片{{ MFQ_timeSlices[idx] }}秒</span>
              </div>
              <div class="queue">
                <el-popover v-for="(item) in q" :hide-after="50" placement="top-start" :width="200" trigger="hover">
                  <template #reference>
                    <div
                      :style="item.highlight ? 'color:white;background-color:red' : 'color:black;background-color:white'">
                      {{ item.name }}</div>
                  </template>
                  <table>
                    <tr class="hover-red">
                      <td>ID</td>
                      <td>{{ item.ID }}</td>
                    </tr>
                    <tr class="hover-red">
                      <td>名称</td>
                      <td>{{ item.name }}</td>
                    </tr>
                    <tr class="hover-red">
                      <td>优先级</td>
                      <td>{{ item.priority }}</td>
                    </tr>
                    <tr class="hover-red">
                      <td>到达时间</td>
                      <td>{{ item.arriveTime }}</td>
                    </tr>
                    <tr class="hover-red">
                      <td>需要时间</td>
                      <td>{{ item.serviceTime }}</td>
                    </tr>
                    <tr class="hover-red">
                      <td>已运行时间</td>
                      <td>{{ item.useCPUTime }}</td>
                    </tr>
                    <tr class="hover-red">
                      <td>等待时间</td>
                      <td>{{ item.waitTime }}</td>
                    </tr>
                    <tr class="hover-red">
                      <td>状态</td>
                      <td>{{ item.status }}</td>
                    </tr>
                  </table>
                </el-popover>
              </div>
            </div>
          </td>
        </tr>
        <tr>
          <td style="border:1px solid black;border-radius: 5px;">运行队列 <span
              v-if="selectedSchedulingAlgorithIndex == 1">时间片:{{ RR_timeSlice
              }}秒</span></td>
          <td>
            <div class="queue">
              <el-popover v-for="(item) in context.runQueue" :hide-after="50" placement="top-start" :width="200"
                trigger="hover">
                <template #reference>
                  <div
                    :style="item.highlight ? 'color:white;background-color:red' : 'color:black;background-color:white'">
                    {{ item.name }}</div>
                </template>
                <table>
                  <tr class="hover-red">
                    <td>ID</td>
                    <td>{{ item.ID }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>名称</td>
                    <td>{{ item.name }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>优先级</td>
                    <td>{{ item.priority }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>到达时间</td>
                    <td>{{ item.arriveTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>需要时间</td>
                    <td>{{ item.serviceTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>已运行时间</td>
                    <td>{{ item.useCPUTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>等待时间</td>
                    <td>{{ item.waitTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>状态</td>
                    <td>{{ item.status }}</td>
                  </tr>
                </table>
              </el-popover>
            </div>
          </td>
        </tr>
        <tr>
          <td style="border:1px solid black;border-radius: 5px;">阻塞队列</td>
          <td>
            <div class="queue">
              <el-popover v-for="(item) in context.blockQueue" :hide-after="50" placement="top-start" :width="200"
                trigger="hover">
                <template #reference>
                  <div
                    :style="item.highlight ? 'color:white;background-color:red' : 'color:black;background-color:white'">
                    {{ item.name }}</div>
                </template>
                <table>
                  <tr class="hover-red">
                    <td>ID</td>
                    <td>{{ item.ID }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>名称</td>
                    <td>{{ item.name }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>优先级</td>
                    <td>{{ item.priority }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>到达时间</td>
                    <td>{{ item.arriveTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>需要时间</td>
                    <td>{{ item.serviceTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>已运行时间</td>
                    <td>{{ item.useCPUTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>等待时间</td>
                    <td>{{ item.waitTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>状态</td>
                    <td>{{ item.status }}</td>
                  </tr>
                </table>
              </el-popover>
            </div>
          </td>
        </tr>
        <tr>
          <td style="border:1px solid black;border-radius: 5px;">结束进程</td>
          <td>
            <div class="queue">
              <el-popover v-for="(item) in context.finishQueue" :hide-after="50" placement="top-start" :width="200"
                trigger="hover">
                <template #reference>
                  <div
                    :style="item.highlight ? 'color:white;background-color:red' : 'color:black;background-color:white'">
                    {{ item.name }}</div>
                </template>
                <table>
                  <tr class="hover-red">
                    <td>ID</td>
                    <td>{{ item.ID }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>名称</td>
                    <td>{{ item.name }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>优先级</td>
                    <td>{{ item.priority }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>到达时间</td>
                    <td>{{ item.arriveTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>需要时间</td>
                    <td>{{ item.serviceTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>已运行时间</td>
                    <td>{{ item.useCPUTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>等待时间</td>
                    <td>{{ item.waitTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>周转时间</td>
                    <td>{{ item.waitTime + item.serviceTime }}</td>
                  </tr>
                  <tr class="hover-red">
                    <td>状态</td>
                    <td>{{ item.status }}</td>
                  </tr>
                </table>
              </el-popover>
            </div>
          </td>
        </tr>
      </table>
    </div>
    <!-- 下面部分 -->
    <div style="width:100%;display: flex;justify-content: center;">
      <div
        style="border:1px solid black;border-radius: 5px;display: inline-block;margin-right:5px;height:295px;overflow-y: auto;">
        <el-tabs type="border-card" style="display: inline-block;">

          <el-tab-pane label="新建队列">
            <table style="border-collapse: collapse;" border="1px" cellpadding="10px">
              <thead>
                <tr>
                  <th width="35px">ID</th>
                  <th width="65px">名称</th>
                  <th v-if="selectedSchedulingAlgorithIndex == 2">优先级</th>
                  <th>到达时间</th>
                  <th>需要时间</th>
                  <th>已运行时间</th>
                  <th>等待时间</th>
                </tr>
              </thead>
              <tbody>
                <tr class="hover-red" v-for="item in context.newQueue">
                  <td>{{ item.ID }}</td>
                  <td>{{ item.name }}</td>
                  <td v-if="selectedSchedulingAlgorithIndex == 2">{{ item.priority }}</td>
                  <td>{{ item.arriveTime }}</td>
                  <td>{{ item.serviceTime }}</td>
                  <td>{{ item.useCPUTime }}</td>
                  <td>{{ item.waitTime }}</td>
                </tr>
              </tbody>
            </table>
          </el-tab-pane>
          <el-tab-pane label="就绪队列">
            <table style="border-collapse: collapse;" border="1px" cellpadding="10px">
              <thead>
                <tr>
                  <th width="35px">ID</th>
                  <th width="65px">名称</th>
                  <th v-if="selectedSchedulingAlgorithIndex == 2">优先级</th>
                  <th v-if="selectedSchedulingAlgorithIndex == 3">响应比</th>
                  <th>到达时间</th>
                  <th>需要时间</th>
                  <th>已运行时间</th>
                  <th>等待时间</th>
                </tr>
              </thead>
              <tbody v-for="q in context.readyQueues">
                <tr @mouseenter="item.highlight = true" @mouseleave="item.highlight = false" class="hover-red"
                  v-for="item in q">
                  <td>{{ item.ID }}</td>
                  <td>{{ item.name }}</td>
                  <td v-if="selectedSchedulingAlgorithIndex == 2">{{ item.priority }}</td>
                  <th v-if="selectedSchedulingAlgorithIndex == 3">
                    {{ 1 + (context.passTime - item.arriveTime) / item.serviceTime }}</th>
                  <td>{{ item.arriveTime }}</td>
                  <td>{{ item.serviceTime }}</td>
                  <td>{{ item.useCPUTime }}</td>
                  <td>{{ item.waitTime }}</td>
                </tr>
              </tbody>
            </table>
          </el-tab-pane>
          <el-tab-pane label="运行队列">
            <table style="border-collapse: collapse;" border="1px" cellpadding="10px">
              <thead>
                <tr>
                  <th width="35px">ID</th>
                  <th width="65px">名称</th>
                  <th v-if="selectedSchedulingAlgorithIndex == 2">优先级</th>
                  <th v-if="selectedSchedulingAlgorithIndex == 4">可用时间片</th>
                  <th>到达时间</th>
                  <th>需要时间</th>
                  <th>已运行时间</th>
                  <th>等待时间</th>
                </tr>
              </thead>
              <tbody>
                <tr @mouseenter="item.highlight = true" @mouseleave="item.highlight = false" class="hover-red"
                  v-for="item in context.runQueue">
                  <td>{{ item.ID }}</td>
                  <td>{{ item.name }}</td>
                  <td v-if="selectedSchedulingAlgorithIndex == 2">{{ item.priority }}</td>
                  <td v-if="selectedSchedulingAlgorithIndex == 4">{{ MFQ_timeSlices[MFQ.curLevel] }}秒</td>
                  <td>{{ item.arriveTime }}</td>
                  <td>{{ item.serviceTime }}</td>
                  <td>{{ item.useCPUTime }}</td>
                  <td>{{ item.waitTime }}</td>
                </tr>
              </tbody>
            </table>
          </el-tab-pane>
          <el-tab-pane label="阻塞队列">
            <table style="border-collapse: collapse;" border="1px" cellpadding="10px">
              <thead>
                <tr>
                  <th width="35px">ID</th>
                  <th width="65px">名称</th>
                  <th v-if="selectedSchedulingAlgorithIndex == 2">优先级</th>
                  <th>到达时间</th>
                  <th>需要时间</th>
                  <th>已运行时间</th>
                  <th>等待时间</th>
                </tr>
              </thead>
              <tbody>
                <tr @mouseenter="item.highlight = true" @mouseleave="item.highlight = false" class="hover-red"
                  v-for="item in context.blockQueue">
                  <td>{{ item.ID }}</td>
                  <td>{{ item.name }}</td>
                  <td v-if="selectedSchedulingAlgorithIndex == 2">{{ item.priority }}</td>
                  <td>{{ item.arriveTime }}</td>
                  <td>{{ item.serviceTime }}</td>
                  <td>{{ item.useCPUTime }}</td>
                  <td>{{ item.waitTime }}</td>
                </tr>
              </tbody>
            </table>
          </el-tab-pane>
          <el-tab-pane label="终止队列">
            <table style="border-collapse: collapse;" border="1px" cellpadding="10px">
              <thead>
                <tr>
                  <th width="35px">ID</th>
                  <th width="65px">名称</th>
                  <th v-if="selectedSchedulingAlgorithIndex == 2">优先级</th>
                  <th>到达时间</th>
                  <th>需要时间</th>
                  <th>已运行时间</th>
                  <th>等待时间</th>
                  <th>周转时间</th>
                </tr>
              </thead>
              <tbody>
                <tr @mouseenter="item.highlight = true" @mouseleave="item.highlight = false" class="hover-red"
                  v-for="item in context.finishQueue">
                  <td>{{ item.ID }}</td>
                  <td>{{ item.name }}</td>
                  <td v-if="selectedSchedulingAlgorithIndex == 2">{{ item.priority }}</td>
                  <td>{{ item.arriveTime }}</td>
                  <td>{{ item.serviceTime }}</td>
                  <td>{{ item.useCPUTime }}</td>
                  <td>{{ item.waitTime + item.serviceTime }}</td>
                </tr>
              </tbody>
            </table>
          </el-tab-pane>
        </el-tabs>
      </div>
      <div style="display: inline-block;vertical-align: top;height:294px;text-align: center;">
        <div id="create-process-area" style="height: 100%;border-radius: 5px;">
          <div style="font-size: 25px;">创建进程</div>
          <el-form ref="formRef" size="small" label-position="right" label-width="100px" :model="addProcessForm"
            style="padding:10px;">
            <el-form-item label="进程ID：">
              <el-input style="width:120px" type="text" v-model.number="addProcessForm.ID" disabled />
            </el-form-item>
            <el-form-item label="进程名：">
              <el-input style="width:120px" type="text" v-model="addProcessForm.name" disabled />
            </el-form-item>
            <el-form-item label="到达时间：">
              <el-input-number v-model="addProcessForm.arriveTime" :min="1"></el-input-number>
              <span style="margin-left:5px">秒后</span>
            </el-form-item>
            <el-form-item label="需要时间：">
              <el-input-number v-model="addProcessForm.serviceTime" :min="1"></el-input-number>
            </el-form-item>
            <el-form-item label="优先级：">
              <el-input-number v-model="addProcessForm.priority" :min="1"></el-input-number>
            </el-form-item>
            <el-form-item>
              <el-button :disabled="context.globalStatus == 0" type="primary" @click="addProcess(formRef)">添加
              </el-button>
              <el-button :disabled="context.globalStatus == 0" @click="resetAddProcessForm()">重置</el-button>
            </el-form-item>
          </el-form>
        </div>
      </div>
    </div>
  </div>
</template>


<style lang="less">
/* @import './assets/css/styles.less'; */

body {
  font-family: Roboto, -apple-system, BlinkMacSystemFont, 'Helvetica Neue', 'Segoe UI', 'Oxygen',
    'Ubuntu', 'Cantarell', 'Open Sans', sans-serif;
}

#page {
  text-align: center;
  max-width: 1020px;
  max-height: 670px;
}

.top-btn {
  vertical-align: middle;
  width: 100px;
  height: 30px;
  font-size: 22px;
  line-height: 30px;
}



.queue {
  border-top: 1px solid black;
  border-bottom: 1px solid black;
  height: 54px;
  width: 920px;
  display: inline-flex;
  background-color: #f5f7fa;
  overflow-x: scroll;
  border-radius: 5px;

  div {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    margin: 6px;
    border: 1px solid black;
    vertical-align: c;
    min-width: 80px;
    width: 80px;
    text-align: center;
    cursor: pointer;
    box-shadow: 1px 1px 8px black;
    border-radius: 5px;
  }
}

.queue::-webkit-scrollbar {
  width: 10px;
  height: 7px;
}

.queue::-webkit-scrollbar-thumb {
  border-radius: 10px;
  -webkit-box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.2);
  background: #cccccc;
}

.queue::-webkit-scrollbar-track {
  -webkit-box-shadow: inset 0 0 5px rgba(0, 0, 0, 0.2);
  border-radius: 10px;
  background: #ffffff;
}

#create-process-area {
  width: 330px;
  border: 1px solid black;
}

#other-option-area {
  width: 380px;
  border: 1px solid black;
  padding-bottom: 3px
}

.option-btn {
  font-size: 17px;
}

.hover-red:hover {
  color: red;
}
</style>
