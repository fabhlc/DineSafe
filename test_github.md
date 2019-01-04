
### [Introduction](#intro)
### [Part I. Methodology](#methodology)

## Introduction <a class ="anchor" id = "intro"></a>


```python
# Import libraries
from collections import Counter
import numpy as np
import pandas as pd

from mapboxgl.utils import *
from mapboxgl.viz import *

# Set display options
pd.set_option("display.max_rows", 500)
pd.set_option('display.max_columns', 100)
pd.set_option("display.max_colwidth", 200)
```


```python
with open('Data/turnover_id.csv', 'r') as f:
    turnover = pd.read_csv(f, index_col = None)
with open('Data/turnover_address (QGIS_withneighbourhoods).csv', 'r') as f:
    turnover_address = pd.read_csv(f, index_col = None)
    
# Create a more compact dataframe without the change/exist columns
turnover_address_1 = pd.concat((turnover_address.sort_values('change_index', ascending = False).iloc[:,:1],
                                turnover_address.sort_values('change_index', ascending = False).iloc[:,14:]), axis = 1)


# Create a reference table for establishments and addresses.

turnover['first_exist'] = ['201'+str(list(i).index(1)) for i in turnover.values]
ests = turnover.groupby('address')['name'].apply(list)
yrs = turnover.groupby('address')['first_exist'].apply(list)
zipped = list(zip(ests, yrs))

est_by_add = pd.DataFrame(ests).join(pd.DataFrame(yrs))
est_by_add.reset_index(inplace=True)

# Clean fields to make it more readable. "Esta." must be clean as it will feed into the map labels later.
est_by_add['Esta.'] = [list(zip(i,j)) for (i,j) in zipped] 
est_by_add['name'] = [", ".join(i) for i in est_by_add['name']]
est_by_add['first_exist'] = [", ".join(i) for i in est_by_add['first_exist']]
est_by_add['Esta.'] = [", ".join(["{} ({})".format(j[0].title(), j[1]) for j in i]) for i in est_by_add['Esta.']]
```


# Part I. Methodology <a class ="anchor" id = "methodology"></a>


![FINAL%20Change_Index%20Diff%20from%20City%20Avg%20by%20Neighbourhood.png](attachment:FINAL%20Change_Index%20Diff%20from%20City%20Avg%20by%20Neighbourhood.png)


```python
# Create dataframe to feed into map
# df = turnover_address.join(est_by_add[['address','Esta.']].set_index('address'), on='address', how = 'left')
data_url = 'https://raw.githubusercontent.com/fabhlc/DineSafe/master/turnover_points_for_map.csv'
df = pd.read_csv(data_url, encoding = 'ANSI')

df_to_geojson(df,
              filename='markers.geojson',
              properties = ['address','total_turnover','annual_turnover','max_units','change_index','Esta.'],
              lat='LATITUDE', lon='LONGITUDE', precision=3)

# Read private token from local file 
with open('mapbox_token.txt','r') as f:
    token = f.read()

# The leftmost and rightmost bin edges
first_edge, last_edge = df['change_index'].min(), df['change_index'].max()
n_equal_bins = 5
color_breaks = np.linspace(start=first_edge, stop=last_edge, num=n_equal_bins + 1, endpoint=True)

# Generate data breaks and color stops from colorBrewer
color_stops = create_color_stops(np.round(color_breaks,3), colors='YlOrRd')

viz = CircleViz('markers.geojson', access_token=token, 
                radius = 2, center = (-79.342406,43.720250), # DVP/Lawrence
                zoom = 10,
                color_property = 'change_index',
                color_stops=color_stops,
                label_color = '#69F0AE',
                color_default='#69F0AE',
                stroke_color='#a7a7a7',
               stroke_width = 0.1)
viz.show()
```


<iframe id="map", srcdoc="<!DOCTYPE html>
<html>
<head>
<title>mapboxgl-jupyter viz</title>
<meta charset='UTF-8' />
<meta name='viewport'
      content='initial-scale=1,maximum-scale=1,user-scalable=no' />
<script type='text/javascript'
        src='https://api.tiles.mapbox.com/mapbox-gl-js/v0.51.0/mapbox-gl.js'></script>
<link type='text/css'
      href='https://api.tiles.mapbox.com/mapbox-gl-js/v0.51.0/mapbox-gl.css' 
      rel='stylesheet' />

<style type='text/css'>
    body { margin:0; padding:0; }
    .map { position: absolute; top:0; bottom:0; width:100%; }
    .legend {
        background-color: white;
        color: #6e6e6e;
        border-radius: 3px;
        bottom: 10px;
        box-shadow: 0 1px 2px rgba(0, 0, 0, 0.10);
        font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
        padding: 0;
        position: absolute;
        right: 10px;
        z-index: 1;
        min-width: 100px;
    }
   .legend.horizontal {bottom: 10px; text-align: left;}

    /* legend header */
    .legend .legend-header { border-radius: 3px 3px 0 0; background: white; }
    .legend .legend-title {
        padding: 6px 12px 6px 12px;
        text-shadow: 0 0 2px white;
        text-transform: capitalize;
        text-align: center;
        font-weight: bold !important;
        font-size: 14px;
        font: 12px/20px 'Helvetica Neue', Arial, Helvetica, sans-serif;
        max-width: 160px;
    }
    .legend-title {padding: 6px 12px 6px 12px; text-shadow: 0 0 2px #FFF; text-transform: capitalize; text-align: center; max-width: 160px; font-size: 0.9em; font-weight: bold;}
    .legend.horizontal .legend-title {text-align: left;}

    /* legend items */
    .legend-content {margin: 6px 12px 6px 12px; overflow: hidden; padding: 0; float: left; list-style: none; font-size: 0.8em;}
    .legend.vertical .legend-item {white-space: nowrap;}
    .legend-value {display: inline-block; line-height: 18px; vertical-align: top;}
    .legend.horizontal ul.legend-content li.legend-item .legend-value,
    .legend.horizontal ul.legend-content li.legend-item {display: inline-block; float: left; width: 30px; margin-bottom: 0; text-align: center; height: 30px;}

    /* legend key styles */
    .legend-key {display: inline-block; height: 10px;}
    .legend-key.default, .legend-key.square {border-radius: 0;}
    .legend-key.circle {border-radius: 50%;}
    .legend-key.rounded-square {border-radius: 20%;}
    .legend.vertical .legend-key {width: 10px; margin-right: 5px; margin-left: 1px;}
    .legend.horizontal .legend-key {width: 30px; margin-right: 0; margin-top: 1px; float: left;}
    .legend.horizontal .legend-key.square, .legend.horizontal .legend-key.rounded-square, .legend.horizontal .legend-key.circle {margin-left: 10px; width: 10px;}
    .legend.horizontal .legend-key.line {margin-left: 5px;}
    .legend.horizontal .legend-key.line, .legend.vertical .legend-key.line {border-radius: 10%; width: 20px; height: 3px; margin-bottom: 2px;}

    /* gradient bar alignment */
    .gradient-bar {margin: 6px 12px 6px 12px;}
    .legend.horizontal .gradient-bar {width: 88%; height: 10px;}
    .legend.vertical .gradient-bar {width: 10px; min-height: 50px; position: absolute; bottom: 4px;}

    /* contiguous vertical bars (discrete) */
    .legend.vertical.contig .legend-key {height: 15px; width: 10px;}
    .legend.vertical.contig li.legend-item {height: 15px;}
    .legend.vertical.contig {padding-bottom: 6px;}

</style>

<style>
    .gradient-bar.bordered, .legend-key.bordered { border: solid #a7a7a7 0.1px; }
</style>

</head>
<body>

<div id='map' class='map'></div>

<script type='text/javascript'>

var legendHeader;

function calcColorLegend(myColorStops, title) {

    // create legend
    var legend = document.createElement('div');
    if ('circle' === 'contiguous-bar') {
        legend.className = 'legend vertical contig';
    }
    else {
        legend.className = 'legend vertical';
    }

    legend.id = 'legend';
    document.body.appendChild(legend);

    // add legend header and content elements
    var mytitle = document.createElement('div'),
        legendContent = document.createElement('ul');
    legendHeader = document.createElement('div');
    mytitle.textContent = title;
    mytitle.className = 'legend-title'
    legendHeader.className = 'legend-header'
    legendContent.className = 'legend-content'
    legendHeader.appendChild(mytitle);
    legend.appendChild(legendHeader);
    legend.appendChild(legendContent);

    if (false === true) {
        var gradientText = 'linear-gradient(to right, ';
        var gradient = document.createElement('div');
        gradient.className = 'gradient-bar';
        legend.appendChild(gradient);
    }

    // calculate a legend entries on a Mapbox GL Style Spec property function stops array
    for (p = 0; p < myColorStops.length; p++) {
        if (!!document.getElementById('legend-points-value-' + p)) {
            //update the legend if it already exists
            document.getElementById('legend-points-value-' + p).textContent = myColorStops[p][0];
            document.getElementById('legend-points-id-' + p).style.backgroundColor = myColorStops[p][1];
        }
        else {
            // create the legend if it doesn't yet exist
            var item = document.createElement('li');
            item.className = 'legend-item';

            var key = document.createElement('span');
            key.className = 'legend-key circle';
            key.id = 'legend-points-id-' + p;
            key.style.backgroundColor = myColorStops[p][1];   

            var value = document.createElement('span');
            value.className = 'legend-value';
            value.id = 'legend-points-value-' + p;

            item.appendChild(key);
            item.appendChild(value);
            legendContent.appendChild(item);
            
            data = document.getElementById('legend-points-value-' + p)

            // round number values in legend if precision defined
            if ((typeof(myColorStops[p][0]) == 'number') && (typeof(null) == 'number')) {
                data.textContent = myColorStops[p][0].toFixed(null);
            }
            else {
                data.textContent = myColorStops[p][0];
            }

            // add color stop to gradient list
            if (false === true) {
                if (p < myColorStops.length - 1) {
                    gradientText = gradientText + myColorStops[p][1] + ', ';
                }
                else {
                    gradientText = gradientText + myColorStops[p][1] + ')';
                }
                if ('vertical' === 'vertical') {
                    gradientText = gradientText.replace('to right', 'to bottom');
                }
            }
        }
    }

    if (false === true) {
        // convert to gradient scale appearance
        gradient.style.background = gradientText;

        // hide legend keys generated above
        var keys = document.getElementsByClassName('legend-key');
        for (var i=0; i < keys.length; i++) {
            keys[i].style.visibility = 'hidden';
        }

        if ('vertical' === 'vertical') {
            gradient.style.height = (legendContent.offsetHeight - 6) + 'px';
        }
    }

    // add class for styling bordered legend keys
    if (true) {
        var keys = document.getElementsByClassName('legend-key');
        for (var i=0; i < keys.length; i++) {
            if (keys[i]) {
                keys[i].classList.add('bordered');
            }
        }
        var gradientBars = document.getElementsByClassName('gradient-bar');
        for (var i=0; i < keys.length; i++) {
            if (gradientBars[i]) {
                gradientBars[i].classList.add('bordered');
            }
        }
    }

    // update right-margin for compact Mapbox attribution based on calculated legend width
    var attribMargin = legend.offsetWidth + 15;
    document.getElementsByClassName('mapboxgl-ctrl-attrib')[0].style.marginRight =  attribMargin.toString() + 'px';

}


function generateInterpolateExpression(propertyValue, stops) {
    var expression;
    if (propertyValue == 'zoom') {
        expression = ['interpolate', ['exponential', 1.2], ['zoom']]
    }
    else if (propertyValue == 'heatmap-density') {
        expression = ['interpolate', ['linear'], ['heatmap-density']]
    }
    else {
        expression = ['interpolate', ['linear'], ['get', propertyValue]]
    }

    for (var i=0; i<stops.length; i++) {
        expression.push(stops[i][0], stops[i][1])
    }
    return expression
}


function generateMatchExpression(propertyValue, stops, defaultValue) {
    var expression;
    expression = ['match', ['get', propertyValue]]
    for (var i=0; i<stops.length; i++) {
        expression.push(stops[i][0], stops[i][1])
    }
    expression.push(defaultValue)
    
    return expression
}


function generatePropertyExpression(expressionType, propertyValue, stops, defaultValue) {
    var expression;
    if (expressionType == 'match') {
        expression = generateMatchExpression(propertyValue, stops, defaultValue)
    }
    else {
        expression = generateInterpolateExpression(propertyValue, stops)
    }

    return expression
}

</script>

<!-- main map creation code, extended by mapboxgl/templates/circle.html -->
<script type='text/javascript'>

    mapboxgl.accessToken = 'pk.eyJ1IjoiZmFiaGxjIiwiYSI6ImNqaGt4c2VqMTMydDMzNnAzbG0xNTU4ZTgifQ.noB8ToOGNBZbQZbJH-2F0A';

    var map = new mapboxgl.Map({
        container: 'map',
        attributionControl: false,
        style: 'mapbox://styles/mapbox/light-v9?optimize=true',
        center: [-79.342406, 43.72025],
        zoom: 10,
        pitch: 0,
        bearing: 0,
        scrollZoom: true,
        touchZoom: true,
        doubleClickZoom: true,
        boxZoom: true,
        transformRequest: (url, resourceType) => {
                    if ( url.slice(0,22) == 'https://api.mapbox.com' || 
                        url.slice(0,26) == 'https://a.tiles.mapbox.com' || 
                        url.slice(0,26) == 'https://b.tiles.mapbox.com' ||
                        url.slice(0,26) == 'https://c.tiles.mapbox.com' ||
                        url.slice(0,26) == 'https://d.tiles.mapbox.com') {
                        //Add Mapboxgl-Jupyter Plugin identifier for Mapbox API traffic
                        return {
                           url: [url.slice(0, url.indexOf('?')+1), 'pluginName=PythonMapboxgl&', url.slice(url.indexOf('?')+1)].join('')
                         }
                     }
                     else {
                         //Do not transform URL for non Mapbox GET requests
                         return {url: url}
                     }
                },
    });

    
    
        map.addControl(new mapboxgl.AttributionControl({ compact: true }));

    

    

        map.addControl(new mapboxgl.NavigationControl());

    

    

    
        
            calcColorLegend([[0.0, 'rgb(255,255,178)'], [0.267, 'rgb(254,217,118)'], [0.533, 'rgb(254,178,76)'], [0.8, 'rgb(253,141,60)'], [1.067, 'rgb(240,59,32)'], [1.333, 'rgb(189,0,38)']] , 'change_index');
        
    
        


    

    map.on('style.load', function() {
        
        

        map.addSource('data', {
            'type': 'geojson',
            'data': 'markers.geojson',
            'buffer': 1,
            'maxzoom': 14
        });

        map.addLayer({
            'id': 'label',
            'source': 'data',
            'type': 'symbol',
            'maxzoom': 24,
            'minzoom': 0,
            'layout': {
                
                'text-size' : generateInterpolateExpression('zoom', [[0, 8],[22, 3* 8]] ),
                'text-offset': [0,-1]
            },
            'paint': {
                'text-halo-color': 'white',
                'text-halo-width': generatePropertyExpression('interpolate', 'zoom', [[0,1], [18,5* 1]]),
                'text-color': '#69F0AE'
            }
        }, '' )
        
        map.addLayer({
            'id': 'circle',
            'source': 'data',
            'type': 'circle',
            'maxzoom': 24,
            'minzoom': 0,
            'paint': {
                
                    'circle-color': generatePropertyExpression('interpolate', 'change_index', [[0.0, 'rgb(255,255,178)'], [0.267, 'rgb(254,217,118)'], [0.533, 'rgb(254,178,76)'], [0.8, 'rgb(253,141,60)'], [1.067, 'rgb(240,59,32)'], [1.333, 'rgb(189,0,38)']], '#69F0AE' ),
                    
                'circle-radius' : generatePropertyExpression('interpolate', 'zoom', [[0,2], [22,10 * 2]]),
                'circle-stroke-color': '#a7a7a7',
                'circle-stroke-width': generatePropertyExpression('interpolate', 'zoom', [[0,0.1], [18,5* 0.1]]),
                'circle-opacity' : 1,
                'circle-stroke-opacity' : 1
            }
        }, 'label');
        
        

        // Popups
        
            var popupAction = 'mousemove',
                popupSettings =  {
                    closeButton: false,
                    closeOnClick: false
                };
        

        // Create a popup
        var popup = new mapboxgl.Popup(popupSettings);

        
        
        // Show the popup on mouseover
        map.on(popupAction, 'circle', function(e) {
            
            
                map.getCanvas().style.cursor = 'pointer';
            

            let f = e.features[0];
            let popup_html = '<div><li><b>Location</b>: ' + f.geometry.coordinates[0].toPrecision(6) + 
                ', ' + f.geometry.coordinates[1].toPrecision(6) + '</li>';

            for (key in f.properties) {
                popup_html += '<li><b> ' + key + '</b>: ' + f.properties[key] + ' </li>'
            }

            popup_html += '</div>'
            popup.setLngLat(e.features[0].geometry.coordinates)
                .setHTML(popup_html)
                .addTo(map);
        });

        

        // change cursor to pointer when mouse is over the circle feature layer
        map.on('mouseenter', 'circle', function () {
            map.getCanvas().style.cursor = 'pointer';
        });

        // reset cursor to pointer when mouse leaves the circle feature layer
        map.on('mouseleave', 'circle', function() {
            map.getCanvas().style.cursor = '';
            
                popup.remove();
            
        });
        
        // Fly to on click
        map.on('click', 'circle', function(e) {
            map.easeTo({
                center: e.features[0].geometry.coordinates,
                zoom: map.getZoom() + 1
            });
        });
    });



</script>

</body>
</html>" style="width: 100%; height: 500px;"></iframe>



```python
# url = 'https://raw.githubusercontent.com/mapbox/mapboxgl-jupyter/master/examples/data/points.csv'
# df = pd.read_csv(url)

# df_to_geojson(df,
#               filename='markers.geojson',
#               properties = ['Avg Medicare Payments', 'Avg Covered Charges', 'date', 'Provider Id'], 
#                      lat='lat', lon='lon', precision=3)


# Generate data breaks using numpy quantiles and color stops from colorBrewer
measure = 'Avg Medicare Payments'
color_breaks = [round(df[measure].quantile(q=x*0.1), 2) for x in range(1,9)]
color_stops = create_color_stops(color_breaks, colors='YlGnBu')
# data = '../data/healthcare_points.geojson'
data = json.loads(df.to_json(orient='records'))
v = CircleViz(data,
              access_token=token,
              vector_url='mapbox://rsbaumann.2pgmr66a',
              vector_layer_name='healthcare-points-2yaw54',
              vector_join_property='Provider Id',
              data_join_property='Provider Id',
              color_property=measure,
              color_stops=color_stops,
              radius=2.5,
              stroke_color='black',
              stroke_width=0.2,
              center=(-95, 40),
              zoom=3,
              below_layer='waterway-label',
              legend_text_numeric_precision=0)
v.show()
```
