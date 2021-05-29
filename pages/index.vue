<template>
  <div>
    <header>
      <h1 class="title">
        Simple Spot Tracker.
      </h1>
      <h4 class="subtitle">
        A simple Binance spot tracker to give extra insight on positions you're HODLing.
      </h4>
      <h6 class="subtitle__description">
          ** Currently supports only trades made with BUSD and USDT as base. <br />
          Slightly off but close to the correct things.
      </h6>
      <div class="action__buttons" v-if="$strapi.user">
        <button class="button--green" @click="show()">
          Add Trade(s)
        </button>
        <button class="button--grey" @click="logout()">
          Logout
        </button>
      </div>
    </header>
    <div class="container">
      <div v-if="$strapi.user">
        <v-data-table
          v-if="this.trades.length > 0"
          :headers="headers"
          :items="tabularData"
          :items-per-page="20"
          class="elevation-1"
          @click:row="handleClick"
          loading
          loading-text="Loading... Please wait"
        >
          <template v-slot:item.principal="{ item }">
              {{ item.principal | currency }}
          </template>
          <template v-slot:item.coinCostBasis="{ item }">
              {{ item.coinCostBasis | currency }}
          </template>
          <template v-slot:item.cPrice="{ item }">
              {{ item.cPrice | currency }}
          </template>
          <template v-slot:item.cValue="{ item }">
              {{ item.cValue | currency }}
          </template>
          <template v-slot:item.difference="{ item }">
            <v-chip
              :color="getColor(item.difference)"
              dark
            >
              {{ item.difference }}%
            </v-chip>
          </template>
          <template v-slot:item.totalBoughtCost="{ item }">
              {{ item.totalBoughtCost | currency }}
          </template>
          <template v-slot:item.totalSoldCost="{ item }">
              {{ item.totalSoldCost | currency }}
          </template>
        </v-data-table>

        <div class="empty" v-else>
          <p>You have no trades yet. Add trade(s) above.</p>
        </div>
      </div>
      <div v-else class="authentication">
        <h2 class="title--account">
          Let's get started.
        </h2>
        <div v-show="error" class="forms__errors">
          <p> Error(s): {{ error }} </p>
        </div>
        <div class="forms">
          <div class="forms--form login">
            <p>Sign in with your details</p>
            <form @submit="login">
              <div>
                <input v-model="email" class="form__input" type="email" placeholder="email" />
              </div>
              <div>
                <input v-model="password" class="form__input" type="password" placeholder="password" />
              </div>
              <div>
                <button class="button" type="submit">
                  Login
                </button>
              </div>
            </form>
          </div>
          <p class="xor">
            OR
          </p>
          <div class="forms--form register">
            <p>Create an account to get started.</p>
            <form @submit="signup">
              <div>
                <input v-model="email" class="form__input" type="email" placeholder="email" />
              </div>
              <div>
                <input v-model="username"  class="form__input" type="text" placeholder="username" />
              </div>
              <div>
                <input v-model="password" class="form__input" type="password" placeholder="password"
                />
              </div>
              <div>
                <button class="button" type="submit">
                  Signup
                </button>
              </div>
            </form>
          </div>
        </div>
      </div>
    </div>
    <footer>
      Built with 💰 by Ace! ♠️ → &nbsp; <a href="https://github.com/acekyd/sst/" target="_blank"> GitHub</a>
    </footer>
    <modal name="addTradesModal"
    :min-width="200"
    :min-height="200"
    :scrollable="true"
    :reset="true"
    width="60%"
    height="auto"
    >
      <h2 class="title--account">
          Add or Import Trade(s)
        </h2>
      <div class="forms forms--modal">
        <div class="forms--form add__trade">
          <p>Add a single trade</p>
          <small>** Make this as accurate as you can for the math to add up.</small>
          <div v-show="tradeError" class="forms__errors">
            <p> Error(s): {{ tradeError }} </p>
          </div>
          <form @submit="addTrade">
            <div>
              <input :disabled="isAdding" v-model="trade.coin" class="form__input" type="text" placeholder="Coin e.g BNB" required />
            </div>
            <div>
              <select :disabled="isAdding" v-model="trade.base" class="form__input" required>
                <option value="USDT">USDT</option>
                <option value="BUSD">BUSD</option>
              </select>
            </div>
            <div>
              <select :disabled="isAdding" v-model="trade.type" class="form__input" required>
                <option value="BUY">BUY</option>
                <option value="SELL">SELL</option>
              </select>
            </div>
            <div>
              <input :disabled="isAdding" v-model="trade.amount" class="form__input" type="number" placeholder="Quantity e.g 0.25" step="0.000001" required />
            </div>
            <div>
              <input :disabled="isAdding" v-model="trade.price" class="form__input" type="number" step="0.000001" placeholder="Coin Buy Price in $ e.g 500" required />
            </div>
            <div>
              <input :disabled="isAdding" v-model="trade.total" step="0.001" class="form__input" type="number" placeholder="Total purchase in $ e.g 20" required />
            </div>
            <div>
              <button :disabled="isAdding" class="button" type="submit">
                Add trade
              </button>
            </div>
          </form>
        </div>
        <p class="xor">
          OR
        </p>
        <div class="forms--form import__modal">
          <div>
            <p>Import your Trade History from Binance</p>
            <em>Sign in to Binance > Orders > Trade History > Export Trade History.</em> <br/><br />
            <small>Can only export 3 months at a time. So if you've been trading longer than 3 months, export multiple .xlsx files.</small>
          </div> <br/> <br />
          <form @submit="importBinance">
            <input type="file" :disabled="isAdding" required class="form__input" accept=".xlsx" name="importBinance" id="importBinance" ref="importBinance" />
            <input :disabled="isAdding" type="submit" class="button" value="Import trades"/>
          </form>
        </div>
      </div>
    </modal>
  </div>
</template>

<script>
import XLSX from 'xlsx';

let pricesAPI = "https://api.cryptonator.com/api/ticker/";
const formatConfig = {
  style: "currency",
  currency: "USD", // CNY for Chinese Yen, EUR for Euro
  currencyDisplay: "symbol",
  minimumFractionDigits: 2,
  maximumFractionDigits: 2,
};
const usdFormatter = new Intl.NumberFormat("en-US", formatConfig);

export default {
  data() {
    return {
      email: "",
      username: "",
      password: "",
      error: "",
      loggedIn: false,
      trades: [],
      importedData: [],
      tradeMarketsData: {},
      tabularData: [],
      showAddTradeModal: false,
      isAdding: false,
      trade: {
        coin: '',
        base: 'USDT',
        amount: '',
        type: 'BUY',
        price: '',
        total: '',
      },
      tradeError: "",
      headers: [
          { text: 'Coin', align: 'start', value: 'title' },
          { text: 'Quantity',  value: 'amount' },
          { text: 'Average Cost Basis (USDT)',  value: 'coinCostBasis' },
          { text: 'Current Price (USDT)',  value: 'cPrice' },
          { text: 'Principal (USDT)',  value: 'principal' },
          { text: 'Current Value (USDT)',  value: 'cValue' },
          { text: 'Difference (%)',  value: 'difference' },
          { text: 'Total Buy (USDT)',  value: 'totalBoughtCost' },
          { text: 'Total Sold (USDT)',  value: 'totalSoldCost' },
          { text: 'No of Trades.',  value: 'noOfTrades' },
      ]
    };
  },

  fetchOnServer: false,

  async fetch() {
    this.fetchTrades();
  },

  computed: {
    user() {
      return this.$strapi.user;
    },
  },

  filters: {
    currency: function (value) {
      if (!value) return;
      if (value == '...') return '$...';
      return usdFormatter.format(value);
    }
  },

  methods: {
    show () {
        this.$modal.show('addTradesModal');
    },
    hide () {
        this.$modal.hide('addTradesModal');
    },
    async login(e) {
      e.preventDefault();
      if(this.email === '' || this.password === '') {
        this.error = "All fields are required!"
        return;
      }
      try {
        const user = await this.$strapi.login({
          identifier: this.email,
          password: this.password,
        })
        if (user !== null) {
          this.error = ''
          this.loggedIn = true;
          this.fetchTrades();
        }
      } catch (error) {
        this.error = 'Error in login credentials'
      }
    },

    async signup(e) {
      e.preventDefault();
      if(this.email === '' || this.password === '' || this.username === '') {
        this.error = "All fields are required!"
        return;
      }
      try {
        const newUser = await this.$strapi.register({
          email: this.email,
          username: this.username,
          password: this.password,
        })
        if (newUser !== null) {
          this.error = ''
          this.loggedIn = true;
          this.fetchTrades();
        }
      } catch (error) {
        this.error = error.message
      }
    },

    async logout() {
      await this.$strapi.logout();
    },

    async fetchTrades() {
      if(this.user) {
        const trades = await this.$strapi.$trades.find({'users_permissions_user.username': this.user.username });
        this.trades = trades;
        this.processTradesData();
      } else return [];
    },

    async processTradesData() {
      // reset values to empty table
      this.tradeMarketsData= {};
      this.tabularData= [];

      for(const trade of this.trades) {
        if(this.tradeMarketsData[trade.coin] === undefined) {
          this.$set(this.tradeMarketsData, trade.coin, []);
        }
        this.tradeMarketsData[trade.coin].push(trade);
      }
      for (const [key, trades] of Object.entries(this.tradeMarketsData)) {
        let title = key;
        let amount = 0,
        principal = 0,
        totalBoughtCost = 0,
        totalSoldCost = 0,
        coinCostBasis = 0,
        noOfTrades = trades.length,
        cPrice = "...",
        cValue = "...",
        difference = 0;
        trades.forEach(trade => {
          if(trade.type === "BUY") {
            amount += parseFloat(trade.amount);
            principal += parseFloat(trade.total);
            totalBoughtCost += parseFloat(trade.total);
          }
          else {
            amount -= parseFloat(trade.amount);
            principal -= parseFloat(trade.total);
            totalSoldCost += parseFloat(trade.total);
          }

          coinCostBasis = parseFloat(principal/amount).toFixed(4);
        })

        if(amount > 0.0000000001 ) {
          this.tabularData.push({
            title,
            amount: +amount.toFixed(4),
            principal,
            totalBoughtCost,
            totalSoldCost,
            noOfTrades,
            coinCostBasis,
            cPrice,
            cValue,
            difference
          });
        }
      }

      this.fetchAPIPrices();
    },

    async fetchAPIPrices() {
      const pricesArray = [];
      for(const data in this.tradeMarketsData) {
        console.log(data);
        pricesArray[data] = await this.$axios.$get(pricesAPI+data+"-usdt");
      }

      for(const item of this.tabularData) {
        item.cPrice = parseFloat(pricesArray[item.title].ticker.price).toFixed(4);
        item.cValue = item.cPrice * item.amount;
        item.difference = parseFloat(((item.cPrice - item.coinCostBasis) / item.coinCostBasis)*100).toFixed(2);

      }

    },

    async importBinance(e) {
      e.preventDefault();
      this.isAdding = true;
      let file = document.getElementById("importBinance").files[0];
      var reader = new FileReader();
      const self = this;
			reader.onload = function (e) {
				// pre-process data
				var binary = "";
				var bytes = new Uint8Array(e.target.result);
				var length = bytes.byteLength;
				for (var i = 0; i < length; i++) {
					binary += String.fromCharCode(bytes[i]);
				}

				/* read workbook */
				var wb = XLSX.read(binary, {type: 'binary'});

				/* grab first sheet */
				var wsname = wb.SheetNames[0];
				var ws = wb.Sheets[wsname];

				/* generate JSON */
				self.importedData = XLSX.utils.sheet_to_json(ws);
			};
      await reader.readAsArrayBuffer(file);


      reader.onloadend = function() {
        self.uploadData();
      }
    },

    async addTrade(e) {
      e.preventDefault();
      if(this.trade.coin === '' || this.trade.base === '' || this.trade.amount <= 0 || this.trade.total <= 0) {
        this.tradeError = "All fields are required!"
        return;
      }
      this.isAdding = true;
      try {
        const trade = await this.$strapi.$trades.create({
          coin: this.trade.coin.toUpperCase(),
          base: this.trade.base.toUpperCase(),
          amount: this.trade.amount,
          type: this.trade.type,
          total: this.trade.total,
          price: this.trade.price,
          market: this.trade.coin.toUpperCase()+this.trade.base.toUpperCase(),
          users_permissions_user: this.user,
        })
        if (trade !== null) {
          this.tradeError = ''
          this.hide();
          this.trade = {
            coin: '',
            base: 'USDT',
            amount: '',
            type: 'BUY',
            price: '',
            total: '',
          };
          this.isAdding = false;
          this.hide();
          this.fetchTrades();
        }
      } catch (error) {
        console.log("Error", error);
        this.tradeError = 'An error occurred while adding trade.'
        this.isAdding = false;
      }
    },

    uploadData() {

      // let supportedCurrencies = ['USDT', 'BUSD'];
      // The code below is designed to only support these base currencies while
      // I figure out the more complex things. BUSD and USDT are usually similar to each other.
      // Upload each entry to DB
      this.importedData.forEach(trade => {
        this.$strapi.$trades.create({
          coin: trade.Market.substring(0, trade.Market.length-4),
          base: trade.Market.substring(trade.Market.length-4, trade.Market.length),
          amount: trade.Amount,
          type: trade.Type,
          total: trade.Total,
          price: trade.Price,
          market: trade.Market,
          purchaseDate: trade['Date(UDT)'],
          fee: trade.Fee,
          feeCoin: trade['Fee Coin'],
          users_permissions_user: this.user,
        });
      });
      this.isAdding = false;
      this.$refs.importBinance.value = null;
      this.hide();
      this.fetchTrades();
    },

    handleClick(value) {
      console.log(value);
    },

    getColor(value) {
        if (value > 0) return 'green'
        else if (value < 0) return 'red'
        else return 'green'
    },

    toggleAddTrades() {
      this.showAddTradeModal = !this.showAddTradeModal;
    }

  }
}
</script>

<style lang="scss">
header {
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
  min-height: 250px;
  scroll-behavior: smooth;
  background: #eee;
  padding: 5px;
}

.action__buttons {
  margin-top: 20px;
  margin-bottom: 20px;
  display: flex;
  justify-content: center;
}

.title {
  font-size: 3rem;
  margin:20px;

  @media screen and (max-width: 600px) {
    & {
      font-size: 1.5rem;
    }
  }

  &--account {
    font-size: 25px;
    margin: 20px;
    font-weight: 400;
    text-align: center;
  }
}

footer {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 50px;
  line-height: 50px;
  background: #eee;
}



.subtitle {
  text-align: center;
  font-weight: 400;
  font-size: 150%;
  margin-bottom: 20px;

  &__description {
    text-align: center;
    font-size: 12px;
  }
}

.authentication {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.forms {
  display: flex;
  justify-content: center;

  @media screen and (max-width: 600px) {
    & {
      flex-direction: column;
    }
  }

  &--modal {
    margin: 20px;

    p {
      font-weight: 500;
    }
  }

  &__errors {
    color: red;
    font-weight: 600;
    margin-bottom: 10px;
  }

  .xor {
    text-align: center;
    margin: 20px 0;
    font-size: 1.5rem;
  }

  @media screen and (min-width: 600px) {
      .xor {
        display: none;
      }
  }

  &--form {
    max-width: 300px;
    margin-bottom: 10px;


    @media screen and (min-width: 600px) {
      & {
        margin-bottom: 0;
      }
      &:first-child {
        margin-right: 20px;
        border-right: 1px solid #ccc;
      }

      .xor {
        display: none;
      }

      &:last-child {
        margin-left: 20px;
      }
    }

    p {
      text-align: center;
      font-size: 14px;
      margin-bottom: 15px;
    }

    .button {
      width: 90%;
      margin-left: 5px;
      margin-top: 10px;
      display: inline-block;
      border-radius: 4px;
      border: 1px solid #35495e;
      color: #35495e;
      text-decoration: none;
      padding: 10px 30px;
    }
  }
}

.form__input {
  width: 90%;
  padding: 10px;
  box-sizing: border-box;
  border-style: solid;
  margin: 5px;
  border-width: 0.5px;
}

.green {
    background-color: #4caf50!important;
    border-color: #4caf50!important;
}

.red {
  background-color: #f44336!important;
  border-color: #f44336!important;
}

.import__modal {
  text-align: center;
}

.add__trade small {
  display: inline-block;
  text-align: center;
  margin-bottom: 10px;
}

.add__trade input[type=text] {
  text-transform: uppercase;
}

.text-start {
  text-align: start;
}

.empty {
  margin-top: 30px;
  text-align: center;
}
</style>
