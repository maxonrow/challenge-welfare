<template>
  <v-app>
    <v-card>
      <v-tabs fixed-tabs v-model="tab">
        <v-tab v-for="item in items" :key="item.tab">
          {{ item.tab }}
        </v-tab>
      </v-tabs>

      <v-tabs-items style="height:100%" v-model="tab">
        <v-tab-item v-for="item in items" :key="item.tab">
          <div v-if="item.tab == 'Create'">
            <div>
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
              <div class="pt-5">
                <v-card dark class="pa-5 mx-10 mb-5">
                  <div v-html="notification.create"></div>
                </v-card>
              </div>
            </div>
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
            <div class="pt-5">
              <v-card dark class="pa-5 mx-10 mb-5">
                <div v-html="notification.transfer"></div>
              </v-card>
            </div>
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
            <div class="pt-5">
              <v-card dark class="pa-5 mx-10">
                <div>Transaction Type: <span style="color:yellow">{{notification.search.type}}</span></div>
                <div>Token: <span style="color:yellow">{{notification.search.token}}</span></div>
                <div>From: <span style="color:yellow">{{notification.search.from}}</span></div>
                <div>To: <span style="color:yellow">{{notification.search.to}}</span></div>
                <div>Value: <span style="color:yellow">{{notification.search.value}}</span></div>
              </v-card>
            </div>
          </div>
        </v-tab-item>
      </v-tabs-items>
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
    notification: {
      create: "Hello",
      transfer: "Please create a Token first",
      search: {
        type: "",
        value: "",
        from: "",
        to: "",
        token: "",
      },
    },
    msg: "",
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
        })
        .catch(error => {
          this.loading = false;
          this.notification.create = "<span style='color:red'>"+error.info.message+"</span>";
        })
        ;
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
        })
        .catch(error => {
          this.loading = false;
          this.notification.transfer = "<span style='color:red'>"+error+"</span>";
        });
    },
    Query(symbol) {
      this.msg = "Updating Token...";
      return token.FungibleToken.fromSymbol(symbol, this.issuer).then(
        (fToken) => {
          console.log("Updated");
          this.loading = false;
          this.notification.create =
            "Token <span style='color:green'>" + symbol + "</span> created";
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
          this.notification.transfer =
            "Transfered token <span style='color:green'>" +
            this.symbol +
            "</span> to: <span style='color:yellow'>" +
            this.address +
            "</span>";
          this.hash = receipt.hash;
        } else {
          this.notification.transfer = "error";
          this.hash = receipt.hash;
        }
      })
      .catch(error => {
        this.loading = false;
        this.notification.transfer = error;
      });
    },
    search() {
      this.providerConnection
        .getTransactionReceipt(this.hash)
        .then((receipt) => {
          this.notification.search.type = receipt.payload.value.msg[0].type;
          this.notification.search.value = receipt.payload.value.msg[0].value.value;
          this.notification.search.from = receipt.payload.value.msg[0].value.from;
          this.notification.search.to = receipt.payload.value.msg[0].value.to;
          this.notification.search.token = receipt.payload.value.msg[0].value.symbol;
          return (this.receipt = receipt);
        });
    },
  },
};
</script>
