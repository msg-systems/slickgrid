# Welcome to SlickGrid

Find documentation and examples in [the wiki](https://github.com/mleibman/SlickGrid/wiki) from mleibman.

Here is the documentation of the extension of SlickGrid by the [grid option](https://github.com/mleibman/SlickGrid/wiki/Grid-Options) for dynamic row height (`dynRowHeight`).

As background for this extension we used the second answer of the question ['Is variable rowheight a possibility in SlickGrid?'](http://stackoverflow.com/questions/2805094/is-variable-rowheight-a-possibility-in-slickgrid) from the stackoverflow thread. 

All changes on SlickGrid try to assure that the existing SlickGrid code does not get side effects if the new `dynRowHeight` is not used.

## Bugfixes

* Fixed strange behaviour of collapsing/expanding groups when using dataviews.
	See also here for further information: https://github.com/mleibman/SlickGrid/pull/898
* Fixed behaviour of hidden cell editor in different Zoom-Level

## Changes

* added events to __DataView__ for updating (`onItemUpdated`), deletion (`onItemDeleted`) and add (`onItemAdded`) from a item -> event returns item and - if possible - index

## Use of grid option 'dynRowHeight'

If the rows of the grid should dynamically adapt their height to their content, just set the grid option `dynRowHeight` to `true`. 

<table style="width:100%">
<tbody>
  <tr>
     <th>Option</th>
     <th>Default</th>
	 <th>Description</th>
  </tr>
  <tr>
     <td style ="width: 33%;">dynRowHeight</td>
     <td style ="width: 33%">false</td>
	 <td>Adapt the height of a row to it's content dynamicly</td>
  </tr>
 </tbody>
</table>

### Attention:

__SlickGrid activates paging automatically. If paging is activated, the functionality for dynamic row heights is disabled, even if the grid option `dynRowHeight` is `true`.__

SlickGrid checks the maximum height of a div-Container that can be rendered by the browser and stores that maximum in the Variable `maxSupportedCssHeight`.

    function getMaxSupportedCssHeight() {
	    var supportedHeight = 1000000;
      // FF reports the height back but still renders blank after ~6M px
      var testUpTo = navigator.userAgent.toLowerCase().match(/firefox/) ? 6000000 : 1000000000;
      var div = $("<div style='display:none' />").appendTo(document.body);
	  while (true) {
        var test = supportedHeight * 2;
        div.css("height", test);
        if (test > testUpTo || div.height() !== test) {
          break;
        } else {
          supportedHeight = test;
        }
      }

      div.remove();
      return supportedHeight;
    }

If the sum of all row heights is bigger then `maxSupportedCssHeight`, then SlickGrid breaks into pages. The number of pages is stored in the variable `n`.

### The functionality for dynamic row heights is activated when: 
1. the grid option `dynRowHeight` is set to `true`
2. Paging is not activated (`n === 1`)

## How does 'dynRowHeight' work?
To support dynamic row heights in SlickGrid this two questions must be answered:

- For a given row X, what is the top offset for that row?
- For a given offset Y, what row is at that offset?

To Answer this two question we did the following:

Generally we work with the heigths, offsets and offsetDeltas that we store in `rowsPositionCache` instead of the default row height. Because only rows that are in the rendered range are stored in `rowsPositionCache`, we sometimes can't answer the two questions directly. Therefore - in some methods (e.g. `scrollRowIntoView`, `getRowFromPosition`) -  we use the heights of the `rowsPositionCache` as long as a row is available otherwise we use the default heights. That means either we can directly access a value through the `rowsPositionCache` or we start calculating with the values of `rowsPositionCache` and then go on with the default heights and offsets.

### __New options and variables__

* added option `dynRowHeight` with default value `false`
* added a local Variable `rowsPositionCache` to track row sizes and positions of all rows, that are in the rendered range.

		rowsPositionCache[rowIndex] = {                // index of the row
			height: rowHeight,                         // actual row's height 
			offset: offset,                            // actual row's offset: previous row's offset + height
			offsetDelta: rowHeight - options.rowHeight // delta of actual row's height and default height 
		}
	
	`rowsPositionCache` include all rows that are stored in the `rowsCache`, except the `activeRow` if it is  outside the rendered range. Because only the rendered rows in the DOM are needed for the calculation of canvas size and scroll position.

### __CSS adaptions__
* in the method `createCssRules` the cssRules for `.slick-cell` and `.slick-row` are not set if dynamic row height is enabled
* in the method `updateRowCount` the page calculation is calling `removeCssRules` and `createCssRules` if the number of pages (`n`) changed

### __Offset calculation__

* method `getRowTop` returns the `offet` of the given row taken from `rowsPositionCache` if the row is in the rendered range
* method `getRowFromPosition` returns the searched row from `rowsPositionCache` if the position is in the rendered range

### __Grid rendering__
* method `render` received the major changes:
	* `getVisibleRange` was adjusted to ensure that the bottom value is not smaller then the top value and not bigger then the maximum number of rows
	* new function `updateRowAndCellHeights` alters the height of a row and its cells by setting it to the maximum height of the row's cells and stores it in the `rowsPositionCache` 
	* new function `updateCanvasHeight` alters the canvas height: rendered rows add their offsetDelta to the canvas height. Due to scrolling it is possible that rows enter or leave the rendered range - their offsetDelta is removed or deleted accordingly      
	* `updateRowPositions` is called to set the top offset of each rendered row. There are no changes to this method since `getRowTop` is already taking care of the rows offsets
	* `handleActiveCellPositionChange` is called to set the position newly. Due to navigation in the grid it might occur that the new active cell triggered a scroll Event. But rows have a dynamic height now - so the active cells position might have changed during rendering. This mainly occured when using grid editors with keyboard navigation resulting in wrong positioned editors
	
### __Scrolling__

One of the ugliest things in a data centered, rendering oriented table is the scrolling. Since SlickGrid is working for millions of rows it implements alot of things that optimize the rendering. All of the cool functions mainly work with a fixed rowHeight. Scrolling a SlickGrid brings in alot of test cases.  

* as explained above: we try to keep track of all currently rendered rows and their current rowHeight, offset and offsetDelta using `rowsPositionCache`. During manual scrolling: 
	* rows leave render range and their impact on the canvas height and the rows offset must be undone. `render` already does this
	* rows enter render range and their impact on the canvas height and the rows offset must be applied. `render` does this already too
* SlickGrid additionally lets us scroll to a specific row or scroll a specific row into view range.
	* method `scrollRowToTop` now uses the row's offset of the `rowsPositionCache` if possible
	* method `scrollRowIntoView` now uses the row's offset of the `rowsPositionCache` as well. But if the desired row is not yet in rendered range then we try to iteratively scroll as near as possible until the desired row is in the rendered range. Due to huge row heights this can have a performance impact