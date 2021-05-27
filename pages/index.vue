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
    </header>
    <div class="container">
      <div v-if="$strapi.user">
        Logged in
        <button class="button--green" @click="logout()">
          Logout
        </button>
        <form @submit="importBinance">
          <input type="file" required class="form__input" accept=".xlsx, .csv" name="importBinance" id="importBinance" ref="importBinance" />
          <input type="submit" value="Import trades"/>
        </form>
        <v-data-table
          :headers="headers"
          :items="tabularData"
          :items-per-page="20"
          class="elevation-1"
        ></v-data-table>
      </div>
      <div v-else class="authentication">
        <h2 class="title--account">
          Let's get started.
        </h2>
        <div v-show="error" class="authentication__forms__errors">
          <p> Error(s): {{ error }} </p>
        </div>
        <div class="authentication__forms">
          <div class="authentication__forms--form login">
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
          <div class="authentication__forms--form register">
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
  </div>
</template>

<script>
import XLSX from 'xlsx';

let pricesAPI = "https://api.cryptonator.com/api/ticker/";

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
      headers: [
          {
            text: 'Market',
            align: 'start',
            value: 'title',
          },
          { text: 'Coin Amount', value: 'amount' },
          { text: 'Principal (USDT)', value: 'principal' },
          { text: 'Average Cost Basis (USDT)', value: 'coinCostBasis' },
          { text: 'Current Price (USDT)', value: 'cPrice' },
          { text: 'Difference (%)', value: 'difference' },
          { text: 'Total Buy (USDT)', value: 'totalBoughtCost' },
          { text: 'Total Sold (USDT)', value: 'totalSoldCost' },
          { text: 'No of Trades.', value: 'noOfTrades' },
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

  methods: {
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
      const pricesArray = [];

      for(const trade of this.trades) {
        if(this.tradeMarketsData[trade.coin] === undefined) {
          pricesArray[trade.coin] = await this.$axios.$get(pricesAPI+trade.coin+"-usdt");
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
        cPrice = parseFloat(pricesArray[title].ticker.price).toFixed(4),
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

        difference = parseFloat(((cPrice-coinCostBasis)/coinCostBasis)*100).toFixed(2);

        if(amount > 0 ) {
          this.tabularData.push({
            title,
            amount: amount.toFixed(4),
            principal: principal.toFixed(4),
            totalBoughtCost: totalBoughtCost.toFixed(4),
            totalSoldCost: totalSoldCost.toFixed(4),
            noOfTrades,
            coinCostBasis,
            cPrice,
            difference
          });
        }
      }
    },

    async importBinance(e) {
      e.preventDefault();
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
      this.$refs.importBinance.value = null;
      this.fetchTrades();
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
  height: 250px;
  scroll-behavior: smooth;
  background: #eee;
}

.title {
  font-size: 40px;
  margin:20px;

  &--account {
    font-size: 25px;
    margin: 20px;
    font-weight: 400;
  }
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

  &__forms {
    display: flex;
    justify-content: center;

    &__errors {
      color: red;
      font-weight: 600;
      margin-bottom: 10px;
    }

    &--form {
      width: 300px;

      &:first-child {
        margin-right: 20px;
        border-right: 1px solid #ccc;
      }

      &:last-child {
        margin-left: 20px;
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
}

.form__input {
  width: 90%;
  padding: 10px;
  box-sizing: border-box;
  border-style: solid;
  margin: 5px;
  border-width: 0.5px;
}
</style>
