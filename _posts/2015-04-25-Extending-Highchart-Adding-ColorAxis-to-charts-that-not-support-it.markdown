---
layout: post
title:  "Highcharts.js : Adding Color Axis to charts that not support it"
date:   2015-04-25 21:4:00
img: "posts/pie-chart.svg"
---

Highcharts JS is a JavaScript charting library based on SVG and VML rendering. Official website : [www.highcharts.com](www.highcharts.com)

In this post, we will extend this library to add a Color Axis to charts that not support it.

#####- What is Color Axis :

A Color Axis is represented by a gradient inside the legend on the chart and indicate colored values on the chart.
You can find a demonstration here : http://www.highcharts.com/demo/treemap-coloraxis 

Color Axis is supported by heat map chart, and we will use heat map properties to add Color Axis to other charts.

#####- Dependencies :

+ highcharts.js

+ heatmap.js

+ jquery

#####- Extending Pie chart :

To add a Color Axis to a Pie chart we will :

1. Extend Pie Series and override these properties : optionalAxis, colorKey, translateColors, type, axisTypes.
2. Wrap the axis prototype render function to not render axis on Pie char.
3. Wrap the Pie Series translate function to call heat map translateColor function.

{% highlight javascript %}
    (function (H) {
        H.wrap(H.Axis.prototype, 'render', function (proceed) {
            if (this.chart.userOptions.chart.type !== 'coloredPie') {
                proceed.call(this);
            }
        });
        H.wrap(H.seriesTypes.pie.prototype, 'translate', function (translate) {
            translate.apply(this, Array.prototype.slice.call(arguments, 1));
            if (this.chart.userOptions.chart.type === 'coloredPie') {
                this.translateColors.call(this);
            }
        });
        var seriesTypes = H.seriesTypes,
            merge = H.merge,
            extendClass = H.extendClass,
            defaultOptions = H.getOptions(),
            plotOptions = defaultOptions.plotOptions;
        var colorSeriesMixin = {
            optionalAxis: 'colorAxis',
            colorKey: 'colorValue', // Point color option key
            translateColors: seriesTypes.heatmap && seriesTypes.heatmap.prototype.translateColors
        };
        plotOptions.coloredPie = merge(plotOptions.pie, {});
        seriesTypes.coloredPie = extendClass(seriesTypes.pie, merge(colorSeriesMixin, {
            type: 'coloredPie',
            axisTypes: ['colorAxis']
        }));
}(Highcharts));

{% endhighlight %}

#####- Extending Column chart :

In the same way as Pie chart : 

{% highlight javascript %}
    (function (H) {
        H.wrap(H.seriesTypes.column.prototype, 'translate', function (translate) {
            translate.apply(this, Array.prototype.slice.call(arguments, 1));
            if (this.chart.userOptions.chart.type === 'coloredColumn') {
                this.translateColors.call(this);
            }
        });
        var seriesTypes = H.seriesTypes,
            merge = H.merge,
            extendClass = H.extendClass,
            defaultOptions = H.getOptions(),
            plotOptions = defaultOptions.plotOptions;
        var colorSeriesMixin = {
            optionalAxis: 'colorAxis',
            colorKey: 'colorValue',
            translateColors: seriesTypes.heatmap && seriesTypes.heatmap.prototype.translateColors
        };
        plotOptions.coloredColumn = merge(plotOptions.column, { });
        seriesTypes.coloredColumn = extendClass(seriesTypes.column, merge(colorSeriesMixin, {
            type: 'coloredColumn',
            axisTypes: ['xAxis', 'yAxis', 'colorAxis']
        }));
    }(Highcharts));
{% endhighlight %}

#####- Examples :

You can find an example [here](http://embed.plnkr.co/XWdQNB/preview).

And sources [here](https://github.com/HichamBI/Extending-Highcharts).