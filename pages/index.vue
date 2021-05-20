<template>
  <div class="container">
    <div>
      <h1 class="title">
        simple-spot-tracker
      </h1>
      <div v-if="$strapi.user">
        Logged in
        <form @submit="importBinance">
          <input type="file" required class="form__input" accept=".xlsx, .csv" name="importBinance" id="importBinance" />
          <input type="submit" value="Import trades"/>
        </form>
        <table>
          <tr>
            <th>Market</th>
            <th>Amount</th>
            <th>Principal</th>
            <th>No of Trades.</th>
          </tr>
          <tbody>
            <tr v-for="(item, index) in tradeMarketsData" :key="index">
              <td>{{ index }}</td>
              <td>{{ item.length }}</td>
            </tr>
          </tbody>
        </table>
      </div>
      <div v-else class="authentication__forms">
        <div v-show="error" class="form__errors">
          <p> Error(s): {{ error }} </p>
        </div>
        <div class="login">
          <h2>Login</h2>
          <form @submit="login">
            <div>
              <input v-model="email" class="form__input" type="email" placeholder="email" />
            </div>
            <div>
              <input v-model="password" class="p-3 my-5 border w-full" type="password" placeholder="password" />
            </div>
            <div>
              <button :disabled="identifier === '' || password === ''" class="button--green" type="submit">
                Login
              </button>
            </div>
          </form>
        </div>
        <div class="register">
          <h2>Register</h2>
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
              <button class="button--green" type="submit">
                Signup
              </button>
            </div>
          </form>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import XLSX from 'xlsx';

export default {
  data() {
    return {
      identifier: "",
      email: "",
      username: "",
      password: "",
      error: "",
      loggedIn: false,
      trades: [],
      importedData: [],
      tradeMarketsData: {},
      tabularData: {},
    };
  },

  fetchOnServer: false,

  async fetch() {
    this.fetchTrades();
  },

  computed: {
    user() {
      console.log(this.$strapi.user);
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
          identifier: this.identifier,
          password: this.password,
        })
        console.log(user)
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
        console.log(newUser)
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

    processTradesData() {
      // this.$set(this.tradeMarketsData, 'trade', [1]);
      this.trades.forEach(trade => {
        if(this.tradeMarketsData[trade.market] === undefined) {
          this.$set(this.tradeMarketsData, trade.market, []);
        }
        this.tradeMarketsData[trade.market].push(trade);
      });



      for (const [key, trades] of Object.entries(this.tradeMarketsData)) {
        let title = key;
        let amount = 0,
        principal = 0,
        totalBoughtCost = 0,
        totalSoldCost = 0,
        noOfTrades = trades.length;
        trades.forEach(trade => {
          console.log(title, trade);
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
        })

        this.$set(this.tabularData, title, {
          amount:amount,
          principal,
          totalBoughtCost,
          totalSoldCost,
          noOfTrades,
        });
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
      console.log("I have been called");
      // Upload each entry to DB
      this.importedData.forEach(trade => {
        this.$strapi.$trades.create({
          amount: trade.Amount,
          type: trade.Type,
          total: trade.Total,
          price: trade.Price,
          market: trade.Market,
          users_permissions_user: this.user,
        });
      });

      this.fetchTrades();
    }

  }
}
</script>

<style>
.container {
  margin: 0 auto;
  min-height: 100vh;
  display: flex;
  justify-content: center;
  align-items: center;
  text-align: center;
}

.title {
  font-family:
    'Quicksand',
    'Source Sans Pro',
    -apple-system,
    BlinkMacSystemFont,
    'Segoe UI',
    Roboto,
    'Helvetica Neue',
    Arial,
    sans-serif;
  display: block;
  font-weight: 300;
  font-size: 100px;
  color: #35495e;
  letter-spacing: 1px;
}

.subtitle {
  font-weight: 300;
  font-size: 42px;
  color: #526488;
  word-spacing: 5px;
  padding-bottom: 15px;
}

.links {
  padding-top: 15px;
}
</style>
