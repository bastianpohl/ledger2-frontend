<template>
   <div>
      <v-card
         id="mainObject"
      >
            <AccountSummary
               :account="account"
            />
            <v-card-subtitle
               id="search"
               class="py-0"
            >
               <v-text-field
                  v-model="searchInput"
                  dense
                  outlined
                  flat
                  solo
                  placeholder="Suche..."
                  clearable
                  inputmode="search"
                  append-icon="mdi-menu-down"
                  @click:append="toogleIntervalContainer()"
               />
               <v-fade-transition>
                  <div v-show="showIntervalContainer === true">
                     <v-row
                        d-flex
                        wrap
                     >
                        <v-col
                           :cols="12"
                           lg="6"
                           class="py-0"
                        >
                           <v-text-field
                              label="Buchungen von"
                              outlined
                              dense
                              v-model="period.from"
                              type="date"
                           />
                        </v-col>
                        <v-col
                           :cols="12"
                           lg="6"
                           class="py-0"
                        >
                           <v-text-field 
                              label="Buchungen bis"
                              outlined
                              dense
                              v-model="period.to"
                              type="date"
                           />
                        </v-col>
                     </v-row>
                     <v-row>
                        <v-col
                           class="pt-0"
                        >
                           <v-btn
                              block
                              outlined
                              color="primary"
                              @click.native="fetchData()"
                           >
                              <v-icon>
                                 mdi-magnify
                              </v-icon>
                              suchen
                           </v-btn>
                        </v-col>
                     </v-row>
                  </div>
               </v-fade-transition>
            </v-card-subtitle>
            <v-divider />
            <v-row>
               <v-col
                  :cols="12"
                  v-for="section in documentGrid"
                  :key="section.date"
                  class="py-1"
               >
                  <v-card-subtitle
                     class="py-0"
                  >
                     <v-row class="py-0">
                        <v-col
                           class="py-0"
                        >
                           {{ formatedDate(section.date) }}
                        </v-col>
                        <v-col class="text-right py-0">
                           {{
                              new Intl.NumberFormat("de-DE", format).format(section.balance)
                           }}
                        </v-col>
                     </v-row>
                  </v-card-subtitle>
                  <v-list
                     class="py-1"
                  >
                     <DocItem
                        v-for="doc in section.documents"
                        :key="doc.id"
                        :doc="doc"
                        @click.native="showUpdateDialog(doc)"
                  />
                  <v-divider />
                  </v-list>
               </v-col>
            </v-row>
            <v-card-text
               class="openingBalance"
            >
               <v-row>
                  <v-col
                     :cols="8"
                  >
                     Saldovortrag
                  </v-col>
                  <v-col
                     :cols="4"
                     class="text-right px-0"
                  >
                     {{
                        new Intl.NumberFormat("de-DE", format).format(openingBalance)
                     }}
                  </v-col>
               </v-row>
            </v-card-text>

            <v-card-actions>
               <v-btn
                  block
                  depressed
                  @click.native="fetchMoreDocuments()"
               >
                  mehr Umsätze
               </v-btn>
            </v-card-actions>
         </v-card>
      <UpdateDocument
         :dialog ="dialog"
         :doc="selectedDoc"
         @updateGrid="fetchData"
         @dialogClosed="toggleDialog()"
      />
   </div>
</template>

<script lang="ts">
import { Vue, Component, Watch } from 'vue-property-decorator'
import { numberFormat } from '../shared/document'
import DocItem from '@/components/documents/DocumentItem.vue'
import UpdateDocument from '@/components/documents/UpdateDocument.vue'
import AccountSummary from '@/components/documents/AccountSummary.vue'
import moment from 'moment'

@Component({
   components: {
      DocItem, UpdateDocument, AccountSummary
   }
})
export default class Documents extends Vue{

   private loading = false;
   private showIntervalContainer = false;

   // define format for numbers
   private format = numberFormat;

   // Account Summary
   private account: Account = {} as Account
   private searchInput = ''

   // Documents
   private documents: Array<Doc> = [];
   private openingBalance = 0
   private period = {
      from: moment().startOf("month").subtract(1, "month").format("YYYY-MM-DD"),
      to: moment().endOf("month").format("YYYY-MM-DD")
   }

   // Update dialog
   private selectedDoc: Doc = {} as Doc
   private dialog = false;

   async beforeMount() {
      await this.fetchData()
      this.loading = false
   }

   async fetchData() {
      Promise.all([
         this.fetchDocuments(this.selectedAccount),
         this.fetchAccount(this.selectedAccount),
         this.fetchOpeningBalance(this.selectedAccount)
      ])
   }

   async fetchDocuments(account: number){
      const res = await this.axios.get(`/documents/${account}/${this.period.from}/${this.period.to}`)
      this.documents = res.data.map((item: Doc) => {return item as Doc})
   }

   async fetchMoreDocuments() {
      this.period.from = moment(this.period.from).subtract(1, "month").format("YYYY-MM-DD")
      await Promise.all([
         this.fetchDocuments(this.selectedAccount),
         this.fetchOpeningBalance(this.selectedAccount)
      ])
   }

   async fetchAccount(account: number){
      const res = await this.axios.get(`/accounts/details/${account}`)
      this.account = res.data[0] as Account
   }

   async fetchOpeningBalance(account: number) {
      const res = await this.axios.get(`/accounts/balance/${account}/${this.period.from}/${this.period.to}`)
      this.openingBalance = res.data[0].opening
   }

   get selectedAccount(): number {
      return Number(this.$route.params.account)
   }

   private showUpdateDialog(doc: Doc) {
      this.selectedDoc = doc
      this.toggleDialog()
   }

   private toggleDialog() {
      this.dialog = !this.dialog
   }

   private toogleIntervalContainer() {
      this.showIntervalContainer = !this.showIntervalContainer
   }

   // eslint-disable-next-line
   get documentGrid(): Record<string, any> {
      // eslint-disable-next-line
      const grid: Record<string, any>[] = []
      const data = this.filter(this.documents)
      let dates = [... new Set(data.map(x => x.date))]
      let saldo = this.openingBalance

      dates = dates.sort((a, b) => {
         return new Date(a).getTime() - new Date(b).getTime()
      })

      dates.forEach( date => {
         const obj = {
            date: date,
            documents: data.filter( doc => doc.date === date),
            balance: data.filter(doc => doc.date === date).map(doc => doc.amount).reduce((prev, current) => { return prev + current}) + saldo
         }
         saldo = obj.balance
         grid.push(obj)
      })
      return grid.sort((a, b) => {
         return new Date(b.date).getTime() - new Date(a.date).getTime()
      })
   }

   private filter(data: Doc[]): Doc[] {

      if (!this.searchInput){
         return data
      }

      const searchArguments = this.searchInput.split(';')
      const filter: Doc[] = []

      searchArguments?.forEach(arg => {
         if (arg === '') return

         if (this.checkDate(arg) === true) {
         filter.push(... data.filter((doc: Doc) => {
               return moment(doc.date).format("DD.MM.YYYY").toString() === arg && filter.indexOf(doc) === -1
            }))
         }

         if (this.checkNumber(arg)) {
            const argAsNum = arg.replace(/,(\d+)$/,'.$1')
            filter.push(... data.filter((doc: Doc) => {
               return Math.abs(doc.amount) === Number(argAsNum) && filter.indexOf(doc) === -1
            }))
         }

         filter.push(... data.filter((doc: Doc) => {
            return (
               (
                  doc.name.toLowerCase().includes(arg.toLowerCase()) ||
                  doc.description.toLowerCase().includes(arg.toLowerCase()) ||
                  doc.category?.toLowerCase().includes(arg.toLowerCase()) ||
                  doc.iban.toLowerCase().includes(arg.toLowerCase())
               )
            && filter.indexOf(doc) === -1 )
         }))
      })

      return filter

   }

   @Watch("searchInput")
   preventNull() {
      this.searchInput = this.searchInput ?? ''
   }

   private checkDate(possibelDate: string): boolean {
      return moment(possibelDate, 'DD.MM.YYYY').isValid()
   }

   // eslint-disable-next-line
   private checkNumber(possibleNumber: any): boolean {
      const argAsNum = possibleNumber.replace(/,(\d+)$/,'.$1')
      return !isNaN(argAsNum)
   }

   private formatedDate(date: Date): string {
      return moment(date).format("DD.MM.YYYY")
   }
}
</script>

<style scoped>
   .openingBalance{
      padding-top: 0px;
      padding-bottom: 0px;
      font-size: 16px;
   }
</style>