## Intermediate Tool Tutorial: Persistent Data

Some tools need to store settings or other data from session to session. For example, the drawing tool should save
the sketch on it so that if you close the app and reopen it you will see the same drawing. Likewise, the limiter
tool lets you set a threshold by dragging the handle to a certain height. And the label tool allows you to enter a
piece of text that will be displayed persistently across multiple sessions. How do we persist the data in all of
these scenarios?

The answer lies in a special kind of node that we haven't used until now. If you haven't worked much with the Toolbox
before, there is something important to understand: just as there are multiple kinds of tools that you can use,
there are multiple kinds of nodes that each tool can use. Most tools use the default type of node (simply named
"**node**"). This node just forwards data to anything it is linked to. But we can use (and even create) other types
of nodes with different functionality.

There is a type of node called "**storeData**" which is invisible (it doesn't show up in the AR app, so you can't
link it to anything) but it has the ability to store arbitrary pieces of data.

Lets look at the three APIs you'll need to use to make use of storeData nodes: `initNode`, `writePublicData`, and
`addReadPublicDataListener`. Afterwards, we'll create a simplified drawing tool to practice using these functions.

### Creating a storeData node

The API for creating a storeData node is the same `initNode` function we used in the
[beginner tool tutorial](),
except we pass in the node type `'storeData'` instead of `'node'` as the second argument. We can also ignore the
remaining arguments (these would specify its x and y position, but since it's invisible these have no effect).

Here's an example contrasting how to create a storeData node compared to a regular node:

```javascript
// creating a regular node
spatialInterface.initNode('hue', 'node', 0, 0);

// creating a storeData node
realityInterface.initNode('storage', 'storeData');
```

### Writing data to a storeData node

The API for storing data in one of these nodes is called `writePublicData`. It has three parameters.

First is the name of the storeData node, which should match the name you gave it when you called `initNode` ("storage" in the previous example).

Second is the property name that you want to write to. This lets you store multiple pieces of data in the same
storeData node, instead of creating a new node for each variable. It also allows you to subscribe to updates to only
a particular setting (as we'll see in the next section). You can name this whatever you'd like. For the drawing, it
might make sense to write to a property named "path" and for the limiter we might name the properties "min" and
"max".

Last of all, `writePublicData` needs the actual data to store. This can be any type of variable – objects, arrays,
strings, numbers, etc. Note: this data will be stringified (using `JSON.stringify`) when it is stored persistently.

Here's an example of using `writePublicData` for a drawing tool:

```javascript
// the drawing tool stores an array of coordinates in the "path" property
let imageContent = [{x: 10, y: 10}, {x: 20, y: 10}, {x: 20, y: 30}];
spatialInterface.writePublicData('storage', 'path',  imageContent);

// we can also store the current color the user is drawing with in a separate property
spatialInterface.writePublicData('storage', 'color', 'red');
```

Here's an example of using `writePublicData` to store the limiter tool's thresholds:

```javascript
// the limiter tool stores min and max threshold numbers in the "min" and "max" properties
spatialInterface.writePublicData('storage', 'min',  0.2);
spatialInterface.writePublicData('storage', 'max',  1.0);
```

If we wanted to, we could also store them together in the same property as an object. It is up to you to decide how
to structure the data:

```javascript
spatialInterface.writePublicData('storage', 'data',  {min: 0.2, max: 1.0});
```

How you structure the data becomes important when determining how to read the data from the node.

### Reading data from a storeData node

The API for reading publicData from a storeData node is similar to the API for reading new node values from a regular
node. The API is called `addReadPublicDataListener`, and it lets you subscribe to updates to a certain storeData
node's properties.

As we did for `initNode`, this example compares the regular node API to the storeData one:

```javascript
// reading a regular value from a node
// this is triggered when another linked node sends it a value
spatialInterface.addReadListener('hue', function(event) {
  console.log('received new value', event.value);
});

// reading publicData from a storeData node
// this is triggered when the app refreshes/loads the tool
// or if another user modifies the tool while you're looking at it
spatialInterface.addReadPublicDataListener('storage', 'path', function(data) {
  console.log('the storeData node named storage received a new path', data);
});
```

Note the differences. For the regular data listener, you only specify the name of the node, and the callback function
receives an event object with a property called `value`. For `addReadPublicDataListener` you specify the name of the
node *and* the name of the property you are subscribing to, and the callback function directly receives that
property's data as a parameter when it is triggered.

This function will get called one time when the tool finishes loading, if there was any data previously stored in it.
This makes it a great way to restore a tool to its state from a previous app session.

It will also get triggered if multiple users are looking at the same tool with their Spatial Toolbox app at the same
time and one of them interacts with it in a way that triggers the `writePublicData` function. This makes it a great
way to build "multiplayer" AR experiences using tools.

There's one more associated API called `reloadPublicData`. If you call the reload function, it will trigger all the
`addReadPublicDataListener` callbacks in this tool one time with the latest saved data. In most cases, you won't
need to use this, but you can trigger it if you need to guarantee that the data has been loaded.

```javascript
// this will trigger (one time) all addReadPublicDataListener callbacks for this tool
spatialInterface.reloadPublicData()
```

### Practical Example: A Simple Drawing Tool

Lets use what we've learned about storeData nodes to build a drawing tool. This is roughly adapted from the
[draw tool from the core-addon]()
but simplified to make it more concise and understandable.

Here is the final code, with comments. You can also download the tool [here]().

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Persistent Data Drawing Tutorial</title>
    <style>
        #canvas {
            width: 600px;
            height: 600px;
            position: absolute;
            left: 0;
            top: 0;
        }
    </style>
</head>
<body style="width: 600px; height: 600px; background-color: rgba(255,255,255,0.3)">
<canvas id="canvas" width="600px" height="600px"></canvas>

<script>
    let imageContent = [];
    let isPointerDown = false;
    let color = "rgb(41,253,47)"; // green color

    let ctx = document.getElementById("canvas").getContext("2d");
    ctx.strokeStyle = color;
    ctx.lineWidth = 10;
    let canvasSize = 600;

    let spatialInterface = new SpatialInterface();
    spatialInterface.initNode("storage", "storeData");

    // add event listeners when tool has safely loaded
    document.addEventListener( "DOMContentLoaded", addPointerEvents);

    function addPointerEvents() {
        // start a new line segment on pointer down
        document.getElementById("canvas").addEventListener("pointerdown", function(e) {
            isPointerDown = true;
            if (!imageContent) {
                imageContent = [];
            }

            // start a new line by moving to the touch position
            moveTo(e.clientX, e.clientY, color);

            // we encode the beginning of a line by including "isNewLine"
            imageContent.push({
                isNewLine: true,
                c: color,
                x: e.clientX,
                y: e.clientY
            });
        });

        // add more points to the current line segment if the pointer is down while moving
        document.getElementById("canvas").addEventListener("pointermove", function(e) {
            if (isPointerDown) {
                x = e.clientX;
                y = e.clientY;
                lineTo(e.clientX, e.clientY);

                // the rest of the points added to a line don't include the color
                imageContent.push({x: e.clientX, y: e.clientY});
            }
        });

        // stop the current path and save the current image to the storeData node for persistence
        document.addEventListener("pointerup", function(_e) {
            isPointerDown = false;

            // here is where we actually save the path data to the storeData node
            spatialInterface.writePublicData("storage", "imageContent",  imageContent);
        });
    }

    // begin a new line by moving to its start position within drawing along the way
    function moveTo(x, y, thisLineColor) {
        ctx.strokeStyle = thisLineColor;
        ctx.beginPath();
        ctx.moveTo(x,y);
    }

    // draw a line segment from the previous ctx position to the new x,y position
    function lineTo(x, y) {
        ctx.lineTo(x,y);
        ctx.stroke();
    }

    // delete all lines and save the empty drawing to the storeData node
    function clearCanvas() {
        ctx.clearRect(0, 0, canvasSize, canvasSize);
        imageContent = [];
        spatialInterface.writePublicData("storage", "imageContent",  imageContent);
    }

    // subscribe to any updates to the "imageContent" property of the storeData node.
    // this will also trigger one time when it loads.
    spatialInterface.addReadPublicDataListener('storage', "imageContent", function (pathData) {
        if (Array.isArray(pathData)) {
            imageContent = pathData;
        }

        if (pathData.length === 0) {
            clearCanvas(); // we reset the canvas by setting imageContent = []
        }

        // loop over all the points and draw lines as needed
        for (let i = 0; i < pathData.length; i++) {
            var point = pathData[i];

            // we decide whether to "move" (go to coordinate without drawing) or "line" (go and draw along the way)
            if (point.isNewLine) {
                moveTo(point.x, point.y, point.c);
            } else {
                lineTo(point.x, point.y);
            }
        }
    });

</script>
</body>
</html>

```
