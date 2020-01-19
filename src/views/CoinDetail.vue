<template>
  <div class="flex-col">
    <div class="flex justify-content">
      <pacman-loader :loading="isLoading" :color="'#68d391'" :size="50" />
    </div>
    <template v-if="!isLoading">
      <div class="flex flex-col sm:flex-row justify-around items-center">
        <div class="flex flex-col items-center">
          <img
            :src="
              `https://static.coincap.io/assets/icons/${asset.symbol.toLowerCase()}@2x.png`
            "
            :alt="asset.name"
            class="w-20 h-20 mr-5"
          />
          <h1 class="text-5xl">
            {{ asset.name }}
            <small class="sm:mr-2 text-gray-500"> {{ asset.symbol }}</small>
          </h1>
        </div>

        <div class="my-10 flex flex-col">
          <ul>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Ranking</b>
              <span># {{ asset.rank }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio actual</b>
              <span> {{ asset.priceUsd | dollar }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio más bajo</b>
              <span> {{ min | dollar }}</span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio más alto</b>
              <span> {{ max | dollar }} </span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Precio Promedio</b>
              <span> {{ avg | dollar }} </span>
            </li>
            <li class="flex justify-between">
              <b class="text-gray-600 mr-10 uppercase">Variación 24hs</b>
              <span> {{ asset.changePercent24Hr | percent }}</span>
            </li>
          </ul>
        </div>

        <div class="my-10 sm:mt-0 flex flex-col justify-center text-center">
          <button
            @click="toggleConverter"
            class="bg-green-500 hover:bg-green-700 text-white font-bold py-2 px-4 rounded"
          >
            {{ fromUsd ? `USD a ${asset.symbol}` : `${asset.symbol} a USD` }}
          </button>

          <div class="flex flex-row my-5">
            <label class="w-full" for="convertValue">
              <input
                v-model="convertValue"
                id="convertValue"
                type="number"
                class="text-center bg-white focus:outline-none focus:shadow-outline border border-gray-300 rounded-lg py-2 px-4 block w-full appearance-none leading-normal"
                :placeholder="`Valor en ${fromUsd ? 'USD' : asset.symbol}`"
              />
            </label>
          </div>

          <span class="text-xl">
            {{ convertResult }} {{ fromUsd ? asset.symbol : "USD" }}
          </span>
        </div>
      </div>
      <line-chart
        class="my-10"
        :colors="['orange']"
        :min="min"
        :max="max"
        :data="history.map(h => [h.date, parseFloat(h.priceUsd).toFixed(2)])"
      />

      <h2 class="text-center text-xl my-10">Mejores Ofertas de Cambio</h2>
      <table class="table-fixed">
        <thead class="bg-gray-100 border-b-2 border-gray-400">
          <tr>
            <th class="w-1/2 px-4 py-2">Exchange</th>
            <th class="w-1/2 px-4 py-2">Price USD</th>
            <th class="w-1/2 px-4 py-2">Symbols</th>
            <th class="w-1/2 px-4 py-2"></th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="m in markets"
            :key="`${m.exchangeId}-${m.priceUsd}`"
            class="border-b border-orange-500 px-4 py-2 text-center hover:bg-gray-100 hover:bg-orange-100"
          >
            <td>
              <b>{{ m.exchangeId }}</b>
            </td>
            <td>{{ m.priceUsd | dollar }}</td>
            <td>{{ m.baseSymbol }} / {{ m.quoteSymbol }}</td>
            <td class="border px-4 py-2">
              <px-button
                :is-loading="m.isLoading || false"
                v-if="!m.url"
                @custom-click="getWebSite(m)"
              >
                <slot> Obtener Link </slot>
              </px-button>
              <a v-else class="hover:underline text-green-600" target="_blank">
                {{ m.url }}
              </a>
            </td>
          </tr>
        </tbody>
      </table>
    </template>
  </div>
</template>

<script>
import PxButton from "@/components/PxButton";
import api from "@/api";
export default {
  name: "CoinDetail",

  components: { PxButton },

  data() {
    return {
      isLoading: false,
      asset: {},
      history: [],
      markets: [],
      fromUsd: true,
      convertValue: null
    };
  },

  computed: {
    convertResult() {
      if (!this.convertValue) {
        return 0;
      }
      const result = this.fromUsd
        ? this.convertValue / this.asset.priceUsd
        : this.convertValue * this.asset.priceUsd;

      return result.toFixed(4);
    },
    min() {
      return Math.min(
        ...this.history.map(h => parseFloat(h.priceUsd).toFixed(2))
      );
    },

    max() {
      return Math.max(
        ...this.history.map(h => parseFloat(h.priceUsd).toFixed(2))
      );
    },

    avg() {
      return (
        this.history.reduce((a, b) => a + parseFloat(b.priceUsd), 0) /
        this.history.length
      );
    }
  },

  watch: {
    $route() {
      this.getCoin();
    }
  },

  created() {
    this.getCoin();
  },

  methods: {
    toggleConverter() {
      this.fromUsd = !this.fromUsd;
    },

    getWebSite(exchange) {
      this.$set(exchange, "isLoading", true);

      return api
        .getExchange(exchange.exchangeId)
        .then(res => {
          this.$set(exchange, "url", res.exchangeUrl);
        })
        .finally(() => {
          this.$set(exchange, "isLoading", true);
        });
    },
    getCoin() {
      const id = this.$route.params.id;
      this.isLoading = true;
      Promise.all([
        api.getAsset(id),
        api.getAssetHistory(id),
        api.getMarkets(id)
      ])
        .then(([asset, history, markets]) => {
          (this.asset = asset),
            (this.history = history),
            (this.markets = markets);
        })
        .finally(() => (this.isLoading = false));
      //api.getAsset(id).then(asset => (this.asset = asset));
    }
  }
};
</script>

<style scoped>
td {
  padding: 10px;
  text-align: center;
}
</style>
