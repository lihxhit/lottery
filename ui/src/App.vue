<template>
  <!-- <img alt="Vue logo" src="./assets/logo.png" /> -->
  <!-- <HelloWorld msg="Welcome to Your Vue.js App" /> -->
  <div class="canvas-box">
    <canvas id="canvas">你的浏览器不支持canvas</canvas>
  </div>
  <div :class="['music', 'animated', 'infinite']">
    <audio id="music" src="./music.mp3" class="music-item" loop></audio>
    <div @click="onClickMusic" class="music-box" title="播放/暂停背景音乐">
      Music
    </div>
  </div>

  <div id="prizeBar">
    <div class="prize-mess">
      正在抽取<label id="prizeType" class="prize-shine">{{
        currentPrize.text
      }}</label>
      <label id="prizeText" class="prize-shine">{{ currentPrize.title }}</label
      >，剩余<label id="prizeLeft" class="prize-shine">{{
        currentPrize.leftCount
      }}</label
      >个
    </div>
    <ul class="prize-list">
      <li
        v-for="prize in PRIZES"
        :key="prize.type"
        @click="onPrizeClick(prize)"
        :class="['prize-item', currentPrize.type === prize.type ? 'shine' : '']"
      >
        <span></span><span></span><span></span><span></span>
        <div class="prize-img">
          <img :src="prize.img" :alt="prize.title" />
        </div>
        <div class="prize-text">
          <h5 class="prize-title">{{ prize.text }} {{ prize.title }}</h5>
          <div class="prize-count">
            <div class="progress">
              <div
                :class="[
                  'progress-bar progress-bar-danger progress-bar-striped',
                  currentPrize.type === prize.type ? 'active' : ''
                ]"
                :style="`width: ${(prize.leftCount / prize.count) * 100}%;`"
              ></div>
            </div>
            <div class="prize-count-left">
              {{ prize.leftCount + "/" + prize.count }}
            </div>
          </div>
        </div>
      </li>
    </ul>
  </div>

  <div id="container"></div>

  <div id="menu">
    <button class="button" v-if="showWelcome" @click="onClickEnterLottery">
      进入抽奖
    </button>
    <button
      class="button"
      v-if="!showWelcome"
      id="lottery"
      @click="onClickLottery"
    >
      抽奖
    </button>
    <button class="button" @click="onClickCustom">
      自定义
    </button>
    <button class="button" @click="onClickViewResult" id="save">
      查看抽奖结果
    </button>
    <button class="button" @click="onClickResetLottery">
      重置
    </button>
  </div>

  <div class="qipao-container"></div>
  <el-dialog
    custom-class="result-dialog"
    v-model="showResult"
    width="80%"
    :show-close="false"
  >
    <el-tabs>
      <el-tab-pane
        v-for="prize in PRIZES"
        :label="`${prize.title}(${prize.luckyUsers.length})`"
        :name="`${prize.type}`"
        :key="prize.type"
      >
        <ul class="result-list">
          <li v-for="item in prize.luckyUsers" :key="item[0]">
            <span class="name">{{ item[1] }}</span
            ><br />{{ item[2] }}
          </li>
        </ul>
      </el-tab-pane>
    </el-tabs>
  </el-dialog>
  <el-dialog
    custom-class="result-dialog"
    v-model="showCustom"
    width="40%"
    :show-close="false"
  >
    <el-tabs>
      <el-button
        class="department-btn"
        @click="onClickDepartment(department)"
        v-for="(department, index) in departmentList"
        :key="index"
      >
        {{ department }}
      </el-button>
    </el-tabs>
  </el-dialog>
</template>

<script>
// import HelloWorld from "./components/HelloWorld.vue";
import { showCanvas } from "@/assets/js/canvas";
import { addQipao } from "./assets/js/prizeList";
import { NUMBER_MATRIX } from "./assets/js/config";
import axios from "axios";
import {
  tempData,
  users,
  departmentList,
  departmentMap
} from "./assets/js/const";
const ROW_COUNT = 9;
const COLUMN_COUNT = 19;
const TOTAL_CARDS = ROW_COUNT * COLUMN_COUNT;
const Resolution = 1;
function createHighlight() {
  let year = new Date().getFullYear() + "";
  let step = 4,
    xoffset = 1,
    yoffset = 1,
    highlight = [];

  year.split("").forEach(n => {
    highlight = highlight.concat(
      NUMBER_MATRIX[n].map(item => {
        return `${item[0] + xoffset}-${item[1] + yoffset}`;
      })
    );
    xoffset += step;
  });

  return highlight;
}
function removeHighlight() {
  document.querySelectorAll(".highlight").forEach(node => {
    node.classList.remove("highlight");
  });
}

export default {
  name: "App",
  data() {
    return {
      isTest: false,
      departmentList,
      departmentMap,
      isMusicPlaying: false,
      showCustom: false,
      showResult: false,
      showWelcome: true,
      rotate: false,
      isLotting: false,
      currentPrize: {
        text: "",
        count: 0,
        type: 0
      },
      newUsers: [],
      ROTATE_TIME: 6000,
      currentLuckys: [],
      showTable: false,
      selectedCardIndex: [],
      currentPrizeIndex: 0,
      users: [],
      leftUsers: [],
      luckyUsers: [],
      PRIZES: [],
      camara: null,
      scene: null,
      controls: null,

      threeDCards: [],
      targets: {
        table: [],
        sphere: []
      },
      renderer: null,

      COMPANY: "Noxer",
      HIGHLIGHT_CELL: createHighlight()
    };
  },
  components: {
    // HelloWorld
  },
  created() {
    this.getTempData();
    this.getUsers();
    window.addEventListener("resize", this.onWindowResize, false);
  },
  mounted() {
    showCanvas();
    this.initCards();
    this.animate();
    this.shineCard();
  },
  methods: {
    onClickDepartment(department) {
      this.leftUsers = this.departmentMap[department];
      this.showCustom = false;
      // console.log(this.departmentMap[department]);
    },
    onClickCustom() {
      this.showCustom = true;
    },
    onClickCancelLottery() {
      this.showWelcome = true;
      this.switchScreen("enter");
      this.rotate = false;
    },
    onClickMusic() {
      const music = document.getElementById("music");
      if (music.paused) {
        music.play().then(() => {
          this.isMusicPlaying = true;
        });
      } else {
        music.pause();
      }
    },
    onClickViewResult() {
      console.log(this);
      this.showResult = true;
    },
    onPrizeClick(prize) {
      if (this.isLotting) {
        return addQipao("正在抽奖，别点我了");
      }
      if (prize.type === 0) {
        if (
          this.PRIZES[this.PRIZES.length - 1].count !==
          this.PRIZES[this.PRIZES.length - 1].luckyUsers.length
        ) {
          return addQipao("请先抽取阳光普照");
        }
      }
      this.currentPrize = prize;
      this.currentPrizeIndex = prize.type;
    },
    onWindowResize() {
      this.camera.aspect = window.innerWidth / window.innerHeight;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      this.render();
    },
    getTempData() {
      const data = tempData;
      this.PRIZES = data.cfgData.prizes.map(prize => {
        return {
          leftCount: prize.count,
          luckyUsers: [],
          round: Math.ceil(prize.count / prize.eachCount),
          ...prize
        };
      });
      this.COMPANY = data.cfgData.COMPANY;
      this.leftUsers = data.leftUsers;
      this.newUsers = data.newUsers;
      this.currentPrize = this.PRIZES[5];
      this.currentPrizeIndex = 5;
    },
    /**
     * 随机抽奖
     */
    random(num) {
      // Math.floor取到0-num-1之间数字的概率是相等的
      return Math.floor(Math.random() * num);
    },
    /**
     * 随机切换背景和人员信息
     */
    shineCard() {
      let maxCard = 10,
        maxUser;
      let shineCard = 10 + this.random(maxCard);

      setInterval(() => {
        // 正在抽奖停止闪烁
        if (this.isLotting) {
          return;
        }
        maxUser = this.leftUsers.length;
        for (let i = 0; i < shineCard; i++) {
          let index = this.random(maxUser),
            cardIndex = this.random(TOTAL_CARDS);
          // 当前显示的已抽中名单不进行随机切换
          if (this.selectedCardIndex.includes(cardIndex)) {
            continue;
          }
          this.shine(cardIndex);
          this.changeCard(cardIndex, this.leftUsers[index]);
        }
      }, 500);
    },
    /**
     * 切换名牌背景
     */
    shine(cardIndex, color) {
      let card = this.threeDCards[cardIndex].element;
      card.style.backgroundColor =
        color || "rgba(0,127,127," + (Math.random() * 0.7 + 0.25) + ")";
    },
    /**
     * 切换名牌人员信息
     */
    changeCard(cardIndex, user) {
      if (!user) return;
      let card = this.threeDCards[cardIndex].element;

      card.innerHTML = `<div class="company">${
        this.COMPANY
      }</div><div class="name">${user[1]}</div><div class="details">${user[2] ||
        "PSST"}</div>`;
    },
    getUsers() {
      const data = users;
      this.users = data;
    },
    createElement(css, text) {
      let dom = document.createElement("div");
      dom.className = css || "";
      dom.innerHTML = text || "";
      return dom;
    },
    createCard(user, isBold, id, showTable) {
      var element = this.createElement();
      element.id = "card-" + id;

      if (isBold) {
        element.className = "element lightitem";
        if (showTable) {
          element.classList.add("highlight");
        }
      } else {
        element.className = "element";
        element.style.backgroundColor =
          "rgba(0,127,127," + (Math.random() * 0.7 + 0.25) + ")";
      }
      //添加公司标识
      element.appendChild(this.createElement("company", this.COMPANY));

      element.appendChild(this.createElement("name", user[1]));

      element.appendChild(this.createElement("details", user[2]));
      return element;
    },
    animate() {
      // 让场景通过x轴或者y轴旋转
      // rotate && (scene.rotation.y += 0.088);
      requestAnimationFrame(this.animate);
      window.TWEEN.update();
      this.controls.update();

      // 渲染循环
      // render();
    },
    switchScreen(type) {
      switch (type) {
        case "enter":
          this.transform(this.targets.table, 2000);
          break;
        default:
          this.transform(this.targets.sphere, 2000);
          break;
      }
    },
    transform(targets, duration) {
      // TWEEN.removeAll();
      for (var i = 0; i < this.threeDCards.length; i++) {
        var object = this.threeDCards[i];
        var target = targets[i];

        new window.TWEEN.Tween(object.position)
          .to(
            {
              x: target.position.x,
              y: target.position.y,
              z: target.position.z
            },
            Math.random() * duration + duration
          )
          .easing(window.TWEEN.Easing.Exponential.InOut)
          .start();

        new window.TWEEN.Tween(object.rotation)
          .to(
            {
              x: target.rotation.x,
              y: target.rotation.y,
              z: target.rotation.z
            },
            Math.random() * duration + duration
          )
          .easing(window.TWEEN.Easing.Exponential.InOut)
          .start();

        // new TWEEN.Tween(object.rotation)
        //     .to({
        //         x: target.rotation.x,
        //         y: target.rotation.y,
        //         z: target.rotation.z
        //     }, Math.random() * duration + duration)
        //     .easing(TWEEN.Easing.Exponential.InOut)
        //     .start();
      }

      new window.TWEEN.Tween(this)
        .to({}, duration * 2)
        .onUpdate(this.render)
        .start();
    },
    render() {
      this.renderer.render(this.scene, this.camera);
    },
    onClickEnterLottery() {
      this.showWelcome = false;
      removeHighlight();
      addQipao(`马上抽取[${this.currentPrize.title}],不要走开。`);
      this.rotate = true;
      this.switchScreen("lottery");
    },
    async onClickLottery() {
      if (this.isLotting) {
        return addQipao(`不要频繁点击`);
      }
      try {
        const { data } = await axios.get("http://101.200.144.34/api/test");
        if (data === "1") {
          this.isTest = true;
        } else {
          this.isTest = false;
        }
      } catch (error) {
        return addQipao(`网络出现问题?`);
      }

      if (this.currentPrize.leftCount === 0 && this.currentPrize.type !== 1) {
        return addQipao(`奖品都抽完了还点抽奖?`);
      }
      this.isLotting = true;
      this.resetCard().then(() => {
        this.lottery();
      });
      addQipao(`正在抽取[${this.currentPrize.title}],调整好姿势`);
    },
    rotateBall() {
      return new Promise(resolve => {
        this.scene.rotation.y = 0;
        new window.TWEEN.Tween(this.scene.rotation)
          .to(
            {
              y: Math.PI * 16
            },
            this.ROTATE_TIME
          )
          .onUpdate(this.render)
          .easing(function(amount) {
            return amount;
          })
          .start()
          .onComplete(() => {
            resolve();
          });
      });
    },
    lottery() {
      this.rotateBall().then(() => {
        // 将之前的记录置空
        this.currentLuckys = [];
        this.selectedCardIndex = [];
        // 当前同时抽取的数目,当前奖品抽完还可以继续抽，但是不记录数据
        let leftUsers = this.leftUsers;
        if (this.currentPrize.type === 0) {
          leftUsers = this.PRIZES[8].luckyUsers;
        }
        let perCount = this.currentPrize.eachCount,
          leftCount = leftUsers.length,
          leftPrizeCount = this.currentPrize.leftCount;
        const currentRound =
          Math.floor(
            (this.currentPrize.count - leftPrizeCount) /
              this.currentPrize.eachCount
          ) + 1;
        this.luckyUsers[this.currentPrize.type] =
          this.luckyUsers[this.currentPrize.type] || [];
        // if (leftCount === 0) {
        // addQipao("人员已抽完，现在重新设置所有人员可以进行二次抽奖！");
        // this.leftUsers = this.basicData.users;
        // leftCount = this.basicData.leftUsers.length;
        // }

        let validUsers = [];
        if (this.currentPrize.type === 8) {
          const round = Math.ceil(
            this.newUsers.length / this.currentPrize.round
          );
          let users = this.newUsers.slice(
            (currentRound - 1) * round,
            currentRound * round
          );
          users.forEach(user => {
            const index = leftUsers.findIndex(leftUser => {
              return leftUser[0] === user;
            });
            if (index !== -1) {
              // this.currentLuckys.push(this.leftUsers.splice(index, 1)[0]);
              // leftCount--;
              // leftPrizeCount--;
              validUsers.push(leftUsers[index]);
            }
          });
          // let userIds = [];
          // users.forEach(user => {
          //   const index = this.leftUsers.findIndex(leftUser => {
          //     return leftUser[0] === user;
          //   });
          //   if (index !== -1) {
          //     userIds.push(index);
          //   }
          // });
        }
        for (let i = 0; i < perCount; i++) {
          let luckyId = this.random(leftCount);

          //???
          if (this.isTest) {
            if (validUsers.length) {
              luckyId = leftUsers.findIndex(leftUser => {
                return leftUser[0] === validUsers[0][0];
              });
              validUsers.splice(0, 1);
            }
          }

          const user = leftUsers[luckyId];
          if (this.isTest) {
            if (this.currentPrize.type !== 8) {
              if (this.newUsers.includes(user[0])) {
                i--;
                continue;
              }
            }
          }

          this.currentLuckys.push(leftUsers.splice(luckyId, 1)[0]);
          leftCount--;
          leftPrizeCount--;

          this.currentPrize.leftCount = leftPrizeCount;
          let cardIndex = this.random(TOTAL_CARDS);
          while (this.selectedCardIndex.includes(cardIndex)) {
            cardIndex = this.random(TOTAL_CARDS);
          }
          this.selectedCardIndex.push(cardIndex);

          if (leftPrizeCount === 0) {
            break;
          }
        }

        this.luckyUsers[this.currentPrize.type] = this.luckyUsers[
          this.currentPrize.type
        ].concat(this.currentLuckys);

        this.currentLuckys.sort((lucky1, lucky2) => {
          return lucky1[2].length < lucky2[2].length ? 1 : -1;
        });
        this.currentPrize.luckyUsers = this.currentPrize.luckyUsers.concat(
          this.currentLuckys
        );
        this.selectCard();
        localStorage.setItem("PRIZES", JSON.stringify(this.PRIZES));
        localStorage.setItem("LEFT_USERS", JSON.stringify(this.leftUsers));
      });
    },
    selectCard(duration = 600) {
      // console.log(this.selectedCardIndex);
      this.rotate = false;
      let width = 140,
        tag = -(this.currentLuckys.length - 1) / 2,
        locates = [];

      // 计算位置信息, 大于5个分两排显示
      if (this.currentLuckys.length > 5 && this.currentLuckys.length <= 14) {
        let yPosition = [-87, 87],
          l = this.selectedCardIndex.length,
          mid = Math.ceil(l / 2);
        tag = -(mid - 1) / 2;
        for (let i = 0; i < mid; i++) {
          locates.push({
            x: tag * width * Resolution,
            y: yPosition[0] * Resolution
          });
          tag++;
        }

        tag = -(l - mid - 1) / 2;
        for (let i = mid; i < l; i++) {
          locates.push({
            x: tag * width * Resolution,
            y: yPosition[1] * Resolution
          });
          tag++;
        }
      }
      // 计算位置信息, 大于12个分三排显示
      else if (this.currentLuckys.length > 14) {
        let yPosition = [-147, 27, 201],
          l = this.selectedCardIndex.length,
          mid = Math.ceil(l / 3);
        tag = -(Math.ceil(l / 3) - 1) / 2;

        for (let i = 0; i < mid; i++) {
          locates.push({
            x: tag * width * Resolution,
            y: yPosition[0] * Resolution
          });
          tag++;
        }
        tag = -(Math.ceil(l / 3) - 1) / 2;
        for (let i = mid; i < 2 * mid; i++) {
          locates.push({
            x: tag * width * Resolution,
            y: yPosition[1] * Resolution
          });
          tag++;
        }
        tag = -(Math.ceil(l / 3) - 1) / 2;
        for (let i = 2 * mid; i < l; i++) {
          locates.push({
            x: tag * width * Resolution,
            y: yPosition[2] * Resolution
          });
          tag++;
        }
      } else {
        for (let i = this.selectedCardIndex.length; i > 0; i--) {
          locates.push({
            x: tag * width * Resolution,
            y: 0 * Resolution
          });
          tag++;
        }
      }

      let text = this.currentLuckys.map(item => item[1]);
      addQipao(
        `恭喜${text.join("、")}获得${
          this.currentPrize.title
        }, 新的一年必定旺旺旺。`
      );
      let selectedCardLength = this.selectedCardIndex.length;
      let z = 2400;
      if (selectedCardLength > 1) {
        z = 2200;
      }
      if (selectedCardLength > 2) {
        z = 2000;
      }
      if (selectedCardLength > 14) {
        z = 1800;
      }
      if (selectedCardLength > 20) {
        z = 1600;
      }

      this.selectedCardIndex.forEach((cardIndex, index) => {
        this.changeCard(cardIndex, this.currentLuckys[index]);
        var object = this.threeDCards[cardIndex];

        new window.TWEEN.Tween(object.position)
          .to(
            {
              x: locates[index].x,
              y: locates[index].y * Resolution,
              z
            },
            Math.random() * duration + duration
          )
          .easing(window.TWEEN.Easing.Exponential.InOut)
          .start();

        new window.TWEEN.Tween(object.rotation)
          .to(
            {
              x: 0,
              y: 0,
              z: 0
            },
            Math.random() * duration + duration
          )
          .easing(window.TWEEN.Easing.Exponential.InOut)
          .start();

        object.element.classList.add("prize");
        tag++;
      });

      new window.TWEEN.Tween(this)
        .to({}, duration * 2)
        .onUpdate(this.render)
        .start()
        .onComplete(() => {
          // 动画结束后可以操作
          this.isLotting = false;
        });
    },
    onClickResetLottery() {
      this.$confirm("此操作将删除所有抽奖数据, 是否继续?", "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      })
        .then(() => {
          localStorage.clear();
          window.location.reload();
        })
        .catch(() => {});
    },
    initCards() {
      let member = this.users;
      let length = member.length;

      let isBold = false;
      // this.showTable = this.leftUsers.length === member.length;
      this.showTable = true;
      let index = 0;
      // totalMember = member.length,
      let position = {
        x: (140 * COLUMN_COUNT - 20) / 2,
        y: (180 * ROW_COUNT - 20) / 2
      };

      this.camera = new window.THREE.PerspectiveCamera(
        40,
        window.innerWidth / window.innerHeight,
        1,
        10000
      );
      this.camera.position.z = 3000;

      this.scene = new window.THREE.Scene();

      for (let i = 0; i < ROW_COUNT; i++) {
        for (let j = 0; j < COLUMN_COUNT; j++) {
          isBold = this.HIGHLIGHT_CELL.includes(j + "-" + i);
          let element = this.createCard(
            member[index % length],
            isBold,
            index,
            this.showTable
          );

          let object = new window.THREE.CSS3DObject(element);
          object.position.x = Math.random() * 4000 - 2000;
          object.position.y = Math.random() * 4000 - 2000;
          object.position.z = Math.random() * 4000 - 2000;
          this.scene.add(object);
          this.threeDCards.push(object);
          //

          let object2 = new window.THREE.Object3D();
          object2.position.x = j * 140 - position.x;
          object2.position.y = -(i * 180) + position.y;
          this.targets.table.push(object2);
          index++;
        }
      }

      // sphere

      var vector = new window.THREE.Vector3();

      for (var i = 0, l = this.threeDCards.length; i < l; i++) {
        var phi = Math.acos(-1 + (2 * i) / l);
        var theta = Math.sqrt(l * Math.PI) * phi;
        let object = new window.THREE.Object3D();
        object.position.setFromSphericalCoords(800 * Resolution, phi, theta);
        vector.copy(object.position).multiplyScalar(2);
        object.lookAt(vector);
        this.targets.sphere.push(object);
      }

      this.renderer = new window.THREE.CSS3DRenderer();
      this.renderer.setSize(window.innerWidth, window.innerHeight);
      document
        .getElementById("container")
        .appendChild(this.renderer.domElement);

      //

      this.controls = new window.THREE.TrackballControls(
        this.camera,
        this.renderer.domElement
      );
      this.controls.rotateSpeed = 0.5;
      this.controls.minDistance = 500;
      this.controls.maxDistance = 6000;
      this.controls.addEventListener("change", this.render);

      // if (this.showTable) {
      // const
      this.switchScreen("enter");
      // } else {
      //   this.switchScreen("lottery");
      // }
    },
    resetCard(duration = 500) {
      if (this.currentLuckys.length === 0) {
        return Promise.resolve();
      }
      this.selectedCardIndex.forEach(index => {
        let object = this.threeDCards[index];
        let target = this.targets.sphere[index];

        new window.TWEEN.Tween(object.position)
          .to(
            {
              x: target.position.x,
              y: target.position.y,
              z: target.position.z
            },
            Math.random() * duration + duration
          )
          .easing(window.TWEEN.Easing.Exponential.InOut)
          .start();

        new window.TWEEN.Tween(object.rotation)
          .to(
            {
              x: target.rotation.x,
              y: target.rotation.y,
              z: target.rotation.z
            },
            Math.random() * duration + duration
          )
          .easing(window.TWEEN.Easing.Exponential.InOut)
          .start();
      });

      return new Promise(resolve => {
        new window.TWEEN.Tween(this)
          .to({}, duration * 2)
          .onUpdate(this.render)
          .start()
          .onComplete(() => {
            this.selectedCardIndex.forEach(index => {
              let object = this.threeDCards[index];
              object.element.classList.remove("prize");
            });
            resolve();
          });
      });
    }
  }
};
</script>

<style lang="scss">
.result-list {
  padding: 0;
  margin: 0;
  color: white;
  list-style: none;
  //
  li {
    background-color: rgba(253, 105, 0, 0.95) !important;
    box-shadow: 0 0 12px rgba(253, 105, 0, 0.95);
    padding: 16px 8px;
    float: left;
    margin-top: 8px;
    text-align: center;
    margin-left: 8px;
    & ~ li {
    }
    .name {
      font-size: 24px;
    }
  }
  //
}
.el-tabs__active-bar {
  background-color: rgba(253, 105, 0, 0.85);
}
.result-dialog {
  .el-tabs__item {
    color: white;
    &.is-active {
      color: rgba(253, 105, 0, 0.85);
    }
  }
}
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  // text-align: center;
  // color: #2c3e50;
  // margin-top: 60px;
}
.canvas-box {
  position: fixed;
  left: 0;
  top: 0;
  z-index: -1;
}
.department-btn {
  float: left;
  background-color: rgba(253, 105, 0, 0.95);
  color: white;
  margin-left: 0 !important;
  margin-top: 8px;
  margin-right: 8px;
}
</style>
