<template>
  <div id="chart" style="width:100%; height:400px;"></div>
</template>

<style>

</style>

<script>
import $ from "jquery";
import moment from 'moment'

export default {
  data(){
    return {
      mainCurrency: 'CAD',
      historyJson: 'https://shakepay.github.io/programming-exercise/web/transaction_history.json',
      ratesHistoryJson:{
        'CAD_BTC': 'https://shakepay.github.io/programming-exercise/web/rates_CAD_BTC.json',
        'CAD_ETH': 'https://shakepay.github.io/programming-exercise/web/rates_CAD_ETH.json'
      },
      rates: {
        "CAD_BTC":0.00001919,
        "BTC_CAD":52100.34,
        "CAD_ETH":0.000466801578448052,
        "ETH_CAD":2142.23,
        "USD_BTC":0.00002447,
        "BTC_USD":40859.33,
        "USD_ETH":0.000595225694702788,
        "ETH_USD":1680.03,
        "BTC_ETH":24.31906614785992,
        "ETH_BTC":0.04112,
        "CAD_USD":0.78,
        "USD_CAD":1.27}
    }
  },
  async mounted(){
    let ratesHistory = {}
    for (let pair of Object.keys(this.ratesHistoryJson)){
      const pairRatesURL = this.ratesHistoryJson[pair]
      const pairRates = await $.getJSON(pairRatesURL)
      const processedPairRates = []
      for (let i=0; i<pairRates.length; i++){
        const rateRecord = pairRates[i]
        processedPairRates.push({...rateRecord, createdAt: moment(rateRecord.createdAt).unix()})
      }

      ratesHistory[pair] = processedPairRates
    }
    console.log(ratesHistory)
    return
    function getRate(pair, timeStr, lastIndex){
      const time = moment(timeStr).unix()
      for (const [index, entry] of ratesHistory.entries()){
      }
    }

    let history = (await $.getJSON(this.historyJson)).reverse()

    const graphEntries = []
    for (let i=0; i<history.length; i++){
      const record = history[i]
      const lastEntry = graphEntries[graphEntries.length-1]
      const lastAmount = lastEntry ? lastEntry.y : 0



      let newAmount = lastAmount

      // as some value might be lost during conversion due to fees, a conversion cannot be considered as having no
      // impact on the net worth
      if (record.type  === 'conversion'){

        let pair = `${record.from.currency}_${record.to.currency}`

        // compute how much has been sold during conversion
        let fromAmount = record.from.amount
        if (record.from.currency !== this.mainCurrency){
          fromAmount *= this.rates[pair]
        }

        // compute how much has been bougth during conversion
        let toAmount = record.to.amount
        if (record.to.currency !== this.mainCurrency){
          toAmount /= this.rates[pair]
        }

        // compute new amount after conversion
        newAmount = newAmount - fromAmount + toAmount


      } else {
        // handle record amount conversion if operation has not been done in main currency
        let recordAmount = record.amount
        if (record.currency !== this.mainCurrency){
          const pair = `${record.currency}_${this.mainCurrency}`
          recordAmount *= this.rates[pair]
        }

        // compute new amount based on record operation direction
        const amountFactor = record.direction === 'credit' ? 1 : -1
        newAmount += (recordAmount * amountFactor)
      }

      const newEntry = {
        x: record.createdAt,
        y: newAmount,
        type: `${record.type}-${record.direction}`
      }
      graphEntries.push(newEntry)
    }
    console.log(graphEntries)

  }
}
</script>
