<template>
  <section>
    <div id="grid-container">
      <div id="title-container">
        <h2
          id="title-main"
          class="title"
        >
          U.S. Groundwater Conditions
        </h2>
        <h3>
          {{ date_start }} to {{ date_end }}
        </h3>
      <!--   <caption id="caption-gwl">Daily groundwater levels</caption> -->
      </div>
      <div id="map-container">
        <GWLmap
          id="map_gwl"
          class="map"
        />
      </div>
      <div id="legend-container">
        <Legend />
      </div>
      <div id="button-container">
        <div id="spacer">
          <button 
            id="button-play"
            class="usa-button usa-button--outline"
          >
            {{ this.button_text }}
          </button>
          <button 
            id="button-speed"
            class="usa-button usa-button--outline"
          >
            {{ this.button_text_speed }}
          </button>
        </div>
      </div>

      <div id="line-container">
        <h4>
          Groundwater sites by water level
        </h4>
        <svg
          id="line-chart"
          preserveAspectRatio="xMinYMin meet"
          aria-labelledby="chartTitleID chartDescID"
          role="img"
        >
          <title id="chartTitleID">
            A line chart showing the proportion of groundwater sites by water level through time.
          </title><desc id="chartDescID">Five lines are drawn for the duration of the time period for sites categorized as very low, low, normal, high, and very high. Each line shows the proportion of the total groundwater sites in each category, which fluctuates through time due.</desc> 
        </svg>
        <caption>
          The percent of groundwater sites on the map by water-level category.
        </caption>
      </div>
      <div id="text-container">
        <p
          class="text-content tooltip"
        >
          This map animates groundwater levels at {{ this.n_sites }} well sites across the U.S. At each site, groundwater levels are shown relative to the historic record (
          <span class="tooltip-span">using percentiles</span>
          <span
            class="tooltiptext"
            style="font-size: 0.8rem;"
          >
            The percentile calculates the percent of days in the past that groundwater was below the current value. For a site is in the 10th percentile, water levels have been lower 10% of the time. 
          </span>
          ), indicating where groundwater is comparatively high or low to what has been observed in the past. The corresponding time series chart shows the percent of sites in each water-level category through time. 
        </p>
        <p
          class="text-content"
        >
          To learn more about groundwater monitoring efforts by the USGS and partners go to: <a
            href="https://www.usgs.gov/programs/groundwater-and-streamflow-information-program"
            target="_blank"
          >usgs.gov/gwsip</a>. 
        </p>
        <br>
        <h4>
          Data processing
        </h4>
        <p
          class="text-content"
        >
          This visualization is based on groundwater data from the <a
            href="https://waterdata.usgs.gov/nwis"
            target="_blank"
          >USGS National Water Information System (NWIS)</a>, accessed using the <a
            href="https://github.com/USGS-R/dataRetrieval"
            target="_blank"
          >dataRetrieval package for R</a>. To calculate percentiles we use a historic daily record spanning January 1, 1900 to December 31, 2020. To pull NWIS groundwater data for this historic daily record, we used the USGS <a
            href="https://help.waterdata.usgs.gov/codes-and-parameters/parameters"
            target="_blank"
          >parameter code</a>, 72019. If no daily values were available for 72019 but instantaneous records were, the daily value was calculated by averaging the instantaneous values per day based on the local time zone. For three states, the 72019 parameter code was not reported and a different parameter code was used to calculate daily groundwater percentiles (62610 was used for Florida and Kansas; 72150 was used for Hawaii). Only groundwater sites with a minimum of 3 years of data were used in the historic record, and sites were limited to those with continuous data. <a
          href="https://waterdata.usgs.gov/provisional-data-statement/"
          target="_blank"
          >Provisional data</a> were included in this analysis. 
        </p>
        <br>
        <hr>
        <img
          id="vizlab-wordmark"
          src="@/assets/usgsHeaderAndFooter/images/usgsvizlab-wordmark-black.png"
        >
        <p
          class="text-content"
        >
          <a
            href="https://www.usgs.gov/media/videos/us-river-conditions-july-september-2022"
            target="_blank"
          >See the latest U.S. River Conditions</a> and other <a
            href="https://labs.waterdata.usgs.gov/visualizations/vizlab-home/index.html?utm_source=viz&utm_medium=link&utm_campaign=gw_conditions#/"
            target="_blank"
          >data visualizations from the USGS VizLab
          </a>
        </p>
      </div>
    </div>
  </section>
</template>
<script>
import {select, selectAll } from 'd3-selection';
import { scaleLinear, scaleThreshold, scaleOrdinal } from 'd3-scale';
import * as d3Trans from 'd3-transition';
import { utcFormat } from 'd3-time-format';
import { axisBottom, axisLeft } from 'd3-axis';
import { csv } from 'd3-fetch';
import { line, path , format} from 'd3';
import GWLmap from "@/assets/gw-conditions-peaks-map.svg";
import { isMobile } from 'mobile-device-detect';

export default {
  name: "GWLsvg",
    components: {
      GWLmap,
      Legend: () => import( /* webpackPreload: true */ /*webpackChunkName: "Legend"*/ "./../components/Legend")
    },
    data() {
    return {
      publicPath: process.env.BASE_URL, // this is need for the data files in the public folder, this allows the application to find the files when on different deployment roots
      d3: null,
      mobileView: isMobile, // test for mobile

      // dimensions
      width: null,
      height: null,
      mar: 50,
      days: null, // used to index days in sequence

      peak_grp: null,
      day_length: 10, // frame duration in milliseconds
      current_time: 0, // tracking animation timing
      n_days: null,
      n_sites: {},
      play_button: null,
      speed_button: null,
      button_text: 'Pause',
      button_text_speed: 'Slower',
      date_start: null,
      date_end: null,
      date_list: null,
      formatTime: null,

      // scales
      quant_path_gylph: null,
      dates: null,
      xScale: null,
      yScale: null,
      line: null,

     // Blue-Brown categorical color scale
      verylow: "#BF6200",
      low: "#FEB100",
      normal: "#B3B3B3",
      high: "#2E9EC6",
      veryhigh: "#28648A",
      pal_BuBr: null,

    }
  },
  mounted(){
      this.d3 = Object.assign({d3Trans, scaleLinear, scaleThreshold, scaleOrdinal, select, selectAll, csv, utcFormat, line, path, axisBottom, axisLeft, format  });

      // resize
      var window_line = document.getElementById('line-container')
      this.width = window_line.clientWidth;
      this.height = window.innerHeight*.5;
      this.pal_BuBr = [this.veryhigh, this.high, this.normal, this.low, this.verylow];
      
      this.play_button = this.d3.select("#button-play")
      this.speed_button = this.d3.select("#button-speed")

      // read in data
      this.loadData();   

    },
    methods:{
      isMobile() {
              if(/Android|webOS|iPhone|iPad|iPod|BlackBerry|IEMobile|Opera Mini/i.test(navigator.userAgent)) {
                  return true
              } else {
                  return false
              }
          },
       loadData() {
        const self = this;
        // read in data 
        let promises = [
        self.d3.csv(self.publicPath + "quant_peaks.csv",  this.d3.autotype), // used to draw legend shapes - color palette needs to be pulled out
        self.d3.csv("https://labs.waterdata.usgs.gov/visualizations/data/gw-conditions-peaks-timeseries-wy22.csv",  this.d3.autotype),
        self.d3.csv("https://labs.waterdata.usgs.gov/visualizations/data/gw-conditions-site-coords.csv",  this.d3.autotype), 
        self.d3.csv("https://labs.waterdata.usgs.gov/visualizations/data/gw-conditions-daily-proportions-wy22.csv",  this.d3.autotype),
        self.d3.csv("https://labs.waterdata.usgs.gov/visualizations/data/gw-conditions-time-labels.csv",  this.d3.autotype),
        ];
        Promise.all(promises).then(self.callback); // once it's loaded
      },
      callback(data) {
        const self = this;
        // assign data

        // builds legend, has row for each category
        var quant_peaks = data[0]; 

        // gwl site level timeseries data to make peak animation
        // row for each day/frame, col for each site
        // first column is the day, subsequent are named "gwl_" + site_no
        // site values are svg scaled svg positions for the animation
        var date_peaks = data[1]; 

        // site coordinates for map
        // TODO: pre-draw on map and pick up with d3
        // requires duplicating map style in R so looks consistent on load
        // need to consider how to handle sites with no data on the first date 
        // variables: x, y, site_no, aqfr_type
        var site_coords = data[2]; 

        // proportion of sites by each category over time
        // variables: Date, day_seq (an integer from 1 to the last day), n_sites, and a column for each gwl category
        var site_count = data[3]; // number of sites x quant_category x day_seq

        // annotations for the timeline
        // R pipeline pulls out each month and the year for time labels
        // TODO: incorporate additional event annotations 
        var time_labels = data[4]; 

        // days in sequence
        var day_seq = date_peaks.columns
        day_seq.shift(); // drop first col with site_no. list of all the active

        // sites 
        var sites_list = site_coords.map(function(d)  { return d.site_no })
        this.n_sites = sites_list.length // to create nested array for indexing in animation

        // site placement on map
        var sites_x = site_coords.map(function(d) { return d.x })
        var sites_y = site_coords.map(function(d) { return d.y })

        // reorganize - site is the key with gwl for each day of the wy
        // can be indexed using site key (gwl_#) - and used to bind data to DOM elements
        var peaky = [];
        for (let i = 1; i < this.n_sites; i++) {
            var key = sites_list[i];
            var day_seq = date_peaks.map(function(d){  return d['day_seq']; });
            var gwl = date_peaks.map(function(d){  return d[key]; });
            var site_x = sites_x[i];
            var site_y = sites_y[i];
            peaky.push({key: key, day_seq: day_seq, gwl: gwl, site_x: site_x, site_y: site_y})
        };

        // same for timeseries data but indexed by percentile category
        var quant_cat = [...new Set(quant_peaks.map(function(d) { return d.quant}))];
        
        var n_quant = quant_cat.length
        var percData = [];
        for (let j = 0; j < n_quant; j++) {
          var key_quant = quant_cat[j];
          var day_seq = site_count.map(function(d){ return d['day_seq']});
          var perc = site_count.map(function(d){ return d[key_quant]});
          percData.push({key_quant: key_quant, day_seq: day_seq, perc: perc})
        };

        // managing dates and time sequencing
        this.days = site_count.map(function(d) { return  d['day_seq']})
        this.n_days = this.days.length
        this.dates = site_count.map(function(d) { return  d['Date']}) 
        this.formatDates(this.dates);
        
        // slightly different dimensions for drawing line chart on mobile and desktop
        if (this.mobileView){
          var line_height = 100;
          var font_size = '16px';
          var label_y = 10;
          var margin_x = 35;
          this.mar = 25;
        } else {
          var line_height = 150;
          var font_size = '20px';
          var label_y = 20;
          var margin_x = 50;
          this.mar = 50;
        }
        // set up scales
        var quant_path = [...new Set(quant_peaks.map(function(d) { return d.path_quant}))];
        this.setScales(quant_path, line_height, margin_x); // axes, color, and line drawing fun

        // draw the map
        var map_svg = this.d3.select("svg.map")
        this.drawFrame1(map_svg, peaky, start);

        // animated timeseries line chart
        var time_container = this.d3.select("#line-container");
        this.drawLineChart(time_container, percData, line_height, margin_x);
        this.addLabels(time_container, time_labels, line_height, margin_x, font_size, label_y);

        // control animation
        var start = 0;
        this.animateLine(start);
        this.animateGWL(start);

        this.setButton();
        this.setSpeed();

      },
      formatDates(dates){

        this.formatTime = this.d3.utcFormat("%b %e, %Y");
        this.date_start = this.formatTime(new Date(dates[0]));
        this.date_end = this.formatTime(new Date(dates[this.n_days-1]));

      },
      setButton(){
        const self = this;
         // set button triggers
         // switches between pause and play
        this.play_button
          .on("click", function(){

            if (self.button_text == "Pause"){
              // interrupt running transitions
              self.peak_grp.interrupt("daily_gwl")
              self.d3.select("rect.hilite").interrupt("daily_line")
              // change button text
              self.button_text = "Play";
            
            } else {
              self.animateGWL(self.current_time)
              self.animateLine(self.current_time)
              self.button_text = "Pause";
            }  
          })

      },
      setSpeed(){
        const self = this;
        this.speed_button
        .on("click", function(){
          if (self.button_text_speed == "Slower"){
            // speed toggle
             self.day_length = 300;
             self.button_text_speed = "Faster";
          } else {
            self.day_length = 10
            self.button_text_speed = "Slower";
          }
        })

      },
      addLabels(time_container, time_labels, line_height, margin_x, font_size, label_y){
        const self = this;
       // timeline labels

        var label_month = time_container.select('#line-chart')
          .append("g")
          .attr("transform", "translate(" + 0+ "," + (line_height+this.mar/2) + ")")

        // month lines on timeline
        label_month.selectAll(".month_tick")
          .data(time_labels).enter()
          .append("line")
          .attr("class", function(d) { return "label_inner inner_" + d.month_label + "_" + d.year } ) 
          .attr("x1", function(d) { return self.xScale(d.day_seq) })
          .attr("x2", function(d) { return self.xScale(d.day_seq) })
          .attr("y1", 0)
          .attr("y2", 5)
          .attr("stroke", "black")

        // month labels
        label_month.selectAll(".label_name")
          .data(time_labels)
          .enter()
          .append("text")
          .attr("class", function(d) { return "label_name name_" + d.month_label + "_" + d.year } ) 
          .attr("x", function(d) { return self.xScale(d.day_seq) }) // centering on pt
          .attr("y", (label_y+10))
          .text(function(d) { return d.month_label })
          .attr("text-anchor", "middle")
          .style("alignment-baseline", "top")
          .attr("font-size", font_size)

          this.d3.selectAll(".tick text")
            .attr("font-size", font_size)


        // filter to just year annotations for first month they appear
        var year_labels = time_labels.filter(function(el) {
          return el.year_label >= 2000;
        });

        // add year labels to timeline
        label_month.selectAll(".label_year")
          .data(year_labels)
          .enter()
          .append("text")
          .attr("class", function(d) { return "label_year label_" + d.year } ) 
          .attr("x", function(d) { return self.xScale(d.day_seq) }) // centering on pt
          .attr("y", (label_y*2+15))
          .text(function(d) { return d.year })
          .attr("text-anchor", "middle")
          .style("alignment-baseline", "top")
          .attr("font-size", font_size)

      },
      drawLineChart(time_container, prop_data, line_height, margin_x) {
        const self = this;

        // set up svg for timeline
        var svg = time_container.select("svg")
          .attr("viewBox", "0 0 " + this.width + " " + (line_height+this.mar*2))
          .append("g")
          .attr("id", "time-chart")
          .attr("transform", "translate(" + 0 + "," + this.mar/2 + ")")

        // define axes
        var xLine = this.d3.axisBottom()
          .scale(this.xScale)
          .ticks(0).tickSize(0); // add using imported label data

        var yLine = this.d3.axisLeft().tickFormat(this.d3.format('~%'))
          .scale(this.yScale)
          .ticks(2).tickSizeInner(4).tickSizeOuter(0);

        // draw axes
        svg.append("g")
          .call(xLine)
          .attr("transform", "translate(0," + (line_height) + ")")
          .classed("axis", true)

        svg.append("g")
          .call(yLine)
          .classed("axis", true)
          .attr("transform", "translate(" + margin_x + ", 0)")
          .attr("z-index", "10")

      // add a tiny box over top of axis to add better label
        svg.append("rect")
          .attr("width", "10")
          .attr("height", "10")
          .attr("x", margin_x-5)
          .attr("y", "0")
          .attr("fill", "rgb(223, 223, 223)")
          .attr("z-index", "1")

        // add line chart showing proportion of gages in each category
        var line_chart = this.d3.select("#time-chart")

        // add percent lines to chart
        line_chart.append("g")
          .attr("fill", "none")
          .attr("stroke-linejoin", "round")
          .attr("stroke-linecap", "round")
          .selectAll("path")
          .data(prop_data)
          .join("path")
          .attr("d", d => self.line(d.perc))
          .attr("stroke", function(d) { return self.gwl_color(d.key_quant) })
          .attr("stroke-width", "3px")
          .attr("opacity", 0.8)

        // animate line to time
        line_chart.append("rect")
          .data(this.days)
          .classed("hilite", true)
          .attr("width", "5")
          .attr("height", line_height)
          .attr("opacity", 0.9)
          .attr("fill", "#9b6adb8e")
          .attr("x", self.xScale(this.days[0]))

        // custom axis annotation
        this.d3.select("g.tick:last-child")
        .select("line")
        .attr("x2", "0")

        svg
        .append("text")
        .attr("class", "axis-text-top")
        .text("of " + this.n_sites + " total sites")
        .attr("text-anchor", "start")
        .attr("font-size", "1rem")
        .attr("x", margin_x)
        .attr("y","0.32rem")
        .attr("fill", "black")
  
      },
      setScales(quant_path, line_height, margin_x){

        // set color scale for path fill
        this.quant_color = this.d3.scaleThreshold()
          .domain([-40, -25, 25, 40])
          .range(this.pal_BuBr) 
        
        this.quant_path_gylph = this.d3.scaleThreshold()
          .domain([-40, -25, 25, 40])
          .range(quant_path.reverse()) 

        // x axis of line chart
        this.xScale = this.d3.scaleLinear()
          .domain([1, this.n_days])
          .range([margin_x, this.width-(margin_x)])

        // y axis of line chart
        this.yScale = this.d3.scaleLinear()
          .domain([0, 0.6]) // this should come from the data - round up from highest proportion value
          .range([line_height, 0])

        // line drawing 
        this.line = this.d3.line()
          .defined(d => !isNaN(d))
          .x((d, i) => this.xScale(this.days[i]))
          .y(d => this.yScale(d))

        // categorical color scale
        this.gwl_color = this.d3.scaleOrdinal()
          .domain(["Veryhigh", "High", "Normal", "Low","Verylow"])
          .range(this.pal_BuBr)

      },
      animateLine(start){
        // animates grey line on timeseries chart to represent current timepoint
        const self = this;
        
        // store time to restart at same point
        self.current_time = start+1

        if (start < this.n_days-1){
          this.d3.selectAll(".hilite")
          .transition('daily_line')
            .duration(this.day_length) 
            .attr("x", self.xScale(this.days[start]))
          .end()
          .then(() => this.animateLine(self.current_time))

          this.d3.selectAll(".ticker-date")
          .transition()
            .duration(this.day_length) 
            .text(this.dates[start])
          .end()
          .then(() => this.animateLine(start))

        } else {
          this.d3.selectAll(".hilite")
            .transition('daily_line')
            .duration(this.day_length) 
            .attr("x", self.xScale(this.days[0]))
          
          // reset play/pause button
          self.current_time = 0;
          self.button_text = "Play";

          // animate date ticker
        /*   this.d3.selectAll(".ticker-date")
          .transition()
            .duration(this.day_length) 
            .text(self.formatTime(new Date(this.dates[start])))//this.dates[0]) */

        }
      },
      drawFrame1(map_svg, data, start){         
        // draw the first frame of the animation
        const self = this;

        // draw sites with D3
          this.peak_grp = map_svg.selectAll("path.gwl_glyph")
            .data(data, function(d) { return d ? d.key : this.class; }) // binds data based on class/key
            .join("path") // match with selection
            .attr("transform", d => `translate(` + d.site_x + ' ' + d.site_y + `) scale(0.35 0.35)`)

        // draws a path for each site, using the first date
        this.peak_grp 
          .attr("class", function(d) { return "gwl " + d.key })
          .attr("fill", function(d) { return self.quant_color(d.gwl[start]) }) 
          .attr("opacity", ".7")
          .attr("d", function(d) { return self.quant_path_gylph(d.gwl[start]) }) 

          // add date ticker
/*     map_svg
        .append("text")
        .attr("class", "ticker-date") 
        //.attr("x", 500) // centering on pt
       // .attr("y", 100)
       .attr("transform",`translate(600, 500)`)
        .text(self.formatTime(new Date(this.dates[start])))//this.dates[start])
        .attr("text-anchor", "end") 
        console.log(this.dates[start]) */
        
      },
      animateGWL(start){
        const self = this;
        // animate path d and fill by wy day  
    
        if (start < this.n_days-1){
        
          this.peak_grp
            .transition('daily_gwl')
            .duration(this.day_length)
            .each(function(d,i) {
              let current_path = self.d3.select(this)
              var today = self.quant_path_gylph(d.gwl[start])
              var yesterday = self.quant_path_gylph(d.gwl[start-1])
              if(today != yesterday){
                self.animatePathD(start, current_path)
              }
            })
            .end() // end is important because it waits for EVERY element to finish the transition before callback, keeps things in sync
            .then(() => this.animateGWL(self.current_time)) // loop animation increasing by 1 day
        } else {
      // if it's the last day of the water year, stop animation on the first frame

          this.peak_grp
            .transition('daily_gwl')
            .duration(this.day_length)  // duration of each day
            .attr("d", function(d) { return self.quant_path_gylph(d.gwl[0]) })//{ return "M-10 0 C -10 0 0 " + d.gwl[this.n_days-1] + " 10 0 Z" })
            .attr("fill", function(d) { return self.quant_color(d.gwl[0]) })
        }
      },
      animatePathD(start, current_path){
        const self = this;
        current_path
          .transition()
          .attr("d", function(d) { 
            return self.quant_path_gylph(d.gwl[start])
            })
          .attr("fill", function(d) { return self.quant_color(d.gwl[start]) })
      }
    }
}
</script>
<style scoped lang="scss">
$dark: rgba(54, 54, 54, 0.8);
$light: #B3B3B3;
// each piece is a separate div that can be positioned or overlapped with grid
// mobile first
section {
  width: 92vw;
  margin: auto;
}
#grid-container {
  display: grid;
  max-width: 1600px;
  margin: auto;
  height: auto;
  vertical-align: middle;
  overflow: hidden;
  grid-template-columns: 1fr;
  grid-template-areas:
  "title"
  "map"
  "legend"
  "button"
  "line"
  "text";
   @media screen and (min-width: 551px) {
      grid-template-columns: 1fr 2fr;
      grid-template-areas:
      "title title"
      "map map"
      "button legend"
      "line line"
      "text text";
      }

}
#map-container{
  grid-area: map;
  padding: 0rem;
  padding-bottom: 0px;
  margin-top: 0.5rem;
  display: flex;
  justify-content: center;
  align-items: center;
  svg.map {
    max-height: 68vh;
  }
}
#line-container {
  grid-area: line;
  width: 100%;
  max-width: 700px;
  margin: auto;
  margin-bottom: 10px;

  svg{
    overflow: hidden;
  }
}
#legend-container {
   grid-area: legend;
  width: 100%;
  margin: auto;
  margin-bottom: 1rem;
 justify-content: center;
  max-width: 550px;
  align-self: right;
  justify-self: start;
  svg{
    max-width: 550px;
    margin: auto;
    align-self: start;
    justify-self: start;
    overflow: visible;
  }
   @media screen and (min-width: 551px) {
      justify-content: start;

      svg {
        align-self: start;
        justify-self: start;
      }
      }

}
#title-container {
  grid-area: title;
  width: 100%;
  max-width: 700px;
  height: auto;
  margin: auto;
  align-items: start;
  h1, h2{
    margin-top: 1rem;
  }
  h3 {
    margin-top: 0.5rem;

  }
}
#text-container {
  grid-area: text;
  max-width: 700px;
  margin: 1rem auto;
  margin-top: 0;
}
#button-container {
  grid-area: button;
  width: 100%;
  max-width: 700px;
  height: auto;
  min-height: 40px;
  margin: auto;
  margin-bottom: 1rem;
  justify-content: space-evenly;
  position: relative;
}
.text-content {
  margin: 0.5rem auto;
  max-width: 700px;
}
.text-content:last-child {
  margin-bottom: 1rem;
}
#map-text {
  display: inline-block;
}
// drop shadow on map outline
#bkgrd-map-grp {
  filter: drop-shadow(0.2rem 0.2rem 0.5rem rgba(38, 49, 43, 0.45));
  stroke-width: 0.2;
  color: white;
  fill: white;
}
// buttons
.usa-button--outline {
  justify-content: space-evenly;
  width: 100px;
  height: auto;
  border: 2px solid $dark;
  background: white;
  color: $dark;
  box-shadow: 2px 3px $dark;
  border-radius: 0.35rem;
  cursor: pointer;
  font-weight: 700;
  font-size: 1rem;
  padding: 0.5rem 0.5rem;
  text-align: center;
  text-decoration: none;
  overflow: visible;
  @media screen and (max-width: 550px) {
        font-size: 16px;
      }

}
button {
    appearance: auto;
    text-rendering: auto;
    letter-spacing: normal;
    word-spacing: normal;
    line-height: normal;
    align-items: center;
    box-sizing: border-box;
    padding: 1rem 4px;
    margin: 0rem 1rem;;
}
[type=button], [type=reset], [type=submit], button {
    -webkit-appearance: button;
}
button:hover {
    background: #9b6adb8e;
    color: white;
    box-shadow: 2px 3px $dark;
}
button:active {
  background-color: #9b6adb8e;
  color: white;
  box-shadow: 2px 3px white;
  transform: translateY(3px) translateX(2px);
}
// legend styling
line.legend-line {
  stroke-dasharray: 3;
  stroke-width: 2;
  stroke: grey;
}
line.legend-tick {
  stroke-width: 2;
  stroke: grey;
}
text.legend-text {
  text-anchor: middle;
  font-size: 0.8rem;
}
text.legend-label {
  text-anchor: middle;
  font-size: 1rem;
  font-weight: 700;
}
.axis {
  color: black;
  stroke-width: 2px;
}
#spacer {
  display: flex;
  justify-content: center;
  @media screen and (min-width: 551px) {
      justify-content: end;
      }
}
#vizlab-wordmark {
  max-width: 200px;
  display: block;
  margin: auto;
  justify-self: center;
}
.line-islands {
  stroke-dasharray: 1,3;
  fill: transparent;
  stroke: $dark;
}
#map-text text{
  font-size: 0.6rem;
  color: $dark;
  opacity: 0.6;
  font-style: italic;
}
#label-islands {
  opacity: 0.65;
}
.tooltip-span {
  position: relative;
  display: inline-block;
  border-bottom: 1px dotted $dark;
  z-index: 10;
}

.tooltip {
  display: inline-block;
}

.tooltiptext {
  visibility: hidden;

  /* Position the tooltip */
  position: absolute;
  z-index: 1;
}
.tooltiptext {
  width: 250px;
  background-color: rgba(54, 54, 54, 0.95);
  color: #fff;
  text-align: center;
  border-radius: 6px;
  padding: 7px 7px;
  P {
  font-size: 0.8 rem;
  }
  overflow: visible;
  //top: 0;
  //left: 50%;
  margin-left: -170px;
  margin-top: 20px;
}
.tooltip:hover .tooltiptext {
  visibility: visible;
}
.ticks {
  font: 10px sans-serif;
}

</style>