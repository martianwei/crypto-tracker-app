<template>
  <div class="track-view">
    <svg class="box" :style="{ height: boxHeight + 'px', width: boxWidth + 'px' }" v-if="jsonData">
      <!-- 添加时间轴箭头定义 -->
      <defs>
        <marker id="arrowhead" viewBox="0 0 10 10" refX="5" refY="5" markerWidth="8" markerHeight="8"
          orient="auto-start-reverse">
          <path d="M 0 0 L 10 5 L 0 10 z" />
        </marker>
      </defs>
      <!-- 添加时间轴 -->
      <line class="timeline" :x1="500" :y1="30" :x2="1800" :y2="30" />
      <!-- 添加时间轴上的刻度 -->
      <g v-for="index in 10" :key="index">
        <!-- <text class="timeline-text" :x="500 + index * 130" :y="20">{{ minTimestamp + index * 86400 }}</text> -->
        <line class="timeline-item" :x1="500 + index * 30" :y1="15" :x2="500 + index * 30" :y2="45" />
      </g>
      <!-- 其他元素保持不变 -->
      <g v-for="(address, index) in addresses" :key="index">
        <text class="address-text" :x="0" :y="50 + index * 200">{{ address }}</text>
        <line class="address-timeline" :x1="500" :y1="50 + index * 200" :x2="1800" :y2="50 + index * 200" />
      </g>
      <g v-for="(txn) in transactions" :key="txn.id">
        <text class="txn-text" :x="500 + calculateX(txn.timestamp)" :y="50 + (txn.sender + txn.receiver - 2) * 100">
          時間: {{ txn.date }} 價格: {{ txn.amount }} {{ txn.symbol }}
        </text>
        <line class="txn-line" :x1="500 + calculateX(txn.timestamp)" :y1="50 + (txn.sender - 1) * 200"
          :x2="500 + calculateX(txn.timestamp)" :y2="50 + (txn.receiver - 1) * 200" marker-end="url(#arrowhead)" />
      </g>
    </svg>
  </div>
</template>


<script>
import axios from 'axios';

export default {
  name: 'TrackView',
  data() {
    return {
      jsonData: null,
      addresses: [],
      transactions: [],
      minTimestamp: Number.MAX_VALUE, // 用于记录最小的时间戳
      maxTimestamp: Number.MIN_VALUE, // 用于记录最大的时间戳
    };
  },
  computed: {
    boxHeight() {
      return this.addresses.length * 200; // 根据地址数量计算高度，每个地址占据200px，加上额外的100px
    },
    boxWidth() {
      // 根据最小和最大时间戳计算交易时间范围并转换为小时数
      const rangeHours = (this.maxTimestamp - this.minTimestamp) / 3600;
      // 根据每小时像素宽度计算 boxWidth
      const hourlyWidth = 40; // 每小时的像素宽度，根据需要调整
      const boxWidth = rangeHours * hourlyWidth + 1000;

      return boxWidth;
    }
  },
  async created() {
    await this.fetchData();
    console.log('jsonData', this.jsonData);
    this.processData();
  },
  methods: {
    async fetchData() {
      try {
        const response = await axios.get('file.json');
        this.jsonData = response.data;
      } catch (error) {
        console.error('Error fetching data:', error);
      }
    },
    formatTimestamp(timestamp) {
      // 实现您所需的时间格式化逻辑，这里只是一个示例
      const date = new Date(timestamp * 1000);
      return date.toLocaleString();
    },
    processData() {
      const addressesMap = new Map(); // 使用 Map 存储地址和 ID 的映射关系
      const transactions = [];

      if (this.jsonData) {
        this.jsonData.data.inbound.forEach((item) => {
          const senderAddress = item.sender.address;
          const receiverAddress = item.receiver.address;
          const timestamp = item.timestamp;
          this.minTimestamp = Math.min(this.minTimestamp, timestamp); // 更新最小的时间戳
          this.maxTimestamp = Math.max(this.maxTimestamp, timestamp); // 更新最大的时间戳

          // 添加地址到 Map，并为每个地址分配一个唯一的 ID
          if (!addressesMap.has(senderAddress)) {
            const senderId = addressesMap.size + 1; // 分配 ID
            addressesMap.set(senderAddress, senderId);
          }
          if (!addressesMap.has(receiverAddress)) {
            const receiverId = addressesMap.size + 1; // 分配 ID
            addressesMap.set(receiverAddress, receiverId);
          }

          const transaction = {
            date: this.formatTimestamp(item.timestamp),
            timestamp: item.timestamp,
            amount: item.amount,
            symbol: item.currency.symbol,
            sender: addressesMap.get(senderAddress), // 使用地址对应的 ID
            receiver: addressesMap.get(receiverAddress), // 使用地址对应的 ID
          };
          transactions.push(transaction);
        });

        this.jsonData.data.outbound.forEach((item) => {
          const senderAddress = item.sender.address;
          const receiverAddress = item.receiver.address;
          const timestamp = item.timestamp;
          this.minTimestamp = Math.min(this.minTimestamp, timestamp); // 更新最小的时间戳
          this.maxTimestamp = Math.max(this.maxTimestamp, timestamp); // 更新最大的时间戳

          // 添加地址到 Map，并为每个地址分配一个唯一的 ID
          if (!addressesMap.has(senderAddress)) {
            const senderId = addressesMap.size + 1; // 分配 ID
            addressesMap.set(senderAddress, senderId);
          }
          if (!addressesMap.has(receiverAddress)) {
            const receiverId = addressesMap.size + 1; // 分配 ID
            addressesMap.set(receiverAddress, receiverId);
          }

          const transaction = {
            date: this.formatTimestamp(item.timestamp),
            timestamp: item.timestamp,
            amount: item.amount,
            symbol: item.currency.symbol,
            sender: addressesMap.get(senderAddress), // 使用地址对应的 ID
            receiver: addressesMap.get(receiverAddress), // 使用地址对应的 ID
          };
          transactions.push(transaction);
        });
      }

      this.addresses = Array.from(addressesMap.keys()).sort((a, b) => addressesMap.get(a) - addressesMap.get(b)); // 按照 ID 的顺序排序地址数组
      this.transactions = transactions;
    },
    calculateX(timestamp) {
      const hours = (timestamp - this.minTimestamp) / 3600; // 将时间差转换为小时数
      const x = hours * 40; // 根据实际需求计算 x 坐标值的偏移量
      return x;
    }
  }
};
</script>

<style>
/* Set the size of the SVG container */
.track-view {
  display: flex;
  height: auto;
  justify-content: center;
  align-items: center;
}

/* Style the box SVG element */
.box {
  display: flex;
  flex-direction: column;
  outline: auto;
  padding: 10px;
  box-sizing: border-box;
}

/* Style the text */
.address-text {
  font-size: 12px;
  font-weight: bold;
}

.txn-text {
  font-size: 12px;
}

.timeline {
  stroke: #000;
  stroke-width: 2;
}

.address-timeline {
  stroke: #ff0000;
  stroke-width: 5px;
}

.timeline-item {
  stroke: #00ff95;
  stroke-width: 1;
}

.txn-line {
  stroke: #0000ff;
  stroke-width: 1px;
}
</style>



