<template>
  <div>
    <div>To test with local EVM, see README</div>
    <div>1. install metamask and switch to Rinkeby network</div>
    <div>2. click button to connect wallet</div>
    <div>
      3. click button to init contract(after reload page, must do this step
      first)
    </div>
    <div>4. click button to PlaceOrder</div>
    <div>5. click button to AddDelivery</div>
    <div>6. click button to CompleteOrder</div>
    <div>...</div>
    <v-btn @click="connectWallet">connect to metamask</v-btn>
    <div>
      chainId: {{ chainId }}
      <br />
      accounts:{{ accounts }}
    </div>
    <v-btn @click="initContracts">InitContracts</v-btn>
    <div>----------</div>
    <ul>
      <li v-for="(address, name, index) in contracts" :key="index">
        {{ name }}:{{ address }}
      </li>
    </ul>
    <v-btn @click="PlaceOrder">PlaceOrder</v-btn>
    <v-btn @click="AddDelivery">AddDelivery</v-btn>
    <v-btn @click="CompleteOrder">CompleteOrder</v-btn>
    <ul>
      <li>orderId: <input v-model="orderId" /></li>
      <li>status: {{ orderStatus }}</li>
      <li>derivativeContract:<input v-model="derivativeContract" /></li>
      <li>derivativeTokenId: <input v-model="derivativeTokenId" /></li>
      <li>licenseId: <input v-model="licenseId" /></li>
    </ul>
    <v-btn @click="derivativeInfo">derivative token detail</v-btn>
    <ul>
      <li v-for="(value, key, index) in derivativeDetial" :key="index">
        {{ key }}:{{ value }}
      </li>
    </ul>
    <v-btn @click="orderInfo">orderDetail</v-btn>
    <ul>
      <li v-for="(value, key, index) in orderDetail" :key="index">
        {{ key }}:{{ key == "licenseId" ? value.toHexString() : value }}
      </li>
    </ul>
    <v-btn @click="licenseInfo">licenseDetail</v-btn>
    <ul>
      <li v-for="(value, key, index) in licenseDetail" :key="index">
        {{ key }}:{{ value }}
      </li>
    </ul>
    <v-btn @click="getAllIP">get IP list</v-btn>
    <ul>
      <li v-for="(item, index) in ipList" :key="index">
        token: {{ item.token }} tokenId: {{ item.tokenId }}
      </li>
    </ul>
    <v-btn @click="getAllOrders">get orders list</v-btn>
    <ul>
      <li v-for="(order, index) in orderList" :key="index">
        <span>----------------orderId:{{ index }}------------------</span>
        <ul>
          <li v-for="(value, key, index) in order" :key="index">
            {{ key }}:{{ key == "licenseId" ? value.toHexString() : value }}
          </li>
        </ul>
      </li>
    </ul>

    <v-btn @click="getAllServices">get services list</v-btn>
    <ul>
      <li v-for="(service, index) in servicesList" :key="index">
        <span>----------------serviceId:{{ index }}------------------</span>
        <ul>
          <li v-for="(value, key, index) in service" :key="index">
            {{ key }}:{{ value }}
          </li>
        </ul>
      </li>
    </ul>
  </div>
</template>

<script>
import { connect } from "./lib/metamask";
import { InitContracts, AttachDerivative } from "./lib/contracts";

const ArrayToObj = (arrayObj) => {
  window.x = arrayObj;
  return Object.keys(arrayObj)
    .filter((key) => isNaN(Number(key)))
    .reduce((obj, key) => ((obj[key] = arrayObj[key]), obj), {});
};
let contracts = {};
export default {
  name: "App",
  data() {
    return {
      chainId: "",
      accounts: [],
      contracts: {},
      orderId: "",
      derivativeContract: "",
      derivativeTokenId: "",
      licenseId: "",
      orderStatus: "",
      derivativeDetial: {},
      orderDetail: {},
      licenseDetail: {},
      ipList: [],
      orderList: [],
      servicesList: [],
    };
  },
  methods: {
    async connectWallet() {
      const { chainId, accounts } = await connect();
      this.chainId = Number(chainId);
      this.accounts = accounts;
    },
    initContracts() {
      contracts = InitContracts(this.chainId);
      Object.keys(contracts).forEach((name) => {
        console.log(contracts[name].address);
        this.contracts[name] = contracts[name].address;
      });
    },
    async PlaceOrder() {
      const DerivativeFactory = contracts.DerivativeFactory;
      const originalNFT = contracts.MockNFT.address;
      const originalTokenId = 1234; // this token already staked in the IP pool
      const serviceId = 0; // this service already registered
      const tx = await DerivativeFactory.place_order(
        originalNFT,
        originalTokenId,
        serviceId
      ); // wait tx send to network
      const receipt = await tx.wait(1); // get receipt of tx, this mean tx is included in a block.
      // parse log and find PlaceOrder to get orderId;
      const filterTopic = DerivativeFactory.filters.PlaceOrder().topics[0];
      const log = receipt.logs.find((log) => log.topics[0] == filterTopic);
      const orderEvent = DerivativeFactory.interface.parseLog({
        data: log.data,
        topics: log.topics,
      });
      console.log("orderEvent args:", orderEvent.args);
      this.orderId = orderEvent.args.orderId;
      this.orderStatus = "Pending";
    },
    async AddDelivery() {
      const orderId = this.orderId;
      const DerivativeFactory = contracts.DerivativeFactory;
      const tokenURI = "https://ipfs/xxxx";
      const tx = await DerivativeFactory.add_delivery(orderId, tokenURI); // wait tx send to network
      const receipt = await tx.wait(1); // get receipt of tx, this mean tx is included in a block.
      // parse log and find PlaceOrder to get orderId;
      const filterTopic = DerivativeFactory.filters.AddDelivery().topics[0];
      const log = receipt.logs.find((log) => log.topics[0] == filterTopic);
      const deliveryEvent = DerivativeFactory.interface.parseLog({
        data: log.data,
        topics: log.topics,
      });
      console.log("deliveryEvent args:", deliveryEvent.args);
      this.derivativeContract = deliveryEvent.args.derivativeContract;
      this.derivativeTokenId = deliveryEvent.args.derivativeTokenId;
      this.orderStatus = "Deliveried";
    },
    async CompleteOrder() {
      const orderId = this.orderId;
      const DerivativeFactory = contracts.DerivativeFactory;
      const tx = await DerivativeFactory.complete_order(orderId); // wait tx send to network
      const receipt = await tx.wait(1); // get receipt of tx, this mean tx is included in a block.
      // parse log and find PlaceOrder to get orderId;
      const filterTopic = DerivativeFactory.filters.CompleteOrder().topics[0];
      const log = receipt.logs.find((log) => log.topics[0] == filterTopic);
      const completeEvent = DerivativeFactory.interface.parseLog({
        data: log.data,
        topics: log.topics,
      });
      console.log("completeEvent args:", completeEvent.args);
      this.licenseId = completeEvent.args.licenseId.toHexString();
      this.orderStatus = "Completed";
    },

    async derivativeInfo() {
      const Derivative = AttachDerivative(this.derivativeContract);
      const name = await Derivative.name();
      const owner = await Derivative.ownerOf(this.derivativeTokenId);
      const tokenURI = await Derivative.tokenURI(this.derivativeTokenId);
      this.derivativeDetial = {
        name,
        derivativeContract: this.derivativeContract,
        tokenId: this.derivativeTokenId,
        owner,
        tokenURI,
      };
    },
    async orderInfo() {
      const DerivativeFactory = contracts.DerivativeFactory;
      this.orderDetail = await DerivativeFactory.orders(this.orderId).then(
        ArrayToObj
      );
    },
    async licenseInfo() {
      const Licenser = contracts.Licenser;
      this.licenseDetail = await Licenser.get(this.licenseId).then(ArrayToObj);
    },
    async getAllIP() {
      const IPPool = contracts.IPPool;
      this.ipList = await IPPool.get_items();
    },
    async getAllOrders() {
      const DerivativeFactory = contracts.DerivativeFactory;
      this.orderList = await DerivativeFactory.get_orders().then((orders) =>
        orders.map(ArrayToObj)
      );
    },
    async getAllServices() {
      const DerivativeFactory = contracts.DerivativeFactory;
      this.servicesList = await DerivativeFactory.list_service().then(
        (services) => services.map(ArrayToObj)
      );
    },
  },
};
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  color: #2c3e50;
  margin-top: 60px;
}
</style>
