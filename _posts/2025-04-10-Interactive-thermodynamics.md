---
layout: post
title:  "Interactive thermodynamics"
date:   2025-04-10 00:00:49 +0100
categories: viz
summary: ""
image: "/assets/images/interactive-ptes.png"
image_alt: "Image shows a temperature-entropy diagram with the charge and discharge cycles of a pumped thermal energy storage system."
usemathjax: false
custom_css: "interactive-thermo"
---

## Learning from a model, one slider at a time

<video style="width: 100%; height: auto;" autoplay loop muted controls>
  <source src="{{site.baseurl}}/assets/images/interactive-ptes.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

Even models that are conceptually simple, but are controlled by multiple parameters, can be challenging to interpret and learn from. One way to address this is to visualise the model dynamically so that changes to the governing parameters are immediately reflected in the visual representation of the model. The humble range slider allows the user to adjust one parameter while keeping the others fixed. A grid of sliders shows the state of all relevant parameters and enables the model to be explored, one slider at a time. 

## Pumped thermal energy storage (PTES)
PTES is a concept for storing excess electrical energy for use at a later time. PTES stores energy as heat, either in a solid or liquid medium, and is suitable for storing energy for multiple days. During charge, the device operates as a heat pump: electrical work is supplied to the system, raising the temperature of the hot store and reducing the temperature of the cold store. During discharge, the device becomes a heat engine (the hot store temperature decreases, the cold store temperature increases, and electrical work is output). More information is available on the [thermodynamics of PTES](https://doi.org/10.1016/j.applthermaleng.2012.03.030), [PTES using liquid thermal storage](https://doi.org/10.1016/B978-0-12-819723-3.00054-8), and on other [thermomechanical energy storage systems](https://iopscience.iop.org/article/10.1088/2516-1083/abdbba). 

## Interactive model
<div id="target" width="100%" ></div>

This chart is an interactive thermodynamic cycle diagram (temperature vs entropy) of the working fluid (air) for both the charge and discharge cycles. Various output metrics are shown to the right of the diagram. Both the cycle diagram and the output metrics respond to changes in the input parameters at the bottom of the chart. 

## Implementation 
I have used [d3](https://d3js.org) to render the cycle diagram (svg elements) and to add the inputs and outputs. The cycle diagram is deleted (all elements removed from the DOM) and redrawn each time an input is changed. The thermodynamic analysis itself is also coded in JavaScript



<script type="module">

import * as d3 from "https://cdn.jsdelivr.net/npm/d3@7/+esm";

class PTES {
    constructor(options) {
        options = options || {};
        this.TambK = options.TambK || 298.15; // in K
        this.THotMaxC = options.THotMaxC || 500; // in C
        this.THotMaxK = this.THotMaxC + 273.15; // in K
        this.THotMinC = options.THotMinC || 300; // in C
        this.THotMinK = this.THotMinC + 273.15; // in K
        this.TColdMaxFactor = options.TColdMaxFactor || 0.;
        this.TColdMaxK = this.TambK + this.TColdMaxFactor * (this.THotMaxK - this.TambK);
        this.pLow = options.pLow || 101325; // in Pa
        this.dischargePower = options.dischargePower || 1; // in MW
        this.dischargeDuration = options.dischargeDuration || 5; // in hours
        if (options.samePrat == undefined) {
            this.samePrat = true;
        } else {
            this.samePrat = options.samePrat;
        }
        this.etaPoly = options.etaPoly || 0.9;
        this.gamma = options.gamma || 1.4;
        this.rGas = options.rGas || 287; // in J/kgK
        this.cp = options.cp || 1005; // in J/kgK
        this.Tref = 298.15;
        this.pref = 101325;
        this.margin = options.margin || {top: 20, right: 20, bottom: 40, left: 50};
        this.xScale = d3.scaleLinear();
        this.yScale = d3.scaleLinear();

        this.inputSliders = [
            { title: 'Hot Store Max (K)', id: 'hot-store-max', min: 200, max: 1000, step: 1, value: 773, varName: 'THotMaxK' },
            { title: 'Hot Store Min (K)', id: 'hot-store-min', min: 200, max: 1000, step: 1, value: 573, varName: 'THotMinK' },
            { title: 'Cold Store Max (K)', id: 'cold-store-max', min: 200, max: 500, step: 1, value: 273, varName: 'TColdMaxK' },
            { title: 'Turbo eta poly', id: 'eta-poly', min: 0.8, max: 1, step: 0.001, value: 0.9, varName: 'etaPoly' },
            { title: 'Low pressure (Pa)', id: 'low-pressure', min: 50000, max: 1000000, step: 100, value: 101325, varName: 'pLow' },
            { title: 'Discharge power (MW)', id: 'discharge-power', min: 0, max: 100, step: 0.1, value: 1, varName: 'dischargePower' },
            { title: 'Discharge duration (h)', id: 'discharge-duration', min: 0, max: 12, step: 0.1, value: 5, varName: 'dischargeDuration' }
        ]

        this.outputs = [
            { title: 'RTE', id: 'rte', varName: "etaRoundTrip", fmt: ".2f" },
            { title: "Charge pRat", id: "pRatCharge", varName: "pRatCharge", fmt: ".2f" },
            { title: "Discharge pRat", id: "pRatDischarge", varName: "pRatDischarge", fmt: ".2f" },
            { title: "Heat pump COP", id: "heatPumpCOP", varName: "heatPumpCOP", fmt: ".2f" },
            { title: "Heat engine eff", id: "heatEngineEta", varName: "heatEngineEta", fmt: ".2f" },
            { title: "Work ratio", id: "workRatio", varName: "workRatio", fmt: ".2f" },
            { title: "Heat to work ratio", id: "heatWorkRatio", varName: "heatWorkRatio", fmt: ".2f" },
            { title: "Discharge turbine power (MW)", id: "dischargeTurbinePower", varName: "dischargeTurbinePower", fmt: ".2f" },
            { title: "Discharge compressor power (MW)", id: "dischargeCompressorPower", varName: "dischargeCompressorPower", fmt: ".2f" },
            { title: "Discharge mass flow (kg/s)", id: "cycleMassFlow", varName: "cycleMassFlow", fmt: ".2f" },
            { title: "Charge compressor inlet vol flow (m3/s)", id: "chargeCompressorInletVolFlow", varName: "chargeCompInletVolumeFlow", fmt: ".2f" },
            { title: "Hot store tank volume (m3)", id: "hotStoreTankVolume", varName: "hotStoreTankVolume", fmt: ".2f" },
            { title: "Hot store tank diameter (m)", id: "hotStoreTankDiameter", varName: "hotStoreTankDiameter", fmt: ".2f" }
        ]

        this.cycleLayout = options.cycleLayout || {width: 400, height: 400 };

        // fluids data from McTigue liquid PTES paper
        this.hotStoreFluids = [
            {
                name: "Water",
                Tmin: 0,
                Tmax: 100,
                TminK: 273.15,
                TmaxK: 373.15,
                cp: 4180,
                density: 980,
                k: 0.65,
                viscosity: 0.55,
                cost: 0.01,
                hazard: "-"
            },
            {
                name: "Mineral oil",
                Tmin: 10,
                Tmax: 316,
                TminK: 283.15,
                TmaxK: 589.15,
                cp: 2470,
                density: 767,
                k: 0.12,
                viscosity: 0.95,
                cost: 1.3,
                hazard: "F(1) H(1) I(0)"
            },
            {
                name: "Synthetic oil",
                Tmin: 12,
                Tmax: 400,
                TminK: 285.15,
                TmaxK: 673.15,
                cp: 2180,
                density: 909,
                k: 0.11,
                viscosity: 0.38,
                cost: 1.0,
                hazard: "F(1) H(2) I(0)"
            },
            {
                name: "Silicone oil",
                Tmin: -40,
                Tmax: 385,
                TminK: 233.15,
                TmaxK: 658.15,
                cp: 1920,
                density: 773,
                k: 0.10,
                viscosity: 1.05,
                cost: 50,
                hazard: "F(1) H(1) I(0)"
            },
            {
                name: "Nitrate molten salt",
                Tmin: 230,
                Tmax: 565,
                TminK: 503.15,
                TmaxK: 838.15,
                cp: 1510,
                density: 1860,
                k: 0.51,
                viscosity: 2.00,
                cost: 0.8,
                hazard: "F(0) H(1) I(0)"
            },
            {
                name: "Chloride molten salt",
                Tmin: 450,
                Tmax: 750,
                TminK: 723.15,
                TmaxK: 1023.15,
                cp: 1030,
                density: 1460,
                k: 0.43,
                viscosity: 2.80,
                cost: 0.6,
                hazard: "F(0) H(1) I(0)"
            }
        ];
        this.coldStoreFluids = [
            {
                name: "N-propane",
                Tmin: -188,
                Tmax: -42,
                TminK: 85.15,
                TmaxK: 231.15,
                cp: 2020,
                density: 660,
                k: 0.17,
                viscosity: 0.55,
                cost: 0.5,
                hazard: "F(4) H(2) I(0)"
            },
            {
                name: "Isopentane",
                Tmin: -160,
                Tmax: 28,
                TminK: 113.15,
                TmaxK: 301.15,
                cp: 1850,
                density: 710,
                k: 0.14,
                viscosity: 0.56,
                cost: 1.2,
                hazard: "F(4) H(1) I(0)"
            },
            {
                name: "N-Hexane",
                Tmin: -95,
                Tmax: 69,
                TminK: 178.15,
                TmaxK: 342.15,
                cp: 2200,
                density: 660,
                k: 0.13,
                viscosity: 0.50,
                cost: null, // No value given
                hazard: "F(3) H(1) I(0)"
            },
            {
                name: "Ethanol",
                Tmin: -115,
                Tmax: 78,
                TminK: 158.15,
                TmaxK: 351.15,
                cp: 2460,
                density: 850,
                k: 0.18,
                viscosity: 6.59,
                cost: 0.5,
                hazard: "F(3) H(2) I(0)"
            },
            {
                name: "Methanol",
                Tmin: -98,
                Tmax: 65,
                TminK: 175.15,
                TmaxK: 338.15,
                cp: 2260,
                density: 850,
                k: null, // No value given
                viscosity: 1.83,
                cost: 0.3,
                hazard: "F(3) H(1) I(0)"
            },
            {
                name: "Ethylene-glycol mixture",
                Tmin: -36,
                Tmax: 160,
                TminK: 237.15,
                TmaxK: 433.15,
                cp: 3700,
                density: 1020,
                k: 0.38,
                viscosity: 0.74,
                cost: 1.2,
                hazard: "F(1) H(2) I(0)"
            }
        ];
          
      
        
        this.chargeStates = [];
        this.dischargeStates = [];
        this.calculateStates();
    }

    getChargeStateByStation(station) {
        return this.chargeStates.find( state => state.station == station );
    }

    getDischargeStateByStation(station) {
        return this.dischargeStates.find( state => state.station == station );
    }

    calculateStates() {
        // 
        // Assumptions:
        // stations as per McTigue TechnoEconomic PTES paper
        // irreversible turbomachinery with etaPoly
        // perfect HX with no pressure drop and effectiveness = 1
        //
        const entropy = (p,T) => {
            return this.cp*Math.log(T/this.Tref) - this.rGas*Math.log(p/this.pref);
        };

       
        let state;
        this.chargeStates = [];
        this.dischargeStates = [];

        //
        // charging
        //
        const TRatComp = this.THotMaxK / this.THotMinK;
        this.pRatCharge = TRatComp**( (this.etaPoly*this.gamma)/(this.gamma-1) );
        const TRatTurb = this.pRatCharge**( ((this.gamma-1)*this.etaPoly)/this.gamma );
        this.pHighCharge = this.pLow*this.pRatCharge;
        state = {
            station : "1",
            label : "Compressor inlet",
            p : this.pLow,
            TinK : this.THotMaxK / TRatComp,
            s : entropy(this.pLow, this.THotMaxK/TRatComp)
        }
        this.chargeStates.push(state);
        state = {
            station : "2",
            label : "Compressor outlet",
            p : this.pHighCharge,
            TinK : this.THotMaxK,
            s : entropy(this.pHighCharge, this.THotMaxK)
        }
        this.chargeStates.push(state);
        state = {
            station : "2a",
            label : "Hot store outlet",
            p : this.pHighCharge,
            TinK : this.THotMinK,
            s : entropy(this.pHighCharge, this.THotMinK)
        }
        this.chargeStates.push(state);
        state = {
            station : "3",
            label : "Turbine inlet",
            p : this.pHighCharge,
            TinK : this.TColdMaxK,
            s : entropy(this.pHighCharge, this.TColdMaxK)
        }
        this.chargeStates.push(state);
        state = {
            station : "4",
            label : "Turbine outlet",
            p : this.pLow,
            TinK : this.TColdMaxK / TRatTurb,
            s : entropy(this.pLow, this.TColdMaxK/TRatTurb)
        }
        this.chargeStates.push(state);
        this.TColdMinK = this.TColdMaxK / TRatTurb;
        this.TColdMinC = this.TColdMinK - 273.15;
        state = {
            station : "4a",
            label : "Cold store outlet",
            p : this.pLow,
            TinK : this.TColdMaxK,
            s : entropy(this.pLow, this.TColdMaxK)
        }
        this.chargeStates.push(state);
        this.chargeProcesses = [
            {from: "1", to: "2", label: "Compressor - Charge", labelCol:"#ebb734", path:"straight"},
            {from: "2", to: "2a", label: "Hot store - Charge", labelCol:"#ebb734", path:"curved", colour:"red"},
            {from: "2a", to: "3", label: "Recuperator - Charge", labelCol:"#ebb734", path:"curved"},
            {from: "3", to: "4", label: "Turbine - Charge", labelCol:"#ebb734", path:"straight"},
            {from: "4", to: "4a", label: "Cold store - Charge", labelCol:"#ebb734", path:"curved", colour:"blue"},
            {from: "4a", to: "1", label: "Recuperator - Charge", labelCol:"#ebb734", path:"curved"}
        ];
            
        //
        // discharging
        //
        if (this.samePrat) {
            this.pRatDischarge = this.pRatCharge;
            this.pHighDischarge = this.pLow*this.pRatDischarge;
            state = {
                station : "2",
                label : "Turbine inlet",
                p : this.pHighDischarge,
                TinK : this.THotMaxK,
                s : entropy(this.pHighDischarge, this.THotMaxK)
            };
            this.dischargeStates.push(state);
            state = {
                station : "1",
                label : "Turbine outlet",
                p : this.pLow,
                TinK : this.THotMaxK / TRatTurb,
                s : entropy(this.pLow, this.THotMaxK / TRatTurb)
            };
            this.dischargeStates.push(state);
            state = {
                station : "1b",
                label : "Recuperator pLow inlet",
                p : this.pLow,
                TinK : this.THotMinK,
                s : entropy(this.pLow, this.THotMinK)
            };
            this.dischargeStates.push(state);
            state = {
                station : "1a",
                label : "Recuperator pLow outlet",
                p : this.pLow,
                TinK : this.TColdMinK * TRatComp,
                s : entropy(this.pLow, this.TColdMinK * TRatComp)
            };
            this.dischargeStates.push(state);
            state = {
                station : "4a",
                label : "Cold store inlet",
                p : this.pLow,
                TinK : this.TColdMaxK,
                s : entropy(this.pLow, this.TColdMaxK)
            };
            this.dischargeStates.push(state);
            state = {
                station : "4",
                label : "Compressor inlet",
                p : this.pLow,
                TinK : this.TColdMinK,
                s : entropy(this.pLow, this.TColdMinK)
            };
            this.dischargeStates.push(state);
            state = {
                station : "3",
                label : "Compressor outlet",
                p : this.pHighDischarge,
                TinK : this.TColdMinK * TRatComp,
                s : entropy(this.pHighDischarge, this.TColdMinK * TRatComp)
            };
            this.dischargeStates.push(state);
            state = {
                station : "2a",
                label : "Hot store inlet",
                p : this.pHighDischarge,
                TinK : this.THotMinK,
                s : entropy(this.pHighDischarge, this.THotMinK)
            };
            this.dischargeStates.push(state);
            this.dischargeProcesses = [
                {from: "2", to: "1", label: "Turbine - Discharge", labelCol:"#71eb34", path:"straight"},
                {from: "1", to: "1b", label: "HX to environment - Discharge", labelCol:"#71eb34", path:"curved"},
                {from: "1b", to: "1a", label: "Recuperator - Discharge", labelCol:"#71eb34", path:"curved"},
                {from: "1a", to: "4a", label: "HX to environment - Discharge", labelCol:"#71eb34", path:"curved"},
                {from: "4a", to: "4", label: "Cold store - Discharge", labelCol:"#71eb34", path:"curved", colour:"blue"},
                {from: "4", to: "3", label: "Compressor - Discharge", labelCol:"#71eb34", path:"straight"},
                {from: "3", to: "2a", label: "Recuperator - Discharge", labelCol:"#71eb34", path:"curved"},
                {from: "2a", to: "2", label: "Hot store - Discharge", labelCol:"#71eb34", path:"curved", colour:"red"}
            ];
        } else {
            const TRatTurb = this.THotMaxK / this.THotMinK;
            this.pRatDischarge = TRatTurb**( this.gamma/(this.etaPoly*(this.gamma-1)) );
            const TRatComp = this.pRatDischarge**( (this.gamma-1)/(this.etaPoly*this.gamma) );
            this.pHighDischarge = this.pLow*this.pRatDischarge;
            state = {
                station : "2",
                label : "Turbine inlet",
                p : this.pHighDischarge,
                TinK : this.THotMaxK,
                s : entropy(this.pHighDischarge, this.THotMaxK)
            };
            this.dischargeStates.push(state);
            state = {
                station : "1",
                label : "Turbine outlet",
                p : this.pLow,
                TinK : this.THotMaxK / TRatTurb,
                s : entropy(this.pLow, this.THotMaxK / TRatTurb)
            };
            this.dischargeStates.push(state);
            const TCompOutlet = this.TColdMinK * TRatComp;
            state = {
                station : "1a",
                label : "Recuperator pLow outlet",
                p : this.pLow,
                TinK : TCompOutlet,
                s : entropy(this.pLow, TCompOutlet)
            };
            this.dischargeStates.push(state);
            state = {
                station : "4a",
                label : "Cold store inlet",
                p : this.pLow,
                TinK : this.TColdMaxK,
                s : entropy(this.pLow, this.TColdMaxK)
            };
            this.dischargeStates.push(state);
            state = {
                station : "4",
                label : "Compressor inlet",
                p : this.pLow,
                TinK : this.TColdMinK,
                s : entropy(this.pLow, this.TColdMinK)
            };
            this.dischargeStates.push(state);
            state = {
                station : "3",
                label : "Compressor outlet",
                p : this.pHighDischarge,
                TinK : TCompOutlet,
                s : entropy(this.pHighDischarge, TCompOutlet)
            };
            this.dischargeStates.push(state);
            state = {
                station : "2a",
                label : "Hot store inlet",
                p : this.pHighDischarge,
                TinK : this.THotMinK,
                s : entropy(this.pHighDischarge, this.THotMinK)
            };
            this.dischargeStates.push(state);
            this.dischargeProcesses = [
                {from: "2", to: "1", label: "Turbine - Discharge", labelCol:"#71eb34", path:"straight"},
                {from: "1", to: "1a", label: "Recuperator - Discharge", labelCol:"#71eb34", path:"curved"},
                {from: "1a", to: "4a", label: "HX to environment - Discharge", labelCol:"#71eb34", path:"curved"},
                {from: "4a", to: "4", label: "Cold store - Discharge", labelCol:"#71eb34", path:"curved", colour:"blue"},
                {from: "4", to: "3", label: "Compressor - Dishcarge", labelCol:"#71eb34", path:"straight"},
                {from: "3", to: "2a", label: "Recuperator - Discharge", labelCol:"#71eb34", path:"curved"},
                {from: "2a", to: "2", label: "Hot store - Discharge", labelCol:"#71eb34", path:"curved", colour:"red"}
            ];
        
        }

        //
        // output metrics
        //
        this.wCompCharge = this.cp*(this.getChargeStateByStation("2").TinK - this.getChargeStateByStation("1").TinK);
        this.wTurbCharge = this.cp*(this.getChargeStateByStation("3").TinK - this.getChargeStateByStation("4").TinK);
        this.wCompDischarge = this.cp*(this.getDischargeStateByStation("3").TinK - this.getDischargeStateByStation("4").TinK);
        this.wTurbDischarge = this.cp*(this.getDischargeStateByStation("2").TinK - this.getDischargeStateByStation("1").TinK);
        this.workRatio = this.wCompCharge / this.wTurbCharge;
        this.wNetCharge = this.wCompCharge - this.wTurbCharge;
        this.wNetDischarge = this.wTurbDischarge - this.wCompDischarge;

        let totalHeatCharge = this.cp*(this.getChargeStateByStation("2").TinK - this.getChargeStateByStation("3").TinK);
        totalHeatCharge += this.cp*(this.getChargeStateByStation("1").TinK - this.getChargeStateByStation("4").TinK);
        this.heatWorkRatio = totalHeatCharge / this.wNetCharge;

        this.qHotStoreCharge = this.cp*(this.getChargeStateByStation("2").TinK - this.getChargeStateByStation("2a").TinK);
        
        this.etaRoundTrip = this.wNetDischarge / this.wNetCharge;
        this.heatPumpCOP = this.qHotStoreCharge / this.wNetCharge;
        this.heatEngineEta = this.wNetDischarge / this.qHotStoreCharge;

        this.cycleMassFlow = this.dischargePower * 1e6 / this.wNetDischarge;
        this.dischargeTurbinePower = this.wTurbDischarge * this.cycleMassFlow / 1e6; // in MW
        this.dischargeCompressorPower = this.wCompDischarge * this.cycleMassFlow / 1e6; // in MW

        this.totalQHotStored = this.qHotStoreCharge * this.cycleMassFlow * this.dischargeDuration * 3600; 
        if (this.hotFluid) {
            const hotStoreFluid = this.hotStoreFluids.find( fluid => fluid.name == this.hotFluid );
            this.hotStoreTankVolume = this.totalQHotStored / (hotStoreFluid.density * hotStoreFluid.cp * (this.THotMaxK - this.THotMinK));
            let tankAspectRatio = 0.5;
            this.hotStoreTankDiameter = Math.pow((4 * this.hotStoreTankVolume)/(Math.PI * tankAspectRatio), 1/3);
        } else {
            this.hotStoreTankVolume = "";
            this.hotStoreTankDiameter = "";
        }

        const chargeCompressorInletDensity = this.getChargeStateByStation("1").p / (this.rGas * this.getChargeStateByStation("1").TinK);
        this.chargeCompInletVolumeFlow = this.cycleMassFlow / chargeCompressorInletDensity;

    }

    renderChargeCycle(targetDivId) {
        this.setScales(targetDivId);
        this.renderCycle(this.chargeStates, this.chargeProcesses, targetDivId, 'charge');
    }

    renderDischargeCycle(targetDivId) {
        this.setScales(targetDivId);
        this.renderCycle(this.dischargeStates, this.dischargeProcesses, targetDivId, 'discharge');
    }

    renderBothCycles(targetDivId) {
        d3.select(`#${targetDivId}`).select(".cycle").selectAll("*").remove();
        this.setScales(targetDivId);
        this.showStorageFluidTempRange(targetDivId);
        this.renderCycle(this.chargeStates, this.chargeProcesses, targetDivId, 'charge');
        this.renderCycle(this.dischargeStates, this.dischargeProcesses, targetDivId, 'discharge');
    }

    setScales(targetDivId) {
        const container = d3.select(`#${targetDivId}`);
        const width = container.node().getBoundingClientRect().width;
        const height = container.node().getBoundingClientRect().height;

        const allStates = this.chargeStates.concat(this.dischargeStates);
        const sExtent = d3.extent(allStates, d => d.s);
        const TExtent = d3.extent(allStates, d => d.TinK);
        this.sScale = [sExtent[0] - 0.1*(sExtent[1]-sExtent[0]), sExtent[1] + 0.1*(sExtent[1]-sExtent[0])];
        this.TScale = [TExtent[0] - 0.1*(TExtent[1]-TExtent[0]), TExtent[1] + 0.1*(TExtent[1]-TExtent[0])];
        const sScale = this.cycleLayout.sScale || this.sScale;
        const TScale = this.cycleLayout.TScale || this.TScale;
        this.xScale.domain(sScale)
            .range([this.margin.left, width-this.margin.right]);
        this.yScale.domain(TScale)
            .range([height-this.margin.bottom, this.margin.top]);
    }

    showStorageFluidTempRange(targetDivId) {
        const container = d3.select(`#${targetDivId}`);
        const svg = container.select(".cycle");
        const sMin = this.xScale.domain()[0];
        const sMax = this.xScale.domain()[1];
        const Tmin = this.yScale.domain()[0];
        const Tmax = this.yScale.domain()[1];

        if (this.hotFluid) {
            const hotFluidData = this.hotStoreFluids.find( fluid => fluid.name == this.hotFluid );
            if (hotFluidData.TminK < Tmax) {
                svg.append("rect")
                    .attr("class", "hot-fluid-range")
                    .attr("x", this.xScale(sMin))
                    .attr("y", this.yScale(hotFluidData.TmaxK))
                    .attr("width", this.xScale(sMax) - this.xScale(sMin))
                    .attr("height", this.yScale(hotFluidData.TminK) - this.yScale(Math.min(hotFluidData.TmaxK, Tmax)))
                    .attr("fill", "red")
                    .attr("opacity", 0.5);
            }
        }

        if (this.coldFluid) {
            const coldFluidData = this.coldStoreFluids.find( fluid => fluid.name == this.coldFluid );
            if (coldFluidData.TmaxK > Tmin) {
                svg.append("rect")
                    .attr("class", "cold-fluid-range")
                    .attr("x", this.xScale(sMin))
                    .attr("y", this.yScale(coldFluidData.TmaxK))
                    .attr("width", this.xScale(sMax) - this.xScale(sMin))
                    .attr("height", this.yScale(Math.max(coldFluidData.TminK, Tmin)) - this.yScale(coldFluidData.TmaxK))
                    .attr("fill", "cornflowerblue")
                    .attr("opacity", 0.5);
            }
        }
    }


    renderCycle(states, processes, targetDivId, name) {
        const TfromPandS = (p,s) => {
            return this.Tref*Math.exp( (s + this.rGas*Math.log(p/this.pref))/this.cp );
        };

        const container = d3.select(`#${targetDivId}`);
        const width = container.node().getBoundingClientRect().width;
        const height = container.node().getBoundingClientRect().height;
        const svg = container.select(".cycle");

        const line = d3.line()
            .x(d => this.xScale(d.s))
            .y(d => this.yScale(d.TinK));
        const processClass = name + "-process";
        const stateClass = name + "-state";
        const tooltip = d3.select(".tooltip");
        svg.selectAll(`.${processClass}`)
            .data(processes)
            .enter()
            .each( d => {
                const from = states.find( state => state.station == d.from );
                const to = states.find( state => state.station == d.to );
                let path;
                if (d.path == "straight") {
                    path = svg.append("path")
                        .attr("class", processClass)
                        .attr("d", line([from, to]))
                        .attr("fill", "none")
                        .attr("stroke", d.colour || "steelblue")
                        .attr("stroke-width", 3);
                } else if (d.path == "curved") {
                    const p = from.p; // const p for now
                    const pts = d3.range(100).map( i => {
                        const s = from.s + i*(to.s-from.s)/100;
                        const TinK = TfromPandS(p,s);
                        return {s:s, TinK:TinK};
                    });
                    path = svg.append("path")
                        .attr("class", processClass)
                        .attr("d", line(pts))
                        .attr("fill", "none")
                        .attr("stroke", d.colour || "steelblue")
                        .attr("stroke-width", 3);
                }
                path.on("mouseover", (event) => {
                    tooltip.style("opacity", 0.9)
                        .html(d.label)
                        .style("background-color", d.labelCol)
                        .style("left", (event.pageX + 5) + "px")
                        .style("top", (event.pageY - 28) + "px");
                })
                .on("mouseout", () => {
                    tooltip.style("opacity", 0);
                });


            });
        svg.selectAll(`.${stateClass}`)
            .data(states)
            .enter()
            .append("circle")
            .attr("class", stateClass)
            .attr("cx", d => this.xScale(d.s))
            .attr("cy", d => this.yScale(d.TinK))
            .attr("r", 5)
            .attr("fill", "steelblue")
        const gX = svg.append("g")
            .attr("transform", `translate(0,${height-this.margin.bottom})`)
            .call(d3.axisBottom(this.xScale))
            .append("text")
            .attr("class", "axis-label")
            .attr("x", width/2)
            .attr("y", this.margin.bottom - 10)
            .attr("text-anchor", "middle")
            .attr("fill", "black")
            .style("font-size", "1.2em")
            .text("Entropy (J/kgK)");

        const gY = svg.append("g")
            .attr("transform", `translate(${this.margin.left},0)`)
            .call(d3.axisLeft(this.yScale))
            .append("text")
            .attr("class", "axis-label")
            .attr("x", -height/2)
            .attr("y", -this.margin.left + 15)
            .attr("text-anchor", "middle")
            .attr("transform", "rotate(-90)")
            .attr("fill", "black")
            .style("font-size", "1.2em")
            .text("Temperature (K)");

    }

    render() {

        const target = d3.select("#target");

        // Create the container div
        target.style("display", "flex")
            .style("flex-direction", "column")
            .style("width", "100%")
            .style("height", "800px")
            .style("background-color", "lightgrey")
            .style("gap", "10px")
            .style("padding", "10px")
            .style("border-radius", "10px");

         // Create a row for cycle and output
        const row = target.append("div")
            .style("display", "flex")
            .style("width", "100%")
            .style("gap", "10px");

        // Create cycle div
        const cycleDiv = row.append("div")
            .attr("id", "cycle-container")
            .style("width", "70%")
            .style("height", "500px")
            .style("border-radius", "10px")
            .style("background-color", "white");

         // Create output div
        row.append("div")
            .attr("id", "output")
            .style("flex-grow", "1")
            .style("height", "500px")
            .style("border-radius", "10px")
            .style("background-color", "#7fffd4")
            .style("overflow", "auto");

         // Create input div below
        target.append("div")
            .attr("id", "input")
            .style("width", "100%")
            .style("height", "300px")
            .style("border-radius", "10px")
            .style("background-color", "#87cefa");

        const width = cycleDiv.node().getBoundingClientRect().width;
        const height = cycleDiv.node().getBoundingClientRect().height;    
        cycleDiv.append("svg")
            .attr("class", "cycle")
            .attr("width", width)
            .attr("height", height);


        const tooltipDiv = cycleDiv.append("div")
            .attr("class", "tooltip")
            .style("position", "absolute")
            .style("border-radius", "5px")
            .style("pointer-events", "none")
            .style("font-weight", "bold")
            .style("padding", "5px")
            .style("opacity", 0);

        this.renderBothCycles("cycle-container");
        this.addOutputs();
        this.addInputs();
    }

    addInputs() {

        const inputDiv = d3.select("#input")
            .style("overflow", "auto");
      
        const container = inputDiv
            .append('div')
            .attr("class", "slider-grid");

        


        // add selects for hot fluid and cold fluid
        const hotFluidNames = this.hotStoreFluids.map( fluid => fluid.name );
        const hotFluidSelectContainer = container.append("div")
            .attr("class", "slider-container");
        hotFluidSelectContainer.append("span")
            .attr("class", "slider-title")
            .text("Hot Store Fluid");
        const hotFluidSelect = hotFluidSelectContainer.append("select")
            .attr("class", "hot-fluid-select")
            .on("change", (event) => {
                this.hotFluid = event.target.value;
                this.calculateStates();
                this.renderBothCycles("cycle-container");
                this.updateOutputs();
            });
        hotFluidSelect.selectAll("option")
            .data(hotFluidNames)
            .enter()
            .append("option")
            .attr("value", d => d)
            .text(d => d);
        hotFluidSelect.append("option")
            .attr("value", "")
            .text("None");
        hotFluidSelect.append("option")
            .attr("value", null)
            .text("Select hot fluid")
            .attr("disabled", true)
            .attr("selected", true);

        const coldFluidNames = this.coldStoreFluids.map( fluid => fluid.name );
        const coldFluidSelectContainer = container.append("div")
            .attr("class", "slider-container");
        coldFluidSelectContainer.append("span")
            .attr("class", "slider-title")
            .text("Cold Store Fluid");
        const coldFluidSelect = coldFluidSelectContainer.append("select")
            .attr("class", "cold-fluid-select")
            .on("change", (event) => {
                this.coldFluid = event.target.value;
                this.calculateStates();
                this.renderBothCycles("cycle-container");
                this.updateOutputs();
            });
        coldFluidSelect.selectAll("option")
            .data(coldFluidNames)
            .enter()
            .append("option")
            .attr("value", d => d)
            .text(d => d);
        coldFluidSelect.append("option")
            .attr("value", "")
            .text("None");
        coldFluidSelect.append("option")
            .attr("value", null)
            .text("Select cold fluid")
            .attr("disabled", true)
            .attr("selected", true);

      
        // Create a slider for each object in the data
        const sliderContainers = container
            .selectAll('.slider-container slider')
            .data(this.inputSliders)
            .enter()
            .append('div')
            .attr('class', 'slider-container');
      
        // Title
        sliderContainers.append('span')
            .attr('class', 'slider-title')
            .text(d => d.title);
      
        // Input Slider
        sliderContainers.append('input')
            .attr('type', 'range')
            .attr('class', 'slider')
            .attr('id', d => d.id)
            .attr('min', d => d.min)
            .attr('max', d => d.max)
            .attr('step', d => d.step)
            .attr('value', d => d.value)
            .on('input', (event, d) => {
                // get slider value
                let value = +d3.select(`#${d.id}`).node().value;

              //d.value = +this.value;
                d3.select(`#value-${d.id}`).text(value);
                    this[d.varName] = value;
                    this.calculateStates();
                    this.renderBothCycles("cycle-container");
                    this.updateOutputs();

            });
      
        // Value Display
        sliderContainers.append('span')
            .attr('class', 'slider-value')
            .attr('id', d => `value-${d.id}`)
            .text(d => d.value);

        // add checkbox for samePrat
        const samePratContainer = container.append("div")
            .attr("class", "slider-container");
        samePratContainer.append("span")
            .attr("class", "slider-title")
            .text("Same pressure ratio");
        const samePratCheckbox = samePratContainer.append("input")
            .attr("type", "checkbox")
            .attr("class", "same-prat-checkbox")
            .on("change", (event) => {
                this.samePrat = event.target.checked;
                this.calculateStates();
                this.renderBothCycles("cycle-container");
                this.updateOutputs();
            });
        samePratCheckbox.node().checked = this.samePrat;
    }

    addOutputs() {
        const outputDiv = d3.select("#output");
        const container = outputDiv.append("div")
            .attr("class", "output-grid");

        const rows = container
            .selectAll(".output-row")
            .data(this.outputs)
            .enter()
                .append("div")
                .attr("class", "output-row");

        // Append titles
        rows.append("span")
            .text(d => d.title + ":")
            .style("font-weight", "bold");

        // Append values
        rows.append("span")
            .attr("class", "value")
            .text(d => d3.format(d.fmt)(this[d.varName]));
    }

    updateOutputs() {
        d3.selectAll(".value")
            .text(d => d3.format(d.fmt)(this[d.varName]));
    }

}

const ptes = new PTES({samePrat:false});
ptes.cycleLayout.sScale=[-600, 1000];
ptes.cycleLayout.TScale=[50, 1050]; 
ptes.render();

</script>