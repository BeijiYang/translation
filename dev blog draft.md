# Libraries

Open Google.com, search for "Vega", and this is what you'll see: Vega, Vega-Lite, Vega-Embed, React-Vega, so many libraries. If you don't have much knowledge about them, you will get lost.

Don't worry, down blew is the most simplified explanation:

* [Vega](https://vega.github.io/vega/):  A visualization grammar. You can describe the chart using JSON, and get a Canvas or SVG. It is powerful and flexible.

* [Vega-Lite](https://vega.github.io/vega-lite/): A high-level grammar of interactive graphics, also JSON in Canvas/SVG out. Vega-Lite specifications can be compiled to Vega specifications. It is a convenient way to create a chart swiftly. 

* [Vega-Embed](https://github.com/vega/vega-embed): A tool to embed the Vega/Vega-Lite chart into web pages.

* [React-Vega](https://github.com/vega/react-vega/tree/master/packages/react-vega): A tool to help you using the Vega/Vega-Lite chart in React project.

Since Vega-Lite is quicker but not as powerful as Vega, you can express your idea rapidly using Vega-Lite, then edit/add more advanced functions based on the Vega specifications generated from Vega-Lite.


# The chart

Regarding there are already some articles talking about the bar chart on the internet, we are going to focus on making a decent area chart.

![](https://img-blog.csdnimg.cn/20191024233116206.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

First, we start with this basic area chart. It shows the active users in the first week in October.

You can copy&paste them in [Vega editor](https://vega.github.io/editor/#/).

```
{
  "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
  "mark": {"type": "area", "color": "#0084FF", "interpolate": "monotone"},
  "encoding": {
    "x": {
      "field": "date",
      "type": "temporal",
      "timeUnit": "yearmonthdate",
      "axis": {"title": "Date"}
    },
    "y": {
      "field": "active_users",
      "type": "quantitative",
      "axis": {"title": "Active Users"}
    },
    "opacity": {"value": 1}
  },
  "width": 400,
  "height": 300,
  "data": {
    "values": [
      {"active_users": 0, "date": "2019-10-01"},
      {"active_users": 2, "date": "2019-10-02"},
      {"active_users": 0, "date": "2019-10-03"},
      {"active_users": 1, "date": "2019-10-04"},
      {"active_users": 0, "date": "2019-10-05"},
      {"active_users": 0, "date": "2019-10-06"},
      {"active_users": 1, "date": "2019-10-07"}
    ]
  },
  "config": {}
}

```

![basic chart](https://img-blog.csdnimg.cn/20191024225707560.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

## Show integer labels in the y-axis

As you noticed, the numbers of active users are pretty small. We do this to show the problem of Vega's default behavior. The chart looks very good when the number is big, but when the number is small, the labels of the y-axis become float numbers. You can edit the data in the Vega editor by yourself.

There is no "a half" user, the labels are supposed to be integers.

According to the official document, we can use the format function `[d]()` to make the labels integers.
```
...
 "y": {
      "field": "active_users",
      "type": "quantitative",
      "axis": {
        "title": "Active Users",
        "format": "d"
        }
    },
...
```

![](https://img-blog.csdnimg.cn/20191024225806544.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

Now the floats are gone but a new issue appears. Some labels are duplicated. 

We need to explicitly set the visible axis tick values using `values` on the y-axis.

```
...
 "y": {
      "field": "active_users",
      "type": "quantitative",
      "axis": {
        "title": "Active Users",
        "format": "d",
        "values": [1,2]
        }
    },
...

```

![](https://img-blog.csdnimg.cn/20191024225915437.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

Now we get a proper y-axis. Notice that there is no `0` in the array, so we can get an empty when the data is empty. The `values` will change when the data change, so it needs to be passed into the spec JSON in the React component dynamically. We'll talk about it later.


## Show proper labels in the x-axis(time axis)

Currently, the x-axis looks good. But if we minimize the amount of data, or increase the width of the chart, the same duplicate-issue occurs.

let try to make the chart larger.

```
 "width": 1200,
 "height": 600,
```
![](https://img-blog.csdnimg.cn/20191024230645542.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)


Set the `type` as [`ordinal`](https://vega.github.io/vega-lite/docs/type.html#ordinal) can solve this problem.

![](https://img-blog.csdnimg.cn/20191024231217533.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70
)

But it is not the silver bullet: when the amount of data increases, 
harm the readability. 

```
...
"values": [
      {"active_users": 0, "date": "2019-10-01"},
      {"active_users": 2, "date": "2019-10-02"},
      {"active_users": 0, "date": "2019-10-03"},
      {"active_users": 1, "date": "2019-10-04"},
      {"active_users": 0, "date": "2019-10-05"},
      {"active_users": 0, "date": "2019-10-06"},
      {"active_users": 1, "date": "2019-10-07"},
      {"active_users": 0, "date": "2019-10-08"},
      {"active_users": 2, "date": "2019-10-09"},
      {"active_users": 0, "date": "2019-10-10"},
      {"active_users": 1, "date": "2019-10-11"},
      {"active_users": 0, "date": "2019-10-12"},
      {"active_users": 0, "date": "2019-10-13"},
      {"active_users": 1, "date": "2019-10-14"},
      {"active_users": 0, "date": "2019-10-15"},
      {"active_users": 0, "date": "2019-10-16"},
      {"active_users": 1, "date": "2019-10-17"},
      {"active_users": 0, "date": "2019-10-18"},
      {"active_users": 2, "date": "2019-10-19"},
      {"active_users": 2, "date": "2019-10-20"},
      {"active_users": 0, "date": "2019-10-21"},
      {"active_users": 2, "date": "2019-10-22"},
      {"active_users": 0, "date": "2019-10-23"},
      {"active_users": 1, "date": "2019-10-24"},
      {"active_users": 0, "date": "2019-10-25"},
      {"active_users": 0, "date": "2019-10-26"},
      {"active_users": 1, "date": "2019-10-27"},
      {"active_users": 0, "date": "2019-10-28"},
      {"active_users": 2, "date": "2019-10-29"}
    ]
  },
...
```

![](https://img-blog.csdnimg.cn/20191024231510377.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

So we need to add a condition: if the range is longer than 30 days, use `temporal`, otherwise, use `ordinal`.

For the sake of users nack, let's adjust the angle of the labels by setting `labelAngle`.

```
const getDateXObj = rangeLen => ({
  field: 'date',
  type: `${rangeLen > 30 ? 'temporal' : 'ordinal'}`,
  timeUnit: 'yearmonthdate',
  axis: {
    title: 'Date',
    labelAngle: -45,
  },
});

```

![](https://img-blog.csdnimg.cn/20191024231600874.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

If we only want to solve the labels-too-crowed problem, basically the `labelAngle` can do the job. 

The true difference between `temporal` and `ordinal` is:
* The former puts the data on an **incompressible** timeline.
* The latter only displays the existing data.

The following two pictures can be used to illustrate the difference. Note that only one day is added to the data.

```
 "values": [
      {"active_users": 0, "date": "2019-10-01"},
      {"active_users": 2, "date": "2019-10-02"},
      {"active_users": 0, "date": "2019-10-03"},
      {"active_users": 1, "date": "2019-10-04"},
      {"active_users": 0, "date": "2019-10-05"},
      {"active_users": 0, "date": "2019-10-06"},
      {"active_users": 1, "date": "2019-10-07"},
      {"active_users": 2, "date": "2020-01-01"} // happy new year~
    ]
```

`temporal`

![temporal](https://img-blog.csdnimg.cn/20191024231931687.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

`ordinal`

![ordinal](https://img-blog.csdnimg.cn/20191024232100676.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

# Display Vega-Lite chart in the React project

Install the `react-vega` library.

```
npm install react vega vega-lite react-vega --save
```
First, show a basic area chart in the project.

```
...
import { Vega } from 'react-vega';


const spec = {
  "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
  "mark": {"type": "area", "color": "#0084FF", "interpolate": "monotone"},
  "encoding": {
    "x": {
      "field": "date",
      "type": "ordinal",
      "timeUnit": "yearmonthdate",
      "axis": {
      		"title": "Date",
      		"labelAngle": -45
      	}
    },
    "y": {
      "field": "active_users",
      "type": "quantitative",
      "axis": {
        "title": "Active Users",
        "format": "d",
        "values": [1,2]
        }
    },
    "opacity": {"value": 1}
  },
  "config": {}
}

const data = [
      {"active_users": 0, "date": "2019-10-01"},
      {"active_users": 2, "date": "2019-10-02"},
      {"active_users": 0, "date": "2019-10-03"},
      {"active_users": 1, "date": "2019-10-04"},
      {"active_users": 0, "date": "2019-10-05"},
      {"active_users": 0, "date": "2019-10-06"},
      {"active_users": 1, "date": "2019-10-07"}
    ]

...

return (
  ...

    <Vega
      spec={{
        ...spec,
         width: 400,
         height: 300,
         data: { values: data },
       }}
      />
  ...
)

```

Then we make the chart better based on what we have talked about earlier.

Let's add some new functions:

* a new `getSpec` function to return the `spec` object.
* a function to generate the values array based on the biggest active user value from the data.
```
// I can make this example code better
... 

const getSpec = (yAxisValues = [], rangeLen = 0) => ({
  "$schema": "https://vega.github.io/schema/vega-lite/v4.json",
  "mark": {"type": "area", "color": "#0084FF", "interpolate": "monotone"},
  "encoding": {
    "x": {
      "field": "date",
      type: `${rangeLen > 30 ? 'temporal' : 'ordinal'}`,
      "timeUnit": "yearmonthdate",
      "axis": {"title": "Date"}
    },
    "y": {
      "field": "active_users",
      "type": "quantitative",
      "axis": {
        "title": "Active Users",
        "format": "d",
        "values": yAxisValues
        }
    },
    "opacity": {"value": 1}
  },
  "config": {}
})

...

function App() {
  // get max value from data arary
  const yAxisMaxValueFor = (...keys) => {
    const maxList = keys.map(key => data.reduce((acc, cur) => (cur[key] > acc[key] ? cur : acc))[key]);
    return Math.max(...maxList);
  };

  const yAxisValues = Array.from(
    { length: yAxisMaxValueFor('active_users') },
  ).map((v, i) => (i + 1));

  const spec = getSpec(yAxisValues, data.length);
  
  return (
    <div className="App">
      <Vega
        spec={{
          ...spec,
          autosize: 'fit',
          resize: true,
          contains: 'padding',
          width: 400,
          height: 300,
          data: { values: data },
        }}
      />
    </div>
  );
}
...

```

So far, we have successfully introduced the charts described by Vega-Lite in the React project.

![](https://img-blog.csdnimg.cn/20191024232353948.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

## Manage the widget

![](https://img-blog.csdnimg.cn/20191024232430421.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

On the right-top corner, there is a widget. Click it and you'll see all the actions you can take: download the chart in different formats, view source/complied Vega, open in Vega editor.

The last three items should only appear in the development stage. In the production, we leave the first two download options, and it would be better if we can specify the download file name instead of a default one.

Luckily, it can be done using `React-Vega`. This is an advantage of React-Vega, which supports several configuration features of `Vega-Embed`, as described in the [documentation](https://vegawidget.github.io/vegawidget/reference/vega_embed.html).

Here, we only need two more configurations: `actions` and `downloadFileName`. The former uses Boolean values to control exported functions, the latter specifies the name of the downloaded file.

```

  ...
  <Vega
    spec={ ... }
    actions={{
        export: true,
        source: false,
        compiled: false,
        editor: false,
      }}
      downloadFileName={'1024.avi'}
    />
  ...

```

Finally, add a title for the chart.

```
  ...
  "title": '1024',
  ...

```

Now we get a decent area chart!

![](https://img-blog.csdnimg.cn/20191024233116206.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

# Multi-layer Chart 

There is nothing special about the multi-layer chart. All we need to do is to add a `layer` array. It is like a stack, we push the chart items in from the top.

To demonstrate a multi-layer chart, we add the "user_comments" in the previous data.

```
...
"data": {
    "values": [
    { "user_comments": 0, "active_users": 0, "date": "2019-10-01" },
    { "user_comments": 3, "active_users": 2, "date": "2019-10-02" },
    { "user_comments": 1, "active_users": 0, "date": "2019-10-03" },
    { "user_comments": 1, "active_users": 1, "date": "2019-10-04" },
    { "user_comments": 2, "active_users": 0, "date": "2019-10-05" },
    { "user_comments": 1, "active_users": 0, "date": "2019-10-06" },
    { "user_comments": 2, "active_users": 1, "date": "2019-10-07" }
    ]
  },
 ...
```

We can create a single layer "User Comments" chart using the same Vega-Lite grammar as the previous one. Just replacing some items in the y-axis object is enough.

```
{
      "mark": {"type": "area", "color": "#e0e0e0", "interpolate": "monotone"},
      "encoding": {
        "x":{
           "field": "date",
           "type": "ordinal",
           "timeUnit": "yearmonthdate",
           "axis": {"title": "Date", "labelAngle": -45}
        },
         "y": {
            "field": "user_comments",
	        "type": "quantitative",
            "axis": {
                "title": "User Comments",
                "format": "d",
                "values": [1,2,3]
            }
        }
      }
  }
```

![user comments](https://img-blog.csdnimg.cn/2019102921460942.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

Then we add a `layer` array. Push the "User Comments" chart object in this array. Nothing changed, it is still a single layer chart.

```
...
"layer":[
    {
      "mark": {"type": "area", "color": "#e0e0e0", "interpolate": "monotone"},
      "encoding": {
        "x":{
           "field": "date",
           "type": "ordinal",
           "timeUnit": "yearmonthdate",
           "axis": {"title": "Date", "labelAngle": -45}
        },
        "y": {
            "field": "user_comments",
            "type": "quantitative",
            "axis": {
                "title": "User Comments",
                "format": "d",
                "values": [1,2,3]
             }
          }
      }
    }
  ],
  ...
```

What will happen if we push the "Active Users" object in this array, too?

```
...
"layer":[
    {
      "mark": {"type": "area", "color": "#e0e0e0", "interpolate": "monotone"},
      "encoding": {
        "x":{
           "field": "date",
           "type": "ordinal",
           "timeUnit": "yearmonthdate",
           "axis": {"title": "Date", "labelAngle": -45}
        },
        "y": {
            "field": "user_comments",
            "type": "quantitative",
            "axis": {
                "title": "User Comments",
                "format": "d",
                "values": [1,2,3]
             }
          }
      }
    },
    {
      "mark": {"type": "area", "color": "#0084FF", "interpolate": "monotone"},
      "encoding": {
        "x": {
          "field": "date",
          "type": "ordinal",
          "timeUnit": "yearmonthdate",
          "axis": {"title": "Date", "labelAngle": -45}
        },
        "y": {
          "field": "active_users",
          "type": "quantitative",
          "axis": {
          "title": "Active Users",
          "format": "d",
          "values": [1,2]
          }
        }
      }
    }
  ],
  ...
```

Here it is! A double layer chart.

![double layer chart](https://img-blog.csdnimg.cn/20191029215753251.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

# add legend

Although this chart looks pretty, it has a problem with clarity. Our user can not know which color indicates which data in a glance. Now is when we need to add legends in the chart.

There are several approaches to make it happen, here is my solution: using `stroke`  to create the legend, using `legend` to optimize the style.

add the `stroke` object in any chart object.

```
...
{
      "mark": {"type": "area", "color": "#e0e0e0", "interpolate": "monotone"},
      "encoding": {
        "x":{
           "field": "date",
           "type": "ordinal",
           "timeUnit": "yearmonthdate",
           "axis": {"title": "Date", "labelAngle": -45}
        },
        "y": {
            "field": "user_comments",
            "type": "quantitative",
            "axis": {
                "title": "User Comments",
                "format": "d",
                "values": [1,2,3]
             }
          },
        "stroke": {
          "field": "symbol",
          "type": "ordinal",
          "scale": {
            "domain": ["User Comments", "Active Users"],
            "range": ["#e0e0e0", "#0084FF"]
          }
        }
      }
    },
    ...
```
![lame legend](https://img-blog.csdnimg.cn/2019102922121540.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

We get a lame legend in the right-top corner. 

Let's make it decent! add `legend` object in the top `config` object.

```
...
 "legend": {
        "offset": -106, // Adjust the horizontal distance of the legend
        "title": null,
        "padding": 5,
        "strokeColor": "#9e9e9e",
        "strokeWidth": 2,
        "symbolType": "stroke",
        "symbolOffset": 0,
        "symbolStrokeWidth": 10,
        "labelOffset": 0,
        "cornerRadius": 10,
        "symbolSize": 100,
        "clipHeight": 20
    }
   ...
```

![decent legend](https://img-blog.csdnimg.cn/20191029221539677.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

Now it looks so much better.

Since we already have the legend, we can safely get rid of the labels on the y-axis. Just delete the `title` of  `y.axis`  object, or leave it empty. 

When the amount of the layer grows, we can also use both the area chart and the line chart, just like the example down blew. Looks nice, isn't it?

![多层图](https://img-blog.csdnimg.cn/20191029223006212.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0JlaWppeWFuZzk5OQ==,size_16,color_FFFFFF,t_70)

# resize 
In a real project, we have to make sure that the size of the chart always fit the browser window. Now the chart won't scale, we need to do something in the React component.

... ...

