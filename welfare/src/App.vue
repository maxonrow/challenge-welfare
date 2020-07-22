<template>
  <v-app>
    <v-card>
      <v-tabs fixed-tabs v-model="tab">
        <v-tab v-for="item in items" :key="item.tab">
          {{ item.tab }}
        </v-tab>
      </v-tabs>

      <v-tabs-items v-model="tab">
        <v-tab-item v-for="item in items" :key="item.tab">
          <!-- <v-card flat>
          <v-card-text>{{ item.content }}</v-card-text>
        </v-card> -->
          <div v-if="item.tab == 'Create'">
            <v-card class="mx-10">
              <v-row class="pa-5 d-flex justify-center">
                <v-col>
                  Create Token
                </v-col>
                <v-col cols="12"
                  ><v-text-field v-model="name" type="text" placeholder="name"
                /></v-col>
                <v-col cols="12"
                  ><v-text-field
                    v-model="symbol"
                    type="text"
                    placeholder="symbol"
                /></v-col>
                <v-col cols="12"
                  ><v-text-field
                    v-model="maxSupply"
                    type="text"
                    placeholder="maximum Supply"
                /></v-col>
                <v-col cols="12"
                  ><v-text-field
                    v-model="metadata"
                    type="text"
                    placeholder="metadata"
                /></v-col>
                <v-btn @click="create()">Create</v-btn>
              </v-row>
            </v-card>
          </div>
          <div v-if="item.tab == 'Transfer'">
            <v-card class="mx-10">
              <v-row class="pa-5 d-flex justify-center">
                <v-col>
                  Transfer Token
                </v-col>
                <v-col cols="12"
                  ><v-text-field
                    v-model="address"
                    type="text"
                    placeholder="wallet address"
                /></v-col>
                <v-col cols="12"
                  ><v-text-field v-model="value" type="text" placeholder="value"
                /></v-col>
                <v-btn @click="transfer()">Transfer</v-btn>
              </v-row>
            </v-card>
          </div>
          <div v-if="item.tab == 'Search'">
            <v-card class="mx-10">
              <v-row class="pa-5 d-flex justify-center">
                <v-col>
                  Check Transaction
                </v-col>
                <v-col cols="12"
                  ><v-text-field v-model="hash" type="text" placeholder="Hash"
                /></v-col>
                <v-btn @click="search">Check</v-btn>
              </v-row>
            </v-card>
          </div>
        </v-tab-item>
      </v-tabs-items>
    </v-card>
    <v-card>
      <!-- <div>Hash: {{ hash }}</div>
      <div>Receipt</div>
      <pre>{{receipt}}</pre> -->
      <pre class="pa-5">{{ notification }}</pre>
    </v-card>
    <v-overlay opacity="0.9" v-if="loading">
      {{ msg }}
    </v-overlay>
  </v-app>
</template>

<script>
import { mxw, token } from "mxw-sdk-js";
import { bigNumberify } from "mxw-sdk-js/dist/utils";
import { FungibleTokenActions } from "mxw-sdk-js/dist/token";

export default {
  name: "App",
  async created() {
    this.loading = true;
    await this.init();
    this.providerConnection.getBlockNumber().then((blockNumber) => {
      console.log("Latest block number: " + blockNumber);
      this.loading = false;
    });
  },
  data: () => ({
    loading: false,
    msg: "",
    notification: "Hello",
    balance: 0,
    address: "",
    value: "",
    silentRpc: "",
    providerConnection: mxw.providers.Provider,
    provider: mxw.Wallet,
    issuer: mxw.Wallet,
    middleware: mxw.Wallet,
    token: token.FungibleToken,
    tab: null,
    name: "",
    symbol: "",
    maxSupply: "",
    metadata: "",
    hash: "",
    receipt: {},
    items: [
      { tab: "Create", content: "test" },
      { tab: "Transfer", content: "test" },
      { tab: "Search", content: "test" },
    ],
  }),
  methods: {
    init() {
      this.msg = "Initializing blockchain...";
      let connection = {
        url: "http://localhost:26657",
        timeout: 60000,
      };
      this.providerConnection = new mxw.providers.JsonRpcProvider(connection, {
        chainId: "maxonrow-chain",
        name: "mxw",
      });
      const providerMnemonic =
        "naive hire arctic injury camp twelve actor valid process voice return unusual glad hen ginger brisk clever solve toss expire type road blood green";
      this.provider = mxw.Wallet.fromMnemonic(providerMnemonic).connect(
        this.providerConnection
      );

      const issuerMnemonic =
        "wreck fiber slice novel nurse guess plate oven cotton life thought tape addict thank frown ready rival walk dish short solution work arena nurse";
      this.issuer = mxw.Wallet.fromMnemonic(issuerMnemonic).connect(
        this.providerConnection
      );

      const middlewareMnemonic =
        "police toilet cupboard song blanket duty wrestle public bike cattle install page option spell scout crop pig answer access alarm gain fish absent pen";
      this.middleware = mxw.Wallet.fromMnemonic(middlewareMnemonic).connect(
        this.providerConnection
      );
    },
    create() {
      this.loading = true;
      this.msg = "Generating Token...";
      let fungibleTokenProperties = null;
      fungibleTokenProperties = {
        name: this.name,
        symbol: this.symbol,
        decimals: 18,
        fixedSupply: true,
        maxSupply: bigNumberify("10000000000000"),
        fee: {
          to: "mxw173qf9y2ae0cx8y07ez6qsl9k2gs2l5955hfc7x",
          value: bigNumberify("0"),
        },
        metadata: "",
      };
      let burnable = true;
      let overrides = {
        tokenFees: [
          { action: FungibleTokenActions.transfer, feeName: "default" },
          {
            action: FungibleTokenActions.transferOwnership,
            feeName: "default",
          },
          { action: FungibleTokenActions.acceptOwnership, feeName: "default" },
        ],
        burnable,
      };
      if (burnable) {
        overrides.tokenFees.push({
          action: FungibleTokenActions.burn,
          feeName: "transfer",
        });
      }
      token.FungibleToken.create(fungibleTokenProperties, this.issuer).then(
        (fToken) => {
          console.log("Token had been created");
          return this.authourize(
            token.FungibleToken.approveFungibleToken,
            fToken.symbol,
            overrides
          );
        }
      );
    },
    authourize(perform, symbol, overrides) {
      this.msg = "Authorizing Token...";
      return perform(symbol, this.provider, overrides)
        .then((transaction) => {
          return token.FungibleToken.signFungibleTokenStatusTransaction(
            transaction,
            this.issuer
          );
        })
        .then((transaction) => {
          return token.FungibleToken.sendFungibleTokenStatusTransaction(
            transaction,
            this.middleware
          );
        })
        .then((receipt) => {
          if (receipt.status == 1) console.log("success");
          else console.log("error");
          return this.Query(symbol);
        });
    },
    Query(symbol) {
      this.msg = "Updating Token...";
      return token.FungibleToken.fromSymbol(symbol, this.issuer).then(
        (fToken) => {
          console.log("Updated");
          this.loading = false;
          this.notification = "Token "+symbol+" created"
          return (this.token = fToken);
        }
      );
    },
    getBalance(token) {
      return token.getBalance().then((balance) => {
        console.log(
          "Total " +
            token.state.name +
            " Token Owned: " +
            mxw.utils.formatUnits(balance, token.state.decimals)
        );
        return balance;
      });
    },
    transfer() {
      this.loading = true;
      this.msg = "Transfering Token...";
      console.log(this.token);
      this.token.transfer(this.address, this.value).then((receipt) => {
        this.loading = false;
        if (receipt.status == 1) {
          this.notification = "Transfered token to: "+ this.address;
          this.hash = receipt.hash;
          }
        else {
          this.notification = "error";
          this.hash = receipt.hash;
          }
        // if (receipt.status == 1) return (this.hash = receipt.hash);
        // else return (this.hash = "error");
      });
    },
    search() {
      this.providerConnection.getTransactionReceipt(this.hash).then((receipt)=>{
        this.notification = receipt.payload.value.msg;
        return (this.receipt = receipt);
      })
    }
  },
};
</script>
