---
title: Links by Domain Suffix
layout: dataset
license: jia-cc0
outputs:
- html
- dcxml
- dcresolve
---

# Links by Domain Suffix

```{download} link-matrix.json
```

<style>

.tab-pane {
  text-align: center;
}

svg {
  font: 10px sans-serif;
}

.axis path, .axis line {
  fill: none;
  stroke: #000;
  shape-rendering: crispEdges;
}

#circle circle {
  fill: none;
  pointer-events: all;
}

.group path {
  fill-opacity: .25;
}

path.chord {
  stroke: #000;
  stroke-width: .25px;
  opacity: .65;
}
path.chord:hover {
  opacity: 1.0;
}
#circle:hover path.fade {
   opacity: 0.1;
}
</style>

<script src="http://d3js.org/d3.v2.min.js?2.8.1"></script>

<div class="tabbable">
<ul class="nav nav-tabs" id="myTab">
  <li class="active"><a href="#net1996" data-toggle="tab">1996</a></li>
  <li><a href="#net1997" data-toggle="tab">1997</a></li>
  <li><a href="#net1998" data-toggle="tab">1998</a></li>
  <li><a href="#net1999" data-toggle="tab">1999</a></li>
  <li><a href="#net2000" data-toggle="tab">2000</a></li>
  <li><a href="#net2001" data-toggle="tab">2001</a></li>
  <li><a href="#net2002" data-toggle="tab">2002</a></li>
  <li><a href="#net2003" data-toggle="tab">2003</a></li>
  <li><a href="#net2004" data-toggle="tab">2004</a></li>
  <li><a href="#net2005" data-toggle="tab">2005</a></li>
  <li><a href="#net2006" data-toggle="tab">2006</a></li>
  <li><a href="#net2007" data-toggle="tab">2007</a></li>
  <li><a href="#net2008" data-toggle="tab">2008</a></li>
  <li><a href="#net2009" data-toggle="tab">2009</a></li>
  <li><a href="#net2010" data-toggle="tab">2010</a></li>
</ul>
 
<div class="tab-content">
  <div class="tab-pane active" id="net1996"></div>
  <div class="tab-pane" id="net1997"></div>
  <div class="tab-pane" id="net1998"></div>
  <div class="tab-pane" id="net1999"></div>
  <div class="tab-pane" id="net2000"></div>
  <div class="tab-pane" id="net2001"></div>
  <div class="tab-pane" id="net2002"></div>
  <div class="tab-pane" id="net2003"></div>
  <div class="tab-pane" id="net2004"></div>
  <div class="tab-pane" id="net2005"></div>
  <div class="tab-pane" id="net2006"></div>
  <div class="tab-pane" id="net2007"></div>
  <div class="tab-pane" id="net2008"></div>
  <div class="tab-pane" id="net2009"></div>
  <div class="tab-pane" id="net2010"></div>
</div>
</div>

<script>

function LinkagePlot( element, year ) {

var width = 640,
    height = 640,
    outerRadius = Math.min(width, height) / 2 - 10,
    innerRadius = outerRadius - 24;

var formatPercent = d3.format(".1%");

var arc = d3.svg.arc()
    .innerRadius(innerRadius)
    .outerRadius(outerRadius);

var layout = d3.layout.chord()
    .padding(.02)
    .sortSubgroups(d3.descending)
    .sortChords(d3.ascending);

var path = d3.svg.chord()
    .radius(innerRadius);

var svg = element.append("svg")
    .attr("width", width)
    .attr("height", height)
  .append("g")
    .attr("id", "circle"+year)
    .attr("transform", "translate(" + width / 2 + "," + height / 2 + ")");

svg.append("circle"+year)
    .attr("r", outerRadius);

d3.csv("link-index.csv", function(domains) {
  d3.json("link-matrix.json", function(matrix) {

    // Compute the chord layout.
    layout.matrix( matrix[year] );

    // Add a group per neighborhood.
    var group = svg.selectAll(".group")
        .data(layout.groups)
      .enter().append("g")
        .attr("class", "group")
        .on("mouseover", mouseover)
        .on("mouseout", mouseout);

    // Add a mouseover title.
    group.append("title").text(function(d, i) {
      return domains[i].name + ": " + formatPercent(d.value) + " of link sources";
    });

    // Add the group arc.
    var groupPath = group.append("path")
        .attr("id", function(d, i) { return "group" + i + "_" + year; })
        .attr("d", arc)
        .style("fill", function(d, i) { return domains[i].color; });

    // Add a text labeLinkagePlot.
    var groupText = group.append("text") 
        .attr("x", 6)
        .attr("dy", 15);

    groupText.append("textPath")
        .attr("xlink:href", function(d, i) { return "#group" + i + "_" + year; })
        .text(function(d, i) { if( domains[i].name.match(/\.uk$/) && d.value > 0.02 ) { return domains[i].name; } else { return ""; } });

    // Remove the labels that don't fit. :(
    //groupText.filter(function(d, i) { return groupPath[0][i].getTotalLength() / 2 - 16 < this.getComputedTextLength(); })
    //    .remove();

    // Add the chords.
    var chord = svg.selectAll(".chord")
        .data(layout.chords)
      .enter().append("path")
        .attr("class", "chord")
        .style("stroke", function(d, i) { return domains[d.source.index].color; })
        .style("fill", function(d) { return domains[d.source.index].color; })
        .attr("d", path);

    // Add an elaborate mouseover title for each chod.
    chord.append("title").text(function(d) {
      return domains[d.source.index].name
          + " → " + domains[d.target.index].name
          + ": " + formatPercent(d.source.value)
          + "\n" + domains[d.target.index].name
          + " → " + domains[d.source.index].name
          + ": " + formatPercent(d.target.value);
    });

    function mouseover(d, i) {
      chord.classed("fade", function(p) {
        return p.source.index != i
            && p.target.index != i;
      });      
    }
    function mouseout(d, i) {
      chord.classed("fade", function(p) {
        return false;
      });      
    }

  });
});

}

  new LinkagePlot(d3.select("#net1996"), "1996");
  new LinkagePlot(d3.select("#net1997"), "1997");
  new LinkagePlot(d3.select("#net1998"), "1998");
  new LinkagePlot(d3.select("#net1999"), "1999");
  new LinkagePlot(d3.select("#net2000"), "2000");
  new LinkagePlot(d3.select("#net2001"), "2001");
  new LinkagePlot(d3.select("#net2002"), "2002");
  new LinkagePlot(d3.select("#net2003"), "2003");
  new LinkagePlot(d3.select("#net2004"), "2004");
  new LinkagePlot(d3.select("#net2005"), "2005");
  new LinkagePlot(d3.select("#net2006"), "2006");
  new LinkagePlot(d3.select("#net2007"), "2007");
  new LinkagePlot(d3.select("#net2008"), "2008");
  new LinkagePlot(d3.select("#net2009"), "2009");
  new LinkagePlot(d3.select("#net2010"), "2010");

</script>


