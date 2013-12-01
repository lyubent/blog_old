---
layout: post
title: "Bootstraping cassandra via VPN"
date: 2013-12-01 02:45:09 +0200
comments: true
categories: [cassandra, vpn, networking]
---

The CassTor application relies on VPN allowing you to bootstrap as if you are part of a LAN network from remote machines. Today a problem was encountered for the mechanism of detecting which IP was to be placed in cassandra's configuration (for the listen_address option).

The current code structure first checks if a VPN connection is available, then looks for a Ethernet connection and finally if the previous two are unavailable, a WIFI connection is used.

<br/>
<script src="https://gist.github.com/lyubent/5293431.js"></script>
<br/>

On a side note, some usability testing was carried out. The aim was to workout which of the three connection types was used most frequently. With the help of ten volunteers, it was established that most liked the VPN connection best as CassTor always managed to bootstrap straight away, while with WIFI and Ethernet connections there were occasionally timeouts:

<script src="http://code.highcharts.com/highcharts.js"></script>
<div id="container" style="width: 320px; height: 320px"></div>
<script>
var chart;$(document).ready(function(){Highcharts.setOptions({colors:["#FF9655","#058DC7","#6AF9C4"]});chart=new Highcharts.Chart({series:[{name:"Values",type:"pie",sliced:true,data:[["VPN",7],["Ethernet",3],["Wifi",0]],pointWidth:15,color:"#C6D9E7",borderColor:"gray",shadow:true}],title:{text:" "},legend:{style:{left:"auto",bottom:"auto",right:"auto",top:"auto"}},chart:{renderTo:"container"},credits:{enabled:false}})})
</script>

The above pretty much concludes that the current approach assumes too much about the network infrastructure that casstor can handle. One possibility is to switch to a Network topology keyspace, but it could be argued that the reliability VPN provides should be part of the prerequisites of the application.


