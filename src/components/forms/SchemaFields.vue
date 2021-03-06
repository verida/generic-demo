<template>
  <ValidationObserver ref="validator" mode="eager" v-slot="{ invalid }">
    <ValidationProvider
      v-slot="{ errors }"
      v-for="(item, key) in data"
      :key="key"
      :rules="{ required: attributes[key].required }"
      :name="attributes[key].title">
        <label>{{attributes[key].title}}</label>
        <b-form-group v-if="attributes[key].enum">
          <b-form-radio-group v-model="data[key]">
            <b-form-radio v-for="item in attributes[key].enum"
              :key="item"
              :value="item">
              {{ item }}
            </b-form-radio>
          </b-form-radio-group>
        </b-form-group>
        <datetime
          v-else-if="attributes[key].format && (attributes[key].format.includes('date') || attributes[key].format.includes('date-date'))"
          :auto="true"
          :format="formatDatetime(attributes[key].format)"
          :type="attributes[key].format.replace('-', '')"
          input-class="form-control"
          v-model="data[key]" />
        <b-form-textarea
          v-else-if="attributes[key].inputType === 'textarea'"
          class="form-control word-break" spellcheck="false"
          v-model="data[key]"
          :name="attributes[key].title"
          :state="!data[key] ? null : !errors[0]"
          size="sm" rows="3" />
        <b-form-input
          v-else
          class="form-control"
          size="sm"
          :type="attributes[key].type | typed"
          :value="data[key]" @input="value => format(key, value)"
          :name="attributes[key].title"
          :state="!data[key] ? null : !errors[0]" />
        <b-form-invalid-feedback>
          {{ errors[0] }}
        </b-form-invalid-feedback>
    </ValidationProvider>
    <b-button @click="submit" :disabled="invalid" variant="primary">
      Send
    </b-button>
  </ValidationObserver>
</template>

<script>
import DateFormatMixin from '@/mixins/date-format'
import { DATA_SEND } from '@/constants/inbox'
import { extract } from '@/helpers/NameModifier'
import Verida from '@verida/datastore'
import { DateTime } from 'luxon'

import { createNamespacedHelpers } from 'vuex'
const {
  mapState: mapSystemState,
  mapMutations: mapSystemMutations
} = createNamespacedHelpers('system')

export default {
  name: 'SchemaFields',
  filters: {
    typed (str) {
      if (str === 'string') return 'text'
      if (str === 'number') return 'number'
      return 'text'
    }
  },
  props: [
    'data',
    'attributes',
    'entity'
  ],
  mixins: [
    DateFormatMixin
  ],
  computed: {
    ...mapSystemState([
      'recipient'
    ])
  },
  methods: {
    ...mapSystemMutations([
      'setProcessing'
    ]),
    async submit () {
      this.setProcessing(true)
      const message = []
      const schemaId = this.entity['$id']

      // Populate metadata fields
      switch (schemaId) {
        case 'https://schemas.verida.io/identity/kyc/AU/schema.json':
          await this.createCredential()
          const now = DateTime.local().toFormat('d MMM yyyy')
          this.data.name = `${this.data.firstName} ${this.data.lastName} - KYC`
          this.data.summary = `Issued on ${now} in ${this.data.state}`
          break;
        case 'https://schemas.verida.io/health/pathology/tests/covid19/pcr/schema.json':
          this.data.name = `${this.data.fullName}: COVID-19 PCR`
          this.data.summary = `Result: ${this.data.result}`
          this.data.testTimestamp = (new Date()).toISOString()
          break;
      }

      const store = await window.veridaApp.openDatastore(schemaId)
      const data = this.formatData(this.data)

      const payload = {
        name: extract(data, schemaId),
        ...data
      }

      // Delete any key avlues that are empty strings
      Object.keys(payload).forEach((key) => (payload[key] == '') && delete payload[key]);

      message.push(payload)
      const saved = await store.save(payload)

      if (!saved) {
        this.setProcessing(false)
        console.error(store.errors)
        this.$bvToast.toast(`An error occurred, when generating credential`, {
          title: 'Unable to generate',
          autoHideDelay: 3000,
          variant: 'danger'
        })
        return false
      }

      await this.sendInbox(message, payload.name)
    },
    format (key, value) {
      this.data[key] = this.attributes[key].type === 'number' ? Number(value) : value
    },
    formatData(data) {
      for (var key in data) {
        // convert date-time to proper format for JSON schema
        if (this.attributes[key] && this.attributes[key].type == 'string' && this.attributes[key].format == 'date') {
          data[key] = DateTime.fromISO(data[key]).toFormat('yyyy-LL-dd')
        }
      }

      return data
    },
    async sendInbox (message, name) {
      const { outbox } = window.veridaApp

      const inboxType = DATA_SEND
      const outboxItem = { data: message }
      const text = `You have a new ${this.entity.title}: "${name}"`

      try {
        await outbox.send(this.recipient, inboxType, outboxItem, text, {})
        //this.$emit('reset')
        this.setProcessing(false)

        this.$bvToast.toast(`Created ${this.entity.title} is sent to ${this.recipient}`, {
          title: 'Inbox sent',
          autoHideDelay: 3000,
          variant: 'success'
        })
      } catch (e) {
        this.setProcessing(false)
        console.info(e)
        this.$bvToast.toast(`An error occurred, when sending ${this.entity.title}`, {
          title: 'Inbox hasn\'t been sent',
          autoHideDelay: 3000,
          variant: 'danger'
        })
      }
    },
    async createCredential () {
      const now = new Date()
      const credential = {
        "@context": [
            "https://www.w3.org/2018/credentials/v1",
            "https://www.w3.org/2018/credentials/examples/v1"
        ],
        "id": this.entity['id'],
        "type": ["VerifiableCredential"],
        "issuer": window.veridaApp.user.did,
        "issuanceDate": now.toISOString(),
        "credentialSubject": {
            "id": this.recipient,
            ...this.data
        }
      };

      const issuer = await Verida.Helpers.credentials.createIssuer(window.veridaApp.user)
      this.data.didJwtVc = await Verida.Helpers.credentials.createVerifiableCredential(credential, issuer)
    }
  }
}
</script>
