<template>
  <div id="chart" style="width:100%; height:400px;"></div>
</template>

<style>

</style>

<script>
import $ from "jquery";

export default {
  data(){
    return {
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
    for (let pair of object.keys(this.ratesHistoryJson)){
      const pairRatesURL = this.ratesHistoryJson[pair]
      const pairRates = await $.getJSON(pairRatesURL)
      ratesHistory[pair] = pairRates
    }

    let history = (await $.getJSON(this.historyJson)).reverse()

    const graphEntries = []
    for (let i=0; i<history.length; i++){
      const record = history[i]
      const lastEntry = graphEntries[graphEntries.length-1]
      const lastAmount = lastEntry ? lastEntry.y : 0



      let newAmount = lastAmount
      if (record.type  === 'conversion'){

        let pair = `${record.from.currency}_${record.to.currency}`

        let fromAmount = record.from.amount
        if (record.from.currency !== 'CAD'){
          fromAmount *= this.rates[pair]
        }

        let toAmount = record.to.amount
        if (record.to.currency !== 'CAD'){
          toAmount *= this.rates[pair]
        }

        newAmount = newAmount - fromAmount + toAmount




      } else {
        const amountFactor = record.direction === 'credit' ? 1 : -1
        newAmount += (record.amount * amountFactor)
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
