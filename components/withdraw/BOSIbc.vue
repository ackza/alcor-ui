<template lang="pug">
// TODO boscore withdraws https://boscore.gitbook.io/docs/essentials/bos-essentials/ibc-inter-blockchain-communication/user-guide
.ibc-withdraw
  el-button(type="primary" icon="el-icon-s-promotion" size="mini" @click="open").ml-auto BOS Ibc Transfer

  el-dialog(title="Withdraw using Inter Blockchain Communication", :visible.sync="visible" width="25%" v-if="user")
    el-form(ref="form" :model="form" label-position="left" :rules="rules")
      el-form-item.mb-2
        b Where you want transfer {{ token.symbol.name }} ?
        .row
          .col
            el-radio-group(v-model='chain_select' @change="setChain")
              el-radio-button(v-for="chain in chains" :key='chain', :value='chain' :label='chain')
                img(:src="require('~/assets/icons/' + chain + '.png')" height=20).mr-2
                span {{ chain }}

      el-form-item.mt-1(v-if="chain.name" prop="address")
        template(slot="label")
          b {{ chain.name }} address:
        el-input(v-model="form.address" placeholder="address..").w-100

      el-form-item(v-if="chain.name" prop="amount")
        template(slot="label")
          b Withdraw amount

          span.w-100  (Balance {{ tokenBalance }})

        el-input(type="number" v-model="form.amount" clearable @change="amountChange").w-100
          span(slot="suffix").mr-1 {{ this.token.symbol.name }}

      el-form-item.mt-1(v-if="chain.name")
        span.dialog-footer(v-if="chain.name").mb-4
          el-button(type='primary' @click="submit" :disabled="!form.amount || !addressValid" :loading="loading").w-100
            | Transfer {{ token.symbol.name }} to {{ chain.desc }}
</template>

<script>
import fetch from 'node-fetch'
import { JsonRpc } from 'eosjs'

import { captureException } from '@sentry/browser'
import { mapGetters, mapState } from 'vuex'

import config from '~/config'
import TokenImage from '~/components/elements/TokenImage'



export default {
  components: {
    TokenImage
  },

  data() {
    return {
      visible: false,

      form: {
        address: '',
        amount: 0.0
      },

      addressValid: false,

      chain_select: null,
      chain: {},

      loading: false,

      rules: {
        address: {
          trigger: 'blur',
          validator: async (rule, value, callback) => {
            const rpc = new JsonRpc(`${this.chain.protocol}://${this.chain.host}:${this.chain.port}`, { fetch })

            try {
              this.loading = true
              await rpc.get_account(value)
              this.addressValid = true
              callback()
            } catch (e) {
              this.addressValid = false
              callback(new Error('Account not exists!'))
            } finally {
              this.loading = false
            }
          }
        }
      }
    }
  },

  computed: {
    ...mapState(['user', 'network']),
    ...mapState('market', ['token']),
    ...mapGetters(['tokenBalance']),

    networks() {
      return Object.values(config.networks)
    },

    chains() {
      return ['eos', 'bos', 'telos', 'wax'].filter(c => c != this.network.name)
    }
  },

  methods: {
    setChain(name) {
      this.chain = config.networks[name]
      this.form.address = ''
      this.$refs.form.resetFields()
    },

    amountChange() {
      this.form.amount = (parseFloat(this.form.amount) || 0).toFixed(this.token.symbol.precision)
    },

    async open() {
      if (!await this.$store.dispatch('chain/asyncLogin')) return
      this.visible = true
    },

    async submit() {
      const quantity = this.form.amount + ' ' + this.token.symbol.name
      const chain = this.chain.baseToken.symbol.toLowerCase()
      const memo = this.network.name != 'bos'
        ? `hub.io@bos >> ${this.form.address}@${chain} alcor.exchange (IBC transfer)`
        : `${this.form.address}@${chain} alcor.exchange (IBC transfer)`

      const loading = this.$loading({ lock: true, text: 'Wait for wallet' })
      try {
        const r = await this.$store.dispatch('chain/transfer', {
          to: 'bosibc.io',
          contract: this.token.contract,
          actor: this.user.name,
          quantity,
          memo
        })

        this.visible = false

        this.$alert(`<a href="${this.network.monitor}/tx/${r.transaction_id}" target="_blank">Transaction id</a>`, 'Transaction complete!', {
          dangerouslyUseHTMLString: true,
          confirmButtonText: 'OK',
          callback: (action) => {
            this.$notify({ title: 'Withdraw in process', type: 'success' })
          }
        })

        //this.$notify({ title: 'Place order', message: r.processed.action_traces[0].inline_traces[1].console, type: 'success' })
      } catch (e) {
        captureException(e)
        this.$notify({ title: 'Withdraw error', message: e.message, type: 'error' })
        console.log(e)
      } finally {
        loading.close()
      }
    }
  }
}
</script>

<style>
.upperinput {
    text-transform: uppercase;
}
.upperinput:placeholder-shown {
    text-transform: none;
}

.ibc-withdraw .el-dialog__body {
  padding: 0px 20px 5px 20px;
}
</style>
