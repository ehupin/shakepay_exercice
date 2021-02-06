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
    let history = await $.getJSON(this.historyJson)
    const graphEntries = []

    for (let i=0, i<history.length, i++){

    }

  }
}
</script>
