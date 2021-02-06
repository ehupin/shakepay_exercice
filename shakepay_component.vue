<template>
  <div id="chart" style="width:100%; height:400px;"></div>
</template>

<style>

</style>

<script>
import $ from "jquery";
import moment from 'moment'
import Highcharts from 'highcharts/highstock'

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
        "USD_CAD":1.27
      },
      ratesHistory: {}
    }
  },
  async mounted(){

    // store rates history in object (per pair), pre-process 'createdAt' property for faster lookup then
    for (let pair of Object.keys(this.ratesHistoryJson)){
      const pairRatesURL = this.ratesHistoryJson[pair]
      const pairRates = await $.getJSON(pairRatesURL)

      // convert 'createdAt' string property to a unix time number
      const processedPairRates = []
      for (let i=0; i<pairRates.length; i++){
        const rateRecord = pairRates[i]
        processedPairRates.push({...rateRecord, createdAt: moment(rateRecord.createdAt).unix()})
      }
      this.ratesHistory[pair] = processedPairRates
    }

    // get all transactions history
    let history = (await $.getJSON(this.historyJson)).reverse()
    history = history.splice(0,100)

    const lastRateHistoryPairIndex = {}
    const graphEntries = []
    for (let i=0; i<history.length; i++){
    // for (let i=0; i<30; i++){
      const record = history[i]
      const lastEntry = graphEntries[graphEntries.length-1]
      const lastAmount = lastEntry ? lastEntry.y : 0



      let newAmount = lastAmount
      let description = ''
      // as some value might be lost during conversion due to fees, a conversion cannot be considered as having no
      // impact on the net worth
      if (record.type  === 'conversion'){


        let pair = `${record.from.currency}_${record.to.currency}`

        // get history rate index for faster lookup
        let ratesHistoryPairIndex = lastRateHistoryPairIndex[pair] -1

        // compute how much has been sold during conversion
        let fromAmount = record.from.amount
        let pairRate
        if (record.from.currency !== this.mainCurrency){
          let {rate, ratesHistoryPairIndex} = this.getRate(pair, record.createdAt, ratesHistoryPairIndex)
          fromAmount *= rate
          pairRate = rate
        }

        // compute how much has been bougth during conversion
        let toAmount = record.to.amount
        if (record.to.currency !== this.mainCurrency){
          let {rate, ratesHistoryPairIndex} = this.getRate(pair, record.createdAt, ratesHistoryPairIndex)
          fromAmount /= rate
          pairRate = rate
        }

        // store history rate index for faster future lookups
        lastRateHistoryPairIndex[pair] = ratesHistoryPairIndex

        // compute new amount after conversion
        newAmount = newAmount - fromAmount + toAmount
        description = `Conversion from `
        description += `${record.from.amount} ${record.from.currency} `
        description += `to ${record.to.amount} ${record.to.currency}`
        description += `at ${pairRate} on ${pair}`


      } else {

        // handle record amount conversion if operation has not been done in main currency
        let recordAmount = record.amount
        if (record.currency !== this.mainCurrency) {
          const pair = `${record.currency}_${this.mainCurrency}`

          // get history rate index for faster lookup
          let ratesHistoryPairIndex = lastRateHistoryPairIndex[pair] -1

          let {rate, index} = this.getRate(pair, record.createdAt, ratesHistoryPairIndex)
          recordAmount *= rate

          // store history rate index for faster future lookups
          lastRateHistoryPairIndex[pair] = index
        }


        // compute new amount based on record operation direction
        const amountFactor = record.direction === 'credit' ? 1 : -1
        newAmount += (recordAmount * amountFactor)

        description = `${record.direction} of ${record.amount} ${record.currency}`
        if (record.currency !== this.mainCurrency) {
          description += `(${recordAmount.toFixed(3)} ${this.mainCurrency})`
        }
      }

      // create new graph entry
      const newEntry = {
        x: moment(record.createdAt).unix(),
        y: newAmount,
        name:`${description} #${i}`
      }
      graphEntries.push(newEntry)
    }
    console.log(graphEntries)

    const chartData = {
      series:[{
        name: 'net worth',
        data: graphEntries
      }],
      xAxis: {
        type: 'datetime'
      },
    }
    Highcharts.stockChart('chart', chartData)

  },
  methods: {
    getRate(pair, dateStr, ratesHistoryPairIndex){

      // as history pair rates are defined only in one way (i.e. CAD_BTC and not BTC-CAD)
      // lets first find the right pair to lookup, and know if returned rade must be reversed
      let lookupPair = pair
      let reverseRate = false
      if (!Object.keys(this.ratesHistory).includes(pair)){
        const pairCurrencies = pair.split('_')
        const reversedPair = pairCurrencies.reverse().join('_')
        if (!Object.keys(this.ratesHistory).includes(reversedPair)){
          return this.rates(pair)
        }
        lookupPair = reversedPair
        reverseRate = true
      }

      // loop over rates history, starting from given index
      // (when a rate is returned, the history index of this rate is also return to allow future request to not loop
      // from the beginning of the history - as they are usalyy made in chronological order)
      const unixDate = moment(dateStr).unix()
      for (const [index, entry] of this.ratesHistory[lookupPair].slice(ratesHistoryPairIndex).entries()){
        if (entry.createdAt < unixDate){
          continue
        }
        const rate = reverseRate ? 1/entry.midMarketRate : entry.midMarketRate
        return {rate, index}
  }
}
  }
}
</script>
