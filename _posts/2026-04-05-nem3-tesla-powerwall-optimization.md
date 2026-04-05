---
layout: post
title: "Home Energy Economics: Optimizing Tesla Powerwalls for NEM 3.0"
date: 2026-04-05
description: "Surviving California's Net Billing Tariff with dual Powerwall 3s, a Model 3, and the NetZero app."
---

California's [NEM 3.0](https://www.tesla.com/support/energy/solar-panels/learn/net-billing#) changed the math for home solar. Under the old rules, the grid acted as a giant 1:1 battery. You overproduced during the day, pushed power to PG&E, and pulled it back at night for roughly the same price. 

Under NEM 3.0 (the Net Billing Tariff), export rates plummeted to around 4 to 8 cents per kWh, while import rates during peak hours stay stubbornly above 50 cents. The only viable path today is absolute self-consumption. If you send power to the grid, you lose.

I got Permission to Operate (PTO) in late December 2025 for a new system at my house in Benicia: 13.34 kW of [REC 460W](https://www.recgroup.com/en/rec-alpha-pure-rx) panels paired with two [Tesla Powerwall 3](https://www.tesla.com/powerwall)s. The dual Powerwalls give me 23 kW of AC inverter capacity and 27 kWh of storage. The challenge became orchestrating this hardware alongside my 2021 Tesla Model 3 Long Range (~75 kWh of storage) to keep grid imports as close to zero as possible.

![REC 460W solar panels installed on a residential roof](/public/images/nem3-rec-panels.webp)

## The Problem: EVs vs. House Loads

My commute dictates my energy needs. I typically drive into San Francisco twice during the week, which is a 74-mile round trip. The other five days of the week involve about 30 miles of local driving total.

If I plug the car in and it immediately starts pulling 11 kW, two bad things happen. First, it drains the Powerwalls before the sun goes down. Second, once the batteries are empty, the car starts pulling expensive peak grid power. The native Tesla app lacks the nuanced logic to say "charge the house first, then the car."

## The Fix: NetZero App and SOC Thresholds

To fix this, I handed control over to the [NetZero](https://www.netzero.energy/) app. It acts as an orchestrator between the Powerwalls and the vehicle. 

![NetZero app showing Powerwall priority set to 90% before EV charging begins](/public/images/nem3-netzero-settings.webp)

I configured NetZero with a strict rule: the Powerwalls must reach 90% State of Charge (SOC) before the EV is allowed to charge. 

Here is how a typical sunny day plays out:
1. Morning sun wakes up the system and covers base house loads.
2. Excess solar flows directly into the Powerwalls.
3. Once the Powerwalls cross 90%, NetZero signals the Model 3 to start charging, dynamically matching its charge rate to the remaining excess solar.
4. When the sun goes down, the car stops charging. The house runs off the top 90% of the Powerwalls through the expensive 4 PM to 9 PM peak window and through the night.

## Setting the Reserve to 5%

I set my Powerwall reserve to 5%. Hoarding energy in a battery defeats the purpose of time-of-use arbitrage. I bought 27 kWh of storage to cycle it every single day to offset expensive evening power. A 5% reserve leaves just enough to keep the lights and internet on during a brief blackout, but puts the rest of the battery to work.

## Long-Term Monitoring: InfluxDB and Grafana

To make sure this orchestration actually works, I pipe the home energy telemetry into [InfluxDB](https://www.influxdata.com/) and visualize it with [Grafana](https://grafana.com/). 

![Grafana power flow dashboard showing solar charging the battery, then the EV pulling the excess once the Powerwalls hit 90%](/public/images/nem3-grafana-flow.webp)

This dashboard tracks system performance, self-consumption ratios, and battery cycling over time. It provides a much clearer picture of historical trends than the native Tesla app. More importantly, it makes it easy to spot when a configuration needs tweaking, especially as the seasons change and solar production dips.



## The Results

With this logic in place, my PG&E bill is essentially reduced to the fixed connection fee and Non-Bypassable Charges (NBCs). However, because I am in [Marin Clean Energy (MCE)](https://www.mcecleanenergy.org/) territory, any actual grid imports during the dark winter months are still hit with MCE generation fees on top of PG&E delivery rates. Avoiding those imports entirely is the only way to win.

![NetZero Today dashboard showing solar production entirely consumed by the home, EV, and battery with zero grid usage](/public/images/nem3-netzero-dashboard.webp)

The system banks energy on the weekends and early in the week, loading up the car for my mid-week commutes. Any hardware investment in NEM 3.0 requires batteries, but extracting the actual ROI requires automation to keep grid exports near zero.
