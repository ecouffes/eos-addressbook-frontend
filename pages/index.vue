<template>
  <section class="container mt-5">
    <nav class="navbar navbar-dark bg-secondary mb-3">
      <span class="navbar-brand mb-0 h1">連絡先リスト</span>
      <span v-if='account' class="navbar-text">{{account.name}}@{{account.authority}}</span>
    </nav>

    <b-alert :show='this.warning' variant="warning" dismissible>{{this.warning}}</b-alert>
    <b-alert :show='this.info' variant="success" dismissible>{{this.info}}</b-alert>

    <table class="table table-striped">
      <thead>
      <tr>
        <th>ID</th>
        <th>名前</th>
        <th>住所</th>
        <th>電話番号</th>
        <th></th>
      </tr>
      </thead>
      <tbody>
        <tr v-for='contact in contacts' :key='contact.id'>
          <td>{{contact.id}}</td>
          <td>{{contact.name}}</td>
          <td>{{contact.address}}</td>
          <td>{{contact.tel}}</td>
          <td>
            <b-btn variant="outline-success" size='sm' @click="editContact(contact)">編集</b-btn>
            <b-btn variant="outline-danger" size='sm' @click="deleteContact(contact)">削除</b-btn>
          </td>
        </tr>
      </tbody>
    </table>

    <b-btn variant="outline-primary" @click="newContact">新規登録</b-btn>

    <b-modal v-model="showModal" id="contactModal" :title="modalTitle" centered>
      <b-form>
        <b-form-group label="名前" label-for="name">
          <b-form-input type="text" v-model="modalContact.name" :state="!$v.modalContact.name.$invalid" aria-describedby="nameFeedback"></b-form-input>
          <b-form-invalid-feedback id="nameFeedback">名前は 2-10 文字で入力してください</b-form-invalid-feedback>
        </b-form-group>
        <b-form-group label="住所" label-for="address">
          <b-form-input type="text" v-model="modalContact.address" :state="!$v.modalContact.address.$invalid" aria-describedby="addressFeedback"></b-form-input>
          <b-form-invalid-feedback id="addressFeedback">住所は 4-20 文字で入力してください</b-form-invalid-feedback>
        </b-form-group>
        <b-form-group label="電話番号" label-for="tel">
          <b-form-input type="text" v-model="modalContact.tel" :state="!$v.modalContact.tel.$invalid" aria-describedby="telFeedback" ></b-form-input>
          <b-form-invalid-feedback id="telFeedback">電話番号は 10-13 文字で入力してください</b-form-invalid-feedback>
        </b-form-group>
      </b-form>

      <div slot="modal-footer" class="w-100">
        <div class="float-right">
          <b-btn size="sm" variant="default" @click="showModal = false">
            キャンセル
          </b-btn>
          <b-btn size="sm" class="ml-1" :variant="modalButtonClass" @click="saveContact" :disabled="$v.modalContact.$invalid">
            {{modalButtonText}}
          </b-btn>
        </div>
      </div>
    </b-modal>

  </section>
</template>

<script>
  import Vue from 'vue'
  import { validationMixin } from 'vuelidate'
  import { required, minLength, maxLength } from 'vuelidate/lib/validators'
  import Eos from 'eosjs'
  import ScatterJS from 'scatter-js/dist/scatter.esm'

  const CONTRACT_ACCOUNT = 'eosaddressbk'

  export default {
    data () {
      return {
        eos: null,
        eosContract: null,
        account: null,
        info: null,
        warning: null,
        modalContact: {},
        showModal: false,
        contacts: []
      }
    },

    mixins: [
      validationMixin
    ],

    computed: {
      idExists () {
        return this.modalContact && this.modalContact.id
      },
      modalTitle () {
        return this.idExists ? '連絡先編集': '連絡先新規登録'
      },
      modalButtonClass () {
        return 'outline-' + (this.idExists ? 'success' : 'primary')
      },
      modalButtonText () {
        return this.idExists ?  '更新する' : '登録する'
      },
    },

    validations: {
      modalContact: {
        name: {
          required,
          minLength: minLength(2),
          maxLength: maxLength(10)
        },
        address: {
          required,
          minLength: minLength(4),
          maxLength: maxLength(20)
        },
        tel: {
          required,
          minLength: minLength(10),
          maxLength: maxLength(13)
        }
      }
    },

    async asyncData (context) {

      const scatter = ScatterJS.scatter

      // ユーザーのScatterにconnect
      const connected = await scatter.connect('EOS Kylin Addressbook', { initTimeout: 5000 })
      if (connected) {
        window.scatter = null
      } else {
        console.log('Please use Scatter application to enter this DApp!')
        return false
      }

      // Kylin Testnet Network info
      const network = {
        blockchain: 'eos',
        protocol: 'https',
        host: 'api-kylin.eoslaomao.com',
        port: 443,
        chainId: '5fff1dae8dc8e2fc4d5b23b2c7665c97f9e9d8edf2b6485a86ba311c25639191'
      }

      // Scatterにnetwork情報を渡す
      const requiredFields = { accounts: [network] }
      await scatter.getIdentity(requiredFields)

      // ScatterからEOS Blockchainのアカウントを取得
      const account = scatter.identity.accounts.find(x => x.blockchain === 'eos')

      const eosOptions = { expireInSeconds: 60 }
      // Scatterを使ってトランザクションへの署名を可能にするeosjsオブジェクトへの参照を取得
      const eos = scatter.eos(network, Eos, eosOptions)

      // EOSコントラクトオブジェクトを作成
      const eosContract = await eos.contract(CONTRACT_ACCOUNT)

      const result = await eos.getTableRows(true, CONTRACT_ACCOUNT, account.name, 'people')

      return { eos, eosContract, account, contacts: result.rows }
    },

    methods: {
      // EOSブロックチェーンのMulti index tableを参照してcontactsデータを更新
      async refreshcontracts () {
        const result = await this.eos.getTableRows(true, CONTRACT_ACCOUNT, this.account.name, 'people')
        this.contacts = result.rows
      },

      newContact () {
        this.modalContact = {}
        this.showModal = true
      },

      editContact (contact) {
        this.modalContact = Object.assign({}, contact)
        this.showModal = true
      },

      // コンタクト情報を削除
      deleteContact (contact) {
        if (confirm('連絡先を削除します。よろしいですか？')) {
          this.warning = null
          this.info = null

          this.eosContract.destroy(
            this.account.name,
            contact.id,
            { authorization: this.account.name }
          ).then(result => {
            this.warning = 'トランザクションを送信しました'
          }).then(async () => {
            this.refreshcontracts()
            this.info = '連絡先を削除しました'
          })
        }
      },

      saveContact () {
        if (this.idExists) {
          this.updateContact()
        } else {
          this.createContact()
        }
        this.showModal = false
      },

      // コンタクト情報を作成
      createContact () {
        this.warning = null
        this.info = null

        this.eosContract.create(
          this.account.name,
          this.modalContact.name,
          this.modalContact.address,
          this.modalContact.tel,
          { authorization: this.account.name }
        ).then(result => {
          this.warning = 'トランザクションを送信しました'
        }).then(async () => {
          this.refreshcontracts()
          this.info = '連絡先を登録しました'
        })
      },

      // コンタクト情報を更新
      updateContact () {
        this.warning = null
        this.info = null

        this.eosContract.update(
          this.account.name,
          this.modalContact.id,
          this.modalContact.name,
          this.modalContact.address,
          this.modalContact.tel,
          { authorization: this.account.name }
        ).then(result => {
          this.warning = 'トランザクションを送信しました'
        }).then(async () => {
          this.refreshcontracts()
          this.info = '連絡先を更新しました'
        })
      }
    }

  }
</script>
<style>

</style>
