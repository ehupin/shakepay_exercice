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
      staticRates: {
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
    // store rates history in object (per pair), pre-process 'createdAt' property for faster lookup then
    let ratesHistory = {}
    for (let pair of Object.keys(this.ratesHistoryJson)){
      const pairRatesURL = this.ratesHistoryJson[pair]
      const pairRates = await $.getJSON(pairRatesURL)

      // convert 'createdAt' string property to a unix time number
      const processedPairRates = []
      for (let i=0; i<pairRates.length; i++){
        const rateRecord = pairRates[i]
        processedPairRates.push({...rateRecord, createdAt: moment(rateRecord.createdAt).unix()})
      }

      ratesHistory[pair] = processedPairRates
    }

    function getRate(pair, dateStr, ratesHistoryPairIndex){

      let lookupPair = pair
      let reverseRate = false
      if (!Object.keys(ratesHistory).includes(pair)){
        const pairCurrencies = pair.split('_')
        const reversedPair = pairCurrencies.reverse().join('_')
        if (!Object.keys(ratesHistory).includes(reversedPair)){
          return this.rates(pair)
        }
        lookupPair = reversedPair
        reverseRate = true
      }


      const unixDate = moment(dateStr).unix()
      for (const [index, entry] of ratesHistory[lookupPair].slice(ratesHistoryPairIndex).entries()){
        if (entry.createdAt < unixDate){
          continue
        }
        const rate = reverseRate ? 1/entry.midMarketRate : entry.midMarketRate
        console.log(index)
        return {rate, index}
      }
    }

    let history = (await $.getJSON(this.historyJson)).reverse()

    const lastRateHistoryPairIndex = {}
    const graphEntries = []
    // for (let i=0; i<history.length; i++){
    for (let i=0; i<20; i++){
      const record = history[i]
      const lastEntry = graphEntries[graphEntries.length-1]
      const lastAmount = lastEntry ? lastEntry.y : 0



      let newAmount = lastAmount

      // as some value might be lost during conversion due to fees, a conversion cannot be considered as having no
      // impact on the net worth
      if (record.type  === 'conversion'){


        let pair = `${record.from.currency}_${record.to.currency}`

        // get history rate index for faster lookup
        let ratesHistoryPairIndex = lastRateHistoryPairIndex[pair] -1

        // compute how much has been sold during conversion
        let fromAmount = record.from.amount
        if (record.from.currency !== this.mainCurrency){
          let {rate, ratesHistoryPairIndex} = getRate(pair, record.createdAt, ratesHistoryPairIndex)
          fromAmount *= rate
        }

        // compute how much has been bougth during conversion
        let toAmount = record.to.amount
        if (record.to.currency !== this.mainCurrency){
          let {rate, ratesHistoryPairIndex} = getRate(pair, record.createdAt, ratesHistoryPairIndex)
          fromAmount /= rate
        }

        // store history rate index for faster future lookups
        lastRateHistoryPairIndex[pair] = ratesHistoryPairIndex

        // compute new amount after conversion
        newAmount = newAmount - fromAmount + toAmount


      } else {

        // handle record amount conversion if operation has not been done in main currency
        let recordAmount = record.amount
        if (record.currency !== this.mainCurrency) {
          const pair = `${record.currency}_${this.mainCurrency}`

          // get history rate index for faster lookup
          let ratesHistoryPairIndex = lastRateHistoryPairIndex[pair] -1

          let {rate, index} = getRate(pair, record.createdAt, ratesHistoryPairIndex)
          recordAmount *= rate

          // store history rate index for faster future lookups
          lastRateHistoryPairIndex[pair] = index
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
