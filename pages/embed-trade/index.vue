<template lang="pug">
.container.mb-5.mt-1
  .row
    .col
      .row.mt-2
        .col
          nuxt-link(to="/new_market")
            //.new-market-btn
            el-button(tag="el-button" type="primary" size="big" icon="el-icon-plus") Open new market

      el-table(:data='filteredItems', style='width: 100%' @row-click="clickOrder")
        el-table-column(label='Pair', prop='date' width="300")
          template(slot-scope="scope")
            TokenImage(:src="$tokenLogo(scope.row.token.symbol.name, scope.row.token.contract)" height="30")
            span.ml-2
              | {{ scope.row.token.symbol.name }}
              a(:href="monitorAccount(scope.row.token.contract)" target="_blank") ({{ scope.row.token.contract }})

        el-table-column(label='Last price', prop='name')
          template(slot-scope="scope")
            .text-success {{ scope.row.last_price | humanPrice }}
        el-table-column(label='24H Volume', prop='name')
          template(slot-scope="scope")
            span.text-mutted {{ scope.row.volume24 }}
        el-table-column(label='24H Change %', prop='name')
          span.text-mutted Available soon
        el-table-column(align='right')
          template(slot='header', slot-scope='scope')
            el-input(size="small" v-model="search" placeholder="Filter by token")

</template>

<script>
import { captureException } from '@sentry/browser'

import { mapGetters, mapState } from 'vuex'
import TokenImage from '~/components/elements/TokenImage'

export default {
  layout: 'embed',

  components: {
    TokenImage
  },

  async fetch({ store, error }) {
    try {
      await store.dispatch('loadMarkets')
    } catch (e) {
      captureException(e)
      return error({ message: e, statusCode: 500 })
    }
  },

  data() {
    return {
      search: '',

      to_assets: [],

      select: {
        from: '',
        to: ''
      },

      loading: true,
    }
  },

  computed: {
    ...mapGetters(['user', 'markets']),
    ...mapGetters('chain', ['rpc']),
    ...mapState({
      markets: state => state.markets
    }),

    filteredItems() {
      return this.markets.filter((i) => {
        if (i.token.str.toLowerCase().includes(this.search.toLowerCase()))
          return true
      }).reverse()
    },

    activeTab: {
      get () {
        return this.$store.state.market.activeTab
      },

      set (value) {
        this.$store.commit('market/setActiveTab', value)
      }
    }

  },
  methods: {
    clickOrder(a, b, event) {
      if (event && event.target.tagName.toLowerCase() === 'a') return

      this.$router.push({ name: 'embed-trade-id', params: { id: this.marketSlug(a) } })
    }
  }
}
</script>

<style>
.search-input {
  width: 100px;
}

#markets {
  margin-top: 10px;
  flex-wrap: wrap;
}

.new-market {
  width: 260px;
  padding: 16px;
}

.market, .market-new {
  width: 260px;
  padding: 0px 10px;
  height: 75px;
}

.market {
  color:inherit;
  text-decoration: none;

  display: block;
  font-size: 87.5%;
  color: #212529;
}

.market:hover {
  cursor: pointer;
  font-weight: bold;
  font-size: 14px;
  color:inherit;
  text-decoration: none;
}

.market:hover img {
  height: 35px;
}

.order-row {
  cursor: pointer;
}
</style>
