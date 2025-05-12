<!--  -->
<script setup>
import SubmitBtn from "@/views/aiSystem/components/button/SubmitBtn.vue";
import PromptInfo from "../template/components/PromptInfo.vue";
import { useChatStore, useSiteStore, useAgentStore } from "@/stores";
import { nextTick, onMounted, ref } from "vue";
import { useScroll } from "@/hooks/useScroll";
import { local } from "@/utils/storage";
import { chatWithHistory, abort } from "@/api/aiSystem/ollama";
import { render } from "@/utils/markdown";
import { ElMessage } from "element-plus";
import moment from "moment";
import useEvent from "./hooks/useEvent";
import useActiveAgentInfo from "./composable/useActiveAgentInfo";
import useChatHistory from "./composable/useChatHistory";
import getChatHistory from "./helper";
import {
  useTemplateInfo,
  updateTemplateInfo,
} from "./composable/useTemplateInfo";
import stop from "@/assets/aiSystem/stop.svg";
const input = defineModel();
const chatStore = useChatStore();
const agentStore = useAgentStore();
const siteStore = useSiteStore();
const leftPanel = ref(null);

const sending = ref(false); // 发送中的状态
const agentInfo = useActiveAgentInfo();
const chatHistory = useChatHistory(agentInfo.value?.id);
const { prompt } = useTemplateInfo(agentInfo.value?.agentPersona);
const { scrollRef, scrollToBottom } = useScroll();
const detailWindowVisible = ref(null);
const importInput = ref(null);
const autoScrollSwitch = ref(siteStore.siteState.autoScroll);
//是否显示思考
const isReflectionBtn = ref(true);
//计算盒子宽度
const chatAreaWidth = ref(500);
onMounted(async () => {
  if (agentInfo.value) {
    await nextTick();
    leftPanel.value.style.width = siteStore.siteState.chatBarW;
    scrollToBottom();
    useEvent();
  }
  getDivWidth();
});
//监听屏幕宽度
onMounted(() => {
  window.onresize = () => {
    return (() => {
      getDivWidth();
    })();
  };
});
function addUserChat(msg) {
  const userChat = {
    inversion: false,
    text: msg,
    chatForm: '',
    isNeedBmiForm: false,
    isNeedProcurePlanForm: false,
    isNeedProcureShopSelectForm: false,
  };
  // 一次消息，生成一个userChat对象。
  chatStore.addChat(agentInfo.value.id, userChat);
}

function addEmptyBotChat() {
  const botChat = {
    inversion: true,
    text: "",
    timestamp: new Date().toISOString(),
    //think
    think: "",
  };
  chatHistory.value.push(botChat); // Will not save immediately
}

async function SubmitChat() {
  if (sending.value) return;
  const userMsg = input.value;
  if (!userMsg || userMsg.trim() === "") return;
  (sending.value = true), (input.value = "");
  const history = getChatHistory(chatHistory.value, chatHistory.value.length);
  addUserChat(userMsg);
  addEmptyBotChat();
  scrollToBottom(true);
  _chat(userMsg, -1, history);
}

function reGenerate(idx) {
  if (sending.value || idx < 1) return;
  sending.value = true;
  const userChat = chatHistory.value[idx - 1];
  if (userChat.inversion) {
    ElMessage({
      message: "Chat has been deleted.",
      type: "warning",
    });
    sending.value = false;
    return;
  }
  const history = getChatHistory(chatHistory.value, idx);
  _chat(userChat.text, idx, history);
}

function delChat(idx) {
  if (sending.value || idx < 0) return;
  chatStore.removeChatItem(agentInfo.value.id, idx);
  ElMessage({
    message: "Chat has been deleted.",
    type: "success",
  });
}

function botThinking(idx) {
  const thinkingText = ["⠸", "⠼", "⠴", "⠦", "⠧", "⠇", "⠏", "⠋", "⠙", "⠹"];
  let i = 0;
  const loadingInterval = setInterval(() => {
    chatHistory.value.at(idx).text = thinkingText[i];
    i = (i + 1) % thinkingText.length;
  }, 100);
  return () => {
    clearInterval(loadingInterval);
    chatHistory.value.at(idx).text = "";
  };
}
let isThink = ref(false);

// let formInputText = '';
// 体重表单
let bmiHeight = ref(0);
let bmiWeight = ref(0);
let formInputText = ref('');

function bmiSubmit() {
  formInputText.value='我的身高是'+bmiHeight.value+' cm, 我的体重是'+bmiWeight.value+' kg. 请帮我计算我的BMI。';
  SubmitFormChat()
  formInputText.value = '';
}

// 采购表单
let procureStage = ref(0);
// 1=while selecting products
// 2=while selecting shops
/**
我想发起一个采购任务。当我问你“我计划采购...”，你回复“请告诉我采购的商品的名称，价格，数量。我来帮你估算价格。”
当我告诉你商品的名称，价格，数量的时候，你告诉我从Woolworths,Coles两个门店的估算价格。Woolworths门店的单价都按1元算。Coles门店的单价都按1.5元算。
 */
let productIdx = ref(2);

let procureProducts = ref([
  {
    id: 1,
    name: '铅笔',
    price: 1.5,
    num: 1,
  }
]);

function addProcureProduct() {
  procureProducts.value.push(
    {
      id: productIdx.value++,
      name: '',
      price: 1,
      num: 1,
    }
  );
}

function removeProcureProduct(productId) {
  procureProducts.value = procureProducts.value.filter((item) => item.id != productId);
}

async function procurePlanSubmit() {
  let text = '我计划采购 ';
  procureProducts.value.forEach(function(item) {
    text += '' + item.name
    +' , 预估单价 $'+item.price
    +' , 数量是 '+item.num
    +'; '
  });
  text += '. 请帮我从Coles,Woolworths商店中找出合适的产品。';
  formInputText.value = text;
  SubmitFormChat();
  procureStage.value = 2; // 提交后，到下一个流程
  formInputText.value = '';
}

// 产品候选商店列表
/**
当我告诉你"请帮我估算出各个商店的价格。"你帮我分别列出Woolworths,Coles算出的估算价格。他们的单价都按1元算。
 */
let procureShopList = ref([
  {
    name: 'Woolworths',
    estimatedPrice: 15,
  },
  {
    name: 'Coles',
    estimatedPrice: 12,
  },
]);

let procureShopSelected = ref('');

async function procureShopSubmit() {
  formInputText.value='我的采购计划使用 '+procureShopSelected.value+' 作为供货方. 请帮我发出审批信。';
  SubmitFormChat();
  procureStage.value = 3; // 提交后，到下一个流程
  formInputText.value = '';
}

async function SubmitFormChat() {
  // formInputText.value='我的身高是'+bmiHeight.value+' cm, 我的体重是'+bmiWeight.value+' kg. 请帮我计算我的BMI。';
  if (sending.value) return;
  // const userMsg = input.value;
  const userMsg = formInputText.value;
  if (!userMsg || userMsg.trim() === "") return;
  (sending.value = true), (input.value = "");
  formInputText.value = '';
  const history = getChatHistory(chatHistory.value, chatHistory.value.length);
  addUserChat(userMsg);
  addEmptyBotChat();
  scrollToBottom(true);
  _chat(userMsg, -1, history);
}

//判断是否deepseek
let testRegex = /deepseek/;
function _chat(userMsg, idx, history) {
  const thinkingDone = botThinking(idx);
  chatWithHistory(agentInfo.value.model, userMsg, history, prompt.value.value)
    .then(async (stream) => {
      thinkingDone();
      console.log("agentInfo.value.model", agentInfo.value.model);
      let isBMIDetected = false;
      let isProcurePlanDetected = false;
      let isProcureShopDetected = false;

      let isProcureProductOccured = false;
      let isProcureProductPriceOccured = false;
      let isProcureProductNumOccured = false;

      let isProcureShopW = false;
      let isProcureShopC = false;

      for await (const part of stream) {
        // chatHistory.value.at(idx).text += part.message.content;
        let tep_mesg = part.message.content;
        //判断是否是deepseek模型
        if (testRegex.test(agentInfo.value.model)) {

          // detect bmi 
          if (tep_mesg == 'BMI') {
            isBMIDetected = true;
          }
          if (tep_mesg == '最终' || tep_mesg == '总结') {
            isBMIDetected = false;
          }
          // detect procurement plan
          if (tep_mesg == '采购') {
            isProcurePlanDetected = true;
          }
          if (tep_mesg == '商品') {
            isProcureProductOccured = true;
          }
          if (tep_mesg == '价格') {
            isProcureProductPriceOccured = true;
          }
          if (tep_mesg == '数量') {
            isProcureProductNumOccured = true;
          }
          if (tep_mesg == '最终' || tep_mesg == '总结') {
            isProcurePlanDetected = false;
          }

          // detect procurement shop select
          if (tep_mesg == 'Woolworths' || tep_mesg == 'Woolworths 店') {
            isProcureShopW = true;
          }
          if (tep_mesg == 'Cool' || tep_mesg == 'Coles' || tep_mesg == 'Cool discount store') {
            isProcureShopC = true;
          }
          if (tep_mesg == '推荐' || tep_mesg == '门店' || tep_mesg == '商店' || tep_mesg == '商品列表') {
            isProcureShopC = true;
            isProcureShopW = true;
          }
          if (tep_mesg == '最终' || tep_mesg == '总结') {
            // isProcureShopDetected = false;
          }

          if (tep_mesg == "\u003cthink\u003e") {
            isThink.value = true;
          }
          if (isThink.value) {
            //清除think
            if (
              tep_mesg == "\u003cthink\u003e" ||
              tep_mesg == "\u003c/think\u003e"
            ) {
              chatHistory.value.at(idx).think += "";
            } else {
              chatHistory.value.at(idx).think += tep_mesg;
            }
            //如果结尾标签则停止拼接
            if (tep_mesg == "\u003c/think\u003e") {
              isThink.value = false;
            }
          } else {
            chatHistory.value.at(idx).text += tep_mesg;
          }
        } else {
          chatHistory.value.at(idx).text += tep_mesg;
        }
        autoScrollSwitch.value && scrollToBottom(true);
      }

      if (isBMIDetected) {
        chatHistory.value.at(idx).isNeedBmiForm = true;
      }
      if (procureStage.value==1) {
        chatHistory.value.at(idx).isNeedProcurePlanForm = true;
      }
      if (procureStage.value==2) {
        chatHistory.value.at(idx).isNeedProcureShopSelectForm = true;
      }


      chatStore.updateChat(agentInfo.value.id);

      sending.value = false;
      useEvent();
    })
    .catch((error) => {
      const generatedText = chatHistory.value.at(idx).text;
      thinkingDone();
      if (error.name === "AbortError") {
        chatHistory.value.at(idx).text +=
          generatedText + "[Request have been aborted.]";
      } else {
        chatHistory.value.at(idx).text = "[Request failed.]";
      }
      chatStore.updateChat(agentInfo.value.id);
      sending.value = false;
    });
}

function handleCopy(text) {
  navigator.clipboard.writeText(text);
  ElMessage({
    message: "Copied.",
    type: "success",
  });
}

function modifyPrompt() {
  if (prompt.value.key === "" || prompt.value.value === "") {
    ElMessage({
      message: "Please fill in the prompt.",
      type: "warning",
    });
    return;
  }
  agentInfo.value.prompt = prompt.value.key;
  agentStore.updateAgent(agentInfo.value);
  updateTemplateInfo(prompt.value);
  detailWindowVisible.value = false;
}

function clearChatHistory(e) {
  e();
  chatStore.removeChat(agentInfo.value.id);
  sending.value = false
  abort()
}

function inputKeyDown(e) {
  if (e.keyCode == 13) {
    e.preventDefault();
    if (input.value.indexOf('我想采购') >= 0
     || input.value.indexOf('我要采购') >= 0
     || input.value.indexOf('我需要采购') >= 0
     || input.value.indexOf('我计划采购') >= 0) {
      procureStage.value = 1;
    } else {
      procureStage.value = 0;
    }
    SubmitChat();
  }
}

function exportChatData() {
  const data = {};
  data.chatData = chatHistory.value;
  const blob = new Blob([JSON.stringify(data)], { type: "application/json" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.href = url;
  link.download = "chatHistory.json";
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

function importChatData(e) {
  const target = e.target;
  if (!target || !target.files) return;
  const file = target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = function (e) {
    const data = JSON.parse(e.target.result);
    data.chatData.forEach((item) =>
      chatStore.addChat(agentInfo.value.id, item)
    );
  };
  reader.readAsText(file);
  ElMessage({
    message: "Imported!",
    type: "success",
  });
}

function switchAutoScroll() {
  autoScrollSwitch.value = !autoScrollSwitch.value;
  siteStore.setAutoScroll(autoScrollSwitch.value);
}
//是否关闭思考按钮
function reflectionBtn() {
  isReflectionBtn.value = !isReflectionBtn.value;
}

//计算div宽度
function getDivWidth() {
  const el = document.getElementById("chat-area");
  chatAreaWidth.value = el.getBoundingClientRect().width - 78 + "px";
}
</script>

<template>
  <div class="view-container">
    <PromptInfo
      :condition="detailWindowVisible"
      :form="prompt"
      @submit="modifyPrompt"
      @close="detailWindowVisible = false"
    />
    <div v-if="!agentInfo" class="main-panel">
      <div class="empty-panel">
        <div class="empty-text">选择一个模型进行聊天</div>
        <RouterLink to="/aiSystem/agent">
          <SubmitBtn class="select-btn">选择</SubmitBtn>
        </RouterLink>
      </div>
    </div>
    <div v-else class="main-panel">
      <div class="left-panel" ref="leftPanel">
        <div class="left-panel-footer">
          <div class="ops">
            <el-popconfirm
              width="220"
              :hide-after="0"
              :hide-icon="true"
              title="确定清除记录?"
            >
              <template #reference>
                <div class="icon icon-clear icon-size2" title="清除聊天记录"></div>
              </template>
              <template #actions="{ cancel, confirm }">
                <el-button size="small" @click="cancel">否</el-button>
                <el-button
                  type="danger"
                  size="small"
                  @click="clearChatHistory(confirm)"
                  >是</el-button
                >
              </template>
            </el-popconfirm>
            <input
              ref="importInput"
              type="file"
              style="display: none"
              @change="importChatData"
            />
            <div
              class="icon icon-export icon-size2"
              title="导出聊天记录"
              @click="exportChatData"
            ></div>
            <div
              class="icon icon-import icon-size2"
              title="导入聊天记录"
              @click="importInput.click()"
            ></div>
            <div
              class="icon icon-scroll icon-size2"
              title="自动滚动"
              @click="switchAutoScroll"
              :class="{ red: autoScrollSwitch }"
            ></div>
          </div>
        </div>
      </div>
      <!-- 主窗口 -->
      <div class="chat-panel">
        <div class="model-info">
           {{ agentInfo.model }}
          </div>
        <div ref="scrollRef" class="chat-area" id="chat-area">
          <div
            v-for="(v, i) in chatHistory"
            :key="i"
            :class="v.inversion ? 'bot-msg' : 'user-msg'"
          >
            <div class="msg-info">
              <!-- <div class="timestamp">{{ moment(v.timestamp).format('YYYY-MM-DD HH:mm:ss') }}</div> -->
            </div>
            <div class="msg-container">
              <div class="icon-position">
                <img
                  class="icon icon-size icon-top"
                  v-if="v.inversion"
                  src="@/assets/aiSystem/ai.png"
                  alt=""
                />
              </div>
              <div v-if="v.inversion" class="msg-content">
                <div>
                  <el-button
                    class="reflectionBtn"
                    type="primary"
                    v-if="v.think && v.think.length > 2"
                    @click="reflectionBtn"
                    :icon="isReflectionBtn ? 'ArrowUp' : 'ArrowDown'"
                    >深度思考</el-button
                  >
                </div>
                <div class="think" v-if="v.think.length > 2 && isReflectionBtn">
                  {{ v.think }}
                </div>
                <div v-html="render(v.text)"></div>
                <!-- <div v-html="render(v.chatForm)" class="chatForm" :id="v.idx"></div> -->
                <div v-if="v.isNeedBmiForm">
                    <div className="bmi-form">
                      <label>请输入您的身高和体重信息：</label>
                      身高:<input v-model="bmiHeight" type="number" placeholder="身高 (cm)" value="175"/>
                      体重:<input v-model="bmiWeight" type="number" placeholder="体重 (kg)" value="65"/>
                      <button @click="bmiSubmit">提交</button>
                    </div>
                </div>
                <div v-if="v.isNeedProcurePlanForm">
                  <el-card>
                    <el-form className="procurePlan-form">
                      <el-form-item label='请输入采购信息：'></el-form-item>
                      <el-form-item v-for="item in procureProducts" :key="item.id">
                        产品名称:<el-input v-model="item.name" placeholder="例如: 便利贴，记号笔" class="chat-form-input"></el-input> &nbsp;
                        预估单价($):<el-input type="number" v-model="item.price" placeholder="预估单价" class="chat-form-input"></el-input> &nbsp;
                        数量:<el-input type="number" v-model="item.num" placeholder="数量" class="chat-form-input"></el-input> &nbsp;
                        <el-button @click="removeProcureProduct(item.id)">-</el-button>
                      </el-form-item>
                      <el-form-item>
                        <el-button @click="addProcureProduct">+</el-button>
                        <el-button @click="procurePlanSubmit" type="primary">提交</el-button>
                      </el-form-item>
                    </el-form>
                  </el-card>
                </div>

                <div v-if="v.isNeedProcureShopSelectForm">
                  <el-card>
                    <el-form className="procureShopSelect-form">
                      <el-form-item label='请选择采购超市：'></el-form-item>
                      <el-form-item v-for="item in procureShopList" :key="item.name">
                        <el-radio name="procureShopSelect" placeholder="" v-model="procureShopSelected" :value="item.name" :id="'shop-'+item.name" border class="chat-form-radio" > {{ item.name }} </el-radio> &nbsp; 预估 ${{ item.estimatedPrice }}
                      </el-form-item>
                      <el-form-item>
                        <el-button @click="procureShopSubmit" type="primary">提交</el-button>
                      </el-form-item>
                    </el-form>
                  </el-card>
                </div>
                <p>
                  <label>{{ formInputText }}</label>
                </p>
              </div>
              <div v-else class="msg-content">{{ v.text }}</div>
            </div>
          </div>
        </div>
        <div class="chat-input">
          <div class="chat-input-top">
            <el-button type="primary" class="reflectionBtn" plain
              >深度思考</el-button
            >
            <!-- <el-button type="primary" class="reflectionBtn" plain
              >联网查询</el-button
            > -->
            <el-button v-if="procureStage > 0" type="primary" class="reflectionBtn" plain
              >采购流程Stage: {{ procureStage }}</el-button
            >
          </div>
          <el-input
            placeholder="AI智能办公"
            @keydown="inputKeyDown($event)"
            v-model="input"
            clearable
            class="chat-inputBox"
          >
            <template #suffix>
              <div v-if="sending" class="chat-icon-box" @click="abort">
                <!-- api输出中，不可点击提交 -->
                <img
                  class="chat-icon chat-stop chat-stop-animation"
                  :src="stop"
                  alt=""
                />
              </div>
              <div
                v-else
                class="chat-icon-box"
                @click="SubmitChat"
                :style="input ? 'background:#615ced' : 'background:#d6d5de'"
              >
                <!-- api不在输出中，可以点击提交 -->
                <el-icon
                  class="chat-icon"
                  size="20"
                  style="margin: 2px 2px 0 0"
                  color="#fff"
                  ><Position
                /></el-icon>
              </div>
            </template>
            <template #prefix>
              <el-icon class="chat-icon" size="20" color="#353535"
                ><Link
              /></el-icon>
            </template>
          </el-input>
          <div class="chat-input-bottom">
            服务生成的所有内容均由人工智能模型生成，其生成内容的准确性和完整性无法保证，不代表我们的态度或观点
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.view-container {
  height: 100%;
  display: flex;
  flex-direction: column;
}

.header-panel {
  .title {
    font-size: 1.5rem;
    font-weight: 600;
    line-height: 2rem;
    color: var(--text-03);
  }

  .agent-id {
    font-size: 0.8rem;
    font-weight: 400;
    line-height: 1rem;
    color: #999;
  }
}

.main-panel {
  flex: 1;
  display: flex;
  overflow-y: auto;
  background-color: #f6f7fb;

  .empty-panel {
    flex: 1;
    display: flex;
    justify-content: center;
    align-items: center;
    flex-direction: column;
    gap: 10px;

    .select-btn {
      display: block;
    }
  }
}

.left-panel {
  width: 200px;
  padding: 10px 0 10px 10px;
  position: sticky;
  overflow: auto;
  display: flex;
  flex-direction:column-reverse;
  background-color: #ffffff;
  z-index: 10;

  .left-panel-footer {
    .ops {
      display: flex;
      justify-content: flex-start;
      gap: 10px;
    }
  }
}

.chat-panel {
  flex: 1;
  max-height: 100%;
  display: flex;
  flex-direction: column;
  position: relative;
  align-items: center;
  .model-info{
    position: absolute;
    left: 20px;
    top:20px;
    font-size:20px;
  }

  .chat-input {
    position: relative;
    padding: 0;
    width: 70%;
    min-height: 150px;
    display: flex;
    flex-direction: column;
    .chat-input-top {
      height: 50px;
      display: flex;
      align-items: center;
    }
    .chat-inputBox {
      width: 100%;
      height: 50px;
      border-radius: 35px;
    }

    .chat-icon {
      margin: 0 10px;
      cursor: pointer;
    }
    .chat-stop {
      width: 100%;
      height: 100%;
    }
    .chat-icon-box {
      display: flex;
      align-items: center;
      justify-content: center;
      width: 35px;
      height: 35px;
      border-radius: 35px;
      cursor: pointer;
    }

    .chat-input-bottom {
      height: 50px;
      display: flex;
      align-items: center;
      justify-content: center;
      color: #8d8fa6;
      font-size: 12px;
      user-select:none;
    }
    .edit-input {
      max-height: 100%;
      background: #f6f7fb;
      color: var(--title-tc-01);
      outline: none;
      border: none;
      resize: none;
      caret-color: #000;
      padding: 5px;
      font-size: 16px;
      line-height: 1.6;
    }

    .submit-btn {
      position: absolute;
      right: 10px;
      bottom: 10px;
    }

    .stop-btn {
      position: absolute;
      right: 50%;
      bottom: 120%;
      transform: translateX(50%);
    }
  }
}

.chat-area {
  flex: 1;
  padding: 0.5em;
  overflow-y: auto;
  padding-bottom: 4em;
  width: 70%;
  -ms-overflow-style: none; /* IE 和 Edge */
  scrollbar-width: none; /* Firefox */

  .user-msg,
  .bot-msg {
    width: 100%;
    margin-bottom: 1em;
  }

  .user-msg {
    .msg-info {
      flex-direction: row-reverse;
    }

    .msg-container {
      flex-direction: row-reverse;
    }

    .msg-content {
      white-space: break-spaces;
      background-color: #e0dfff !important;
      color: #2c2c36;
      line-height: 27px;
      max-width: v-bind("chatAreaWidth");
    }
  }

  .msg-info {
    display: flex;
    align-items: center;

    .timestamp {
      font-size: 10px;
      color: #999;
    }
  }

  .msg-container {
    display: flex;
    margin: 0 0px;

    .fast-op {
      display: flex;
      flex-direction: column;
      justify-content: space-between;
      align-items: center;
      opacity: 0;
      transition: opacity 0.3s ease-out;

      &:hover {
        opacity: 1;
      }
    }

    .msg-content {
      // background-color: var(--chat-bot-msg-bgc);
      background-color: rgb(255, 255, 255);
      box-shadow: 0px 1px 4px rgba(65, 65, 65, 0.339);
      padding: 0.6em 1.4em;
      border-radius: 5px;
    }

    .msg-operation {
      margin-top: auto;
    }

    .dropdown-item {
      transition: background-color 0.1s ease-out;
      padding: 5px 10px;
      cursor: pointer;

      &:hover {
        background-color: #84c99b;
      }
    }
  }
}

.icon-cat {
  margin: 5px;
  mask-image: url("@/assets/svg/cat1.svg");
}
.icon-size {
  width: 40px;
  height: 40px;
}
.icon-size2{
  width: 20px;
  height: 20px;
}
.icon-robot {
  margin: 5px;
  mask-image: url("@/assets/aiSystem/ai.png");
}

.icon-pen {
  mask-image: url("@/assets/svg/pen.svg");
}

.icon-refresh {
  width: 10px;
  height: 10px;
  margin: 0 5px;
  mask-image: url("@/assets/svg/refresh.svg");
}

.icon-clear {
  margin: 0;
  mask-image: url("@/assets/svg/clear.svg");
  background-color: #3d5fea;
}

.icon-export {
  margin: 0;
  mask-image: url("@/assets/svg/export.svg");
  background-color: #3d5fea;
}

.icon-import {
  margin: 0;
  mask-image: url("@/assets/svg/import.svg");
  background-color: #3d5fea;
}

.icon-scroll {
  margin: 0;
  mask-image: url("@/assets/svg/scroll.svg");
  background-color: #3d5fea;
}

.icon-arrow {
  margin: 0;
  mask-image: url("@/assets/svg/arrow.svg");
}

.red {
  background-color: red !important;
  color: red !important;
}

.icon-pen,
.icon-refresh,
.icon-clear,
.icon-export,
.icon-import,
.icon-scroll,
.icon-arrow {
  cursor: pointer;

  &:hover {
    background-color: rgb(57, 58, 57);
  }
}
.icon-position {
  height: 100%;
}
.think {
  color: #8b8b8b;
  padding: 0 0 20px 0;
  box-sizing: border-box;
  border-radius: 10px;
  font-size: 14px;
  line-height: 30px;
}
.reflectionBtn {
  min-width: 80px;
  height: 30px;
  font-size: 12px;
  margin-bottom: 10px;
}

/*搜索组件最外层div */

/*搜索input框 */
:deep(.el-input__wrapper) {
  background-color: #fff; /*覆盖原背景颜色，设置成透明 */
  // border-radius: 95px;
}
/*搜索button按钮 */
:deep(.el-input-group__append) {
  background: rgb(0 234 245 / 48%);
  border-radius: 95px;
}
/* 隐藏 Chrome、Safari 和 Opera 的滚动条 */
.chat-area::-webkit-scrollbar {
  // display: none;
}

.chat-form-input {
  border-radius: 1px;
  width: 150px;
}

.el-form-item {
  // margin-top: 5px;
  margin-bottom: 5px;
}

.el-card {
  margin-top: 5px;
  // padding: 5px; // 不生效
}

.chat-form-radio {
  width: 200px;
}

</style>
