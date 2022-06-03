<template>
  <div class="common-layout">
    <el-container>
      <el-aside>
        <!-- 相关信息的检索 -->
        <h1>作业指令总数</h1>
        <p>{{ totalAssignmentNum }}</p>
        <h1>每页存放指令数</h1>
        <p>{{ instructionPerPage }}</p>
        <h1>作业占用内存页数</h1>
        <p>{{ usePageNum }}</p>
        <h1>页面置换算法</h1>
        <p>FIFO</p>
        <h1>当前执行页数</h1>
        <p>{{ curStep }}</p>
        <h1>缺页数</h1>
        <p>{{ missingPage }}</p>
        <h1>缺页率</h1>
        <p>{{ missingRate }}</p>
        <h1>执行速度(ms)</h1>
        <p>
          <!-- 可以修改指令连续运行的速度 -->
          <el-input
            v-model="timePerExecution"
            style="width: 100px"
            placeholder="默认300"
          />
        </p>
      </el-aside>
      <el-container>
        <el-header>
          <p>
            Author：2052329 方必诚 请求调页存储管理方式模拟
            <el-link
              href="https://github.com/Baokker/page_management_project"
              target="前往github项目首页"
              >项目GitHub源码</el-link
            >
          </p></el-header
        >

        <el-main>
          <el-container>
            <el-aside>
              <!-- 显示实时Memory中情况 -->
              <table>
                <th v-for="m in this.usePageNum" :key="m">
                  <p>第{{ m }}块</p>
                  <p>
                    内存中第{{
                      usePages[m - 1] == undefined ? "XX" : usePages[m - 1]
                    }}块
                  </p>
                </th>

                <tr v-for="i in this.instructionPerPage" :key="i">
                  <td v-for="j in this.usePageNum" :key="j">
                    <el-check-tag
                      size="large"
                      :checked="
                        this.usePages[j - 1] * this.instructionPerPage + i ==
                        this.selectedID
                      "
                      >{{ i }}</el-check-tag
                    >
                  </td>
                </tr>
              </table>

              <p></p>

              <div style="text-align: center">
                <el-button type="primary" @click="moveOneStep"
                  >单步运行</el-button
                >
                <el-button type="warning" @click="moveForward"
                  >{{ continuousExecutionReminder }}连续执行</el-button
                >
                <el-button type="info" @click="reset">重设</el-button>
              </div>
              <p></p>
            </el-aside>
            <el-main>
              <!-- 显示已经执行的命令表格 -->
              <h1>已经执行的命令</h1>
              <el-table :data="outputData" max-height="400px">
                <el-table-column prop="id" label="执行地址" />
                <el-table-column prop="isPageMissing" label="是否缺页" />
                <el-table-column prop="pageOut" label="换出页" />
                <el-table-column prop="pageIn" label="换入页" />
              </el-table>
            </el-main>
          </el-container>
        </el-main>
      </el-container>
    </el-container>
  </div>
</template>

<script>
import { ElMessage } from "element-plus";

export default {
  name: "app",
  data() {
    return {
      // continuousExecutionReminder: "开始",
      isBeginLoop: false,
      selectedID: undefined,
      totalAssignmentNum: 320,
      instructionPerPage: 10,
      usePageNum: 4,
      missingPage: 0,
      // missingRate: 0,
      usePages: [],
      lastAddress: undefined,
      curStep: 0, // start from 0
      order: [],
      outputData: [],
      timePerExecution: 300,
    };
  },
  computed: {
    missingRate() {
      return this.curStep == 0 ? "XX" : this.missingPage / this.curStep;
    },
    continuousExecutionReminder() {
      return this.isBeginLoop ? "暂停" : "开始";
    },
  },
  watch: {
    executeSpeed(timePerExecution) {
      if (isNaN(timePerExecution)) {
        this.timePerExecution = timePerExecution;
      }
    },
  },
  methods: {
    // 重开过程
    reset() {
      this.missingPage = 0;
      this.order = [];
      this.usePages = [];
      this.outputData = [];
      this.curStep = 0;
    },
    // 产生下一个地址
    // range: [0,319]
    generateTask() {
      var result;
      if (this.lastAddress != undefined) {
        let proability = Math.random();
        console.log("proability: " + proability);
        if (proability < 0.5) {
          if (this.lastAddress + 1 < this.totalAssignmentNum) {
            result = this.lastAddress + 1;
          } else {
            result = Math.floor(Math.random() * this.totalAssignmentNum);
          }
        } else if (proability >= 0.5 && proability < 0.75) {
          result = Math.floor(
            Math.random() * (this.totalAssignmentNum - this.lastAddress) +
              this.lastAddress
          );
        } else {
          result = Math.floor(Math.random() * this.lastAddress);
        }
      } else {
        result = Math.floor(Math.random() * this.totalAssignmentNum);
      }
      this.lastAddress = result;
      this.selectedID = result;
      return result;
    },
    // 先入先出选择替换的page
    FIFO(pageID) {
      var index = this.usePages.indexOf(this.order.shift());
      var pre = this.usePages[index];
      this.move(pageID, index);
      return "第" + index.toString() + "块，地址为：" + pre.toString();
    },
    // 方法：将第pageID页转移到memoryID的页面中
    move(pageID, memoryID) {
      console.log("pageID: ", pageID, "memoryID: ", memoryID);
      if (memoryID < 0 || memoryID >= this.usePageNum) {
        alert("out of range !!");
      }
      this.order.push(pageID);
      this.usePages[memoryID] = pageID;
      console.log("MOVEEEEE this.usePages[%d]= %d", memoryID, pageID);
    },
    // 跳转到下一个地址
    // 如果没有直接找到，就根据情况
    // 如果usePage未满（最开始），则调入新的页面
    // 否则采用FIFO
    // 同时将数据输出到outputData中
    jumptoNext(nextAddress) {
      let isFound = false;
      let pageID = Math.floor(nextAddress / this.instructionPerPage);
      for (var i = 0; i < this.usePages.length; i++)
        if (this.usePages[i] == pageID) isFound = true;

      if (isFound == true) {
        this.outputData.push({
          id: nextAddress,
          isPageMissing: false,
          pageOut: "-",
          pageIn: "-",
        });
      } else {
        this.missingPage++;
        // false
        if (this.usePages.length < this.usePageNum) {
          this.move(pageID, this.usePages.length);
          this.outputData.push({
            id: nextAddress,
            isPageMissing: true,
            pageOut: "-",
            pageIn: pageID,
          });
        } else {
          this.outputData.push({
            id: nextAddress,
            isPageMissing: false,
            pageOut: this.FIFO(pageID),
            pageIn: pageID,
          });
        }
      }
      console.log(
        "nextaddress: " +
          nextAddress +
          " pageID: " +
          pageID +
          " usePages: " +
          this.usePages
      );
    },
    // 移动一步
    moveOneStep() {
      if (this.curStep >= this.totalAssignmentNum) {
        ElMessage.success({
          type: "success",
          message: "执行完毕！",
        });
        return;
      }
      console.log("curstep: " + this.curStep);
      this.jumptoNext(this.generateTask());
      this.curStep++;
    },
    // 连续移动
    // setInterval定时调用
    moveForward() {
      if (this.curStep > this.totalAssignmentNum) {
        ElMessage.success({
          type: "success",
          message: "执行完毕！",
        });
      }
      if (this.isBeginLoop == true) {
        clearInterval(this.timer);
        this.isBeginLoop = false;
      } else {
        this.timer = setInterval(() => {
          if (this.curStep < this.totalAssignmentNum) this.moveOneStep();
        }, this.timePerExecution);
        this.isBeginLoop = true;
      }
    },
  },
};
</script>

<style>
h1 {
  text-align: center;
}
p {
  text-align: center;
}
table {
  text-align: center;
  border: 0;
}
.el-header {
  position: relative;
  margin: 10px;
  border: 2px solid #ecf5ff;

  border-radius: 10px;
  -webkit-box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
  -moz-box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
  box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
}
.el-aside {
  color: var(--el-text-color-primary);
  /* background: var(--el-color-primary-light-8); */
  border-radius: 10px;
  margin: 10px;
  border: 2px solid #ecf5ff;

  -webkit-box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
  -moz-box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
  box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
}
.el-main {
  border-radius: 10px;
  margin: 10px;
  border: 2px solid #ecf5ff;

  -webkit-box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
  -moz-box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
  box-shadow: 0px 9px 5px 0px rgba(50, 50, 50, 0.28);
}
body {
  margin: 10px;
}
</style>
