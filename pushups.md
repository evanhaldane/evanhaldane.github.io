---
layout: default
title: Pushups
---

After a suggestion from a friend, I set the goal of 10,000 pushups in 2015. I have been tracking my progress via a Google Sheet. Thanks to [Tabletop.js](https://github.com/jsoma/tabletop), I can retrieve the contents of my Google Sheet (as JSON) completely in Javascript. The result is then plotted (again, completely in Javascript) using [Bokeh.js](http://bokeh.pydata.org/en/latest/docs/dev_guide/bokehjs.html), which I found out about at [2015 PyCon](https://us.pycon.org/2015/) (H/T: [Sarah Bird](http://www.sarahbird.org/))

  <head>
    <meta charset="utf-8">
    <script type="text/javascript" src="//ajax.googleapis.com/ajax/libs/jquery/2.0.3/jquery.min.js"></script>
    <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/tabletop.js/1.3.5/tabletop.js"></script>
    <script type="text/javascript" src="http://cdn.pydata.org/bokeh/release/bokeh-0.8.0.min.js"></script>
    <link rel="stylesheet" href="http://cdn.pydata.org/bokeh/release/bokeh-0.8.0.min.css">
  </head>
  <body>
  

  <div id="placeholder"></div>

  <script type="text/javascript">
    document.addEventListener('DOMContentLoaded', function() {
        var URL = "1otOt7tKmKnNK374D9COSFN53fkYmBxE2XKZWahxq7FQ"
        Tabletop.init( { key: URL, callback: myData, simpleSheet: true } )
    })
    function myData(data) {
        date = [];
        cumulative = [];
        for (var i = 0; i < data.length; i++){ 
        	if (Date.parse(data[i].date) > Date.now()) {
        		break;
        	}
        	date.push(Date.parse(data[i].date));
        	cumulative.push(parseFloat(data[i].cumulative));
        }

        dataForGraph = {date: date, cumulative: cumulative}

        xaxis = {
 			type: "Datetime",
  			location: "below",
  			grid: true
		}
		yaxis = {
			type: "auto",
			location: "left",
			grid: true
		}

		var plotWidth = jQuery("main").width();
		var plotHeight = Math.ceil(plotWidth*0.6);
		options = {
		  title: 'Total Pushups in 2015',
		  plot_width: plotWidth,
		  plot_height: plotHeight,
		  x_range: [Math.min.apply(null, date), Date.now()],
		  y_range: [0, Math.max.apply(null, cumulative)+50]
		}

		line = {
			type: 'Line',
			source: dataForGraph,
			line_color: '#43A2CA',
			line_width: 2,
			x: 'date',
			y: 'cumulative'
		}

		jQuery("#placeholder").bokeh("figure", {
			options: options,
			sources: { data: dataForGraph },
			glyphs: [line],
			guides: [xaxis, yaxis],
			tools: ["Pan", "WheelZoom" ,"Resize", "Reset", "PreviewSave"]
		})
    }
	  jQuery(window).resize(function() {
			    var plot = Bokeh.Collections("Plot").models[0];
			    plot.set("plot_width", jQuery("main").width());
			    plot.set("plot_height", Math.ceil(jQuery("main").width()*.6));
			});
  </script>

  </body>