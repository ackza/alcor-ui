<template lang="pug">
.container-fluid.mb-5.mt-1
  .row.mt-2
    .col-9
      .row
        .col
          .row
            .col
              .text.item
                .row.trade-window(v-if="!isMobile")
                  .col-lg-5
                    .row
                      .col
                        .overflowbox.box-card.p-2
                          .row
                            .col-md-2.p-1.pl-4
                              TokenImage(:src="$tokenLogo(token.symbol.name, token.contract)" height="40")

                            .col-md-6
                              .row
                                .col
                                  b {{ token.symbol.name }}@
                                  a(:href="monitorAccount(token.contract )" target="_blank") {{ token.contract }}
                              .row
                                .col
                                  span Volume 24H:
                                  span.text-success  {{ stats.volume24 }}
                            .col-md-4(v-if="isPeg")
                              withdraw
                          .row(v-if="isPeg")
                            .col
                              .p-2
                                .text {{ this.network.pegs[this.token.str].desc }}
                    .row.mt-2
                      .col
                        order-book(v-loading="loading")

                    .row
                      .col
                        LatestDeals.mt-2
                  .col-lg-7
                    .row
                      .col
                        chart

                    .row
                      .col
                        el-tabs.h-100
                          el-tab-pane(label="Limit trade")
                            LimitTrade

                          el-tab-pane(label="Market trade")
                            market-trade

                // Mobile verion
                .trade-window(v-else)
                  .row
                    .col
                      .overflowbox.box-card.p-2
                        .row
                          .col-2.p-1.pl-4
                            TokenImage(:src="$tokenLogo(token.symbol.name, token.contract)" height="40")

                          .col-10
                            .row
                              .col
                                b {{ token.symbol.name }}@
                                a(:href="monitorAccount(token.contract )" target="_blank") {{ token.contract }}
                            .row
                              .col
                                span Volume 24H:
                                span.text-success.ml-1  {{ stats.volume24 }}
                  chart

                  .text.item
                    MobileTrade

    .col-3
      .overflowbox
        markets
  .row
    .col
      hr
      .row
        .col
          my-orders(v-if="user" v-loading="loading")
</template>

<script>
import { Name, SymbolCode } from 'eos-common'
import { captureException } from '@sentry/browser'
import { mapGetters, mapState } from 'vuex'
import TokenImage from '~/components/elements/TokenImage'
import AssetImput from '~/components/elements/AssetInput'

import MarketTrade from '~/components/trade/MarketTrade'
import LimitTrade from '~/components/trade/LimitTrade'
import MyOrders from '~/components/trade/MyOrders'
import OrderBook from '~/components/trade/OrderBook'
import Markets from '~/components/trade/Markets'
import LatestDeals from '~/components/trade/LatestDeals'
import Chart from '~/components/trade/Chart'
import MobileTrade from '~/components/trade/MobileTrade'

export default {
  layout: 'embed',

  components: {
    TokenImage,
    AssetImput,
    MarketTrade,
    MyOrders,
    LimitTrade,
    OrderBook,
    LatestDeals,
    Chart,
    Markets,
    MobileTrade,
  },

  async fetch({ store, error, params }) {
    const [symbol, contract] = params.id.split('-')

    let market_id

    if (contract && symbol) {
      // If it's slug

      //if (c_market) {
      //  market_id = c_market.id
      //} else {
      const i128_id = new Name(contract).value.shiftLeft(64).or(new SymbolCode(symbol.toUpperCase()).raw()).toString(16)

      const { rows: [market] } = await store.getters['api/rpc'].get_table_rows({
        code: store.state.network.contract,
        scope: store.state.network.contract,
        table: 'markets',
        lower_bound: `0x${i128_id}`,
        key_type: 'i128',
        index_position: 2,
        limit: 1
      })

      if (!market) {
        error(`Market ${symbol}@${contract} not found!`)
      }

      market_id = market.id
      //}
    } else {
      market_id = params.id
    }

    store.commit('market/setId', market_id)

    this.loading = true
    try {
      await Promise.all([
        store.dispatch('market/fetchMarket'),
        store.dispatch('market/fetchDeals')
      ])
    } catch (e) {
      captureException(e)
      return error({ message: e, statusCode: 500 })
    } finally {
      this.loading = false
    }
  },

  data() {
    return {
      price: 0.0,
      amount: 0.0,

      no_found: false,
      loading: false
    }
  },

  computed: {
    ...mapState(['network']),
    ...mapGetters('chain', ['rpc', 'scatter']),
    ...mapState('market', ['token', 'id', 'stats']),
    ...mapGetters(['user']),

    isPeg() {
      return Object.keys(this.network.pegs).includes(this.token.str)
    }
  },

  head() {
    return {
      title: `Alcor Exchange | Market ${this.token.symbol.name}`,

      meta: [
        { hid: 'description', name: 'description', content: `Trade ${this.token.str} token for EOS` }
      ]
    }
  }
}
</script>

<style scoped>
.bordered {
  border-right: 1px solid;
}

.trade-window {
  min-height: 650px;
}

.el-form-item {
  margin-bottom: 0px;
}

.el-card__body {
    padding: 10px;
}

input[type=number]::-webkit-inner-spin-button,
input[type=number]::-webkit-outer-spin-button {
  -webkit-appearance: none;
  margin: 0;
}

.display-4 {
  font-size: 2.5rem;
  font-weight: 230;
  line-height: 1;
}
</style>
